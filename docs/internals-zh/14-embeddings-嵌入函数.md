# 14 · Embeddings 嵌入函数体系：让数据库自己把文字算成向量

> **本篇解剖的源码**：
> - `rust/lancedb/src/embeddings.rs`（307 行，模块根：两个核心 trait + 注册表 + 写入适配器）
> - `rust/lancedb/src/embeddings/openai.rs`（262 行）、`bedrock.rs`（214 行）、`sentence_transformers.rs`（474 行）——三家具体实现
> - 跨模块呼应：`table.rs:88`（`ColumnKind`）、`table.rs:1957`（`NativeTable::add` 接入点）、`connection.rs`（注册与定义入口）、`error.rs:24`
>
> **覆盖区间**：整个 `embeddings` 模块自身结构 + 它如何缝进写入链路。**不**重复第 04 篇写入链路的全貌。
>
> **前置阅读**：`01 Rust 垫脚石`（trait / `dyn` / `Arc` / async 四节）；`09 Table 解剖（上）` §3.2（给 schema "贴标签"，本篇 §4 直接呼应）；`04 写入链路`（本篇 §5 是它的一段放大镜特写）。
>
> **与哪些篇互补**：`04 写入链路`讲"一条数据从 `add` 到落盘"的全程，本篇只放大其中"凭空多出一列向量"那一瞬；`09/10 Table 解剖`讲 `TableDefinition`/`ColumnKind` 的持久化，本篇讲谁来读这个标签、读了干什么。

---

## 0. 本篇要回答的问题

如果你只带走三句话：

1. **整个模块只定义了两个 trait**——`EmbeddingFunction`（一台"翻译机"：吃一列文字、吐一列向量）和 `EmbeddingRegistry`（"设备名册"：按名字找翻译机）。其余全是围着这两个 trait 转的胶水。
2. **"自动算向量"这件事是被"缝"进写入流水线的，没改 lance 一行**——靠一个叫 `WithEmbeddings` 的适配器，它本身又是一个 `RecordBatchReader`（餐盘传送带），所以能无缝接到第 04 篇的写入链路上。这是"组装而非重造"最干净的一个范例。
3. **同一个 trait，三种后厨**——OpenAI / Bedrock 是"打电话点外卖"（HTTP 调云端 API），sentence-transformers 是"自己开火"（本地 candle 跑 BERT 模型）。三者签名完全一致，所以注册进同一本名册、走同一条流水线，调用方完全无感。

把它画成一张图：

```
  用户建表时声明：「text 这一列，用名叫 'openai' 的翻译机算成向量」
                              │
                              ▼  序列化进 schema 元数据(ColumnKind::Embedding)  ← §4
  ┌──────────────────────────────────────────────────────────┐
  │                  EmbeddingRegistry (设备名册/trait)         │  embeddings.rs:81
  │     get("openai") ─┐   functions()   register(name, fn)    │
  └────────────────────┼──────────────────────────────────────┘
                       │  默认实现
                       ▼
  ┌──────────────────────────────────────────────────────────┐
  │       MemoryRegistry  =  Arc<RwLock<HashMap<名字, 翻译机>>>  │  embeddings.rs:93
  └──────────────────────────────────────────────────────────┘
                       │  存的每个值都是
                       ▼
  ┌──────────────────────────────────────────────────────────┐
  │            EmbeddingFunction (翻译机规格书/trait)            │  embeddings.rs:45
  │  name / source_type / dest_type / compute_source_*/query_* │
  └───┬───────────────────┬────────────────────┬──────────────┘
      │ impl              │ impl               │ impl
      ▼                   ▼                    ▼
 OpenAIEmbedding…   BedrockEmbedding…   SentenceTransformers…
 (HTTP 外卖)         (HTTP 外卖)          (本地灶台·candle BERT)
   openai.rs:71       bedrock.rs:55        sentence_transformers.rs:47

  ── 写入时怎么用上它？───────────────────────────────────────────
  NativeTable::add (table.rs:1957)
      └─ MaybeEmbedded::try_new (embeddings.rs:143)   ← 分流闸：这单要不要加料?
           ├─ 要 → WithEmbeddings (加料工位，自己也是传送带 RecordBatchReader)
           │        └─ next() 每读一盘，就 compute_source_embeddings 多加一列向量
           └─ 不要 → 原样透传(零开销)
```

> **比喻总钥匙**（延续餐厅家族）：`EmbeddingFunction` 是**翻译机的规格书**（规定"吃文字、吐向量"的统一接口，内部接外卖还是自己开火它不管）；`EmbeddingRegistry`/`MemoryRegistry` 是后厨墙上的**设备名册白板**（按名字登记/领取翻译机）；`WithEmbeddings` 是传送带上的**加料工位**（餐盘经过时照点菜单多加一列向量配菜，再放回传送带）；`MaybeEmbedded` 是传送带入口的**分流闸**（要加料走工位，不加料直通）；`ColumnKind::Embedding` 是用工合同里盖的**"此列由翻译机代工"的章**。

---

## 1. 为什么数据库要内置 embedding

先说清楚这个模块解决什么痛点。

向量数据库的日常是"语义搜索"：用户想搜"和这句话意思相近的文档"。但磁盘上存的、能被向量索引（见 `12 Index 体系`）检索的，是**向量**（一串浮点数），不是原始文字。于是中间永远有一道工序：**把文字/图片送进某个模型，得到向量**。这道工序就叫 embedding（嵌入）。

没有内置 embedding 时，用户得这样：

```
用户代码:  text = "a cat"
           vec  = openai_client.embed(text)     # ← 用户自己调模型
           table.add({text: "a cat", vector: vec})
```

每次写入、每次查询前，用户都要手动算一遍向量，还要保证写入用的模型和查询用的模型一致。这又啰嗦又容易出错。

LanceDB 的选择是：**让用户只写原始文字，数据库在写入流水线里自动补上向量列**。用户建表时声明一次"`text` 这列请用 `openai` 翻译机"，之后只管 `add({text: "a cat"})`，向量列凭空多出来。

```
用户代码:  table.add({text: "a cat"})       # ← 只写文字
           （DB 内部自动算出 text_embedding 列并一起落盘）
```

> **比喻**：好比餐厅菜单上写"本店所有牛排自动配一份酱汁"。顾客（用户）只管点牛排（写文字），加料工位（`WithEmbeddings`）会照规格书自动浇上酱汁（向量）。顾客不用自己带酱汁来。

这就是本模块的全部使命。下面我们把它拆开。

---

## 2. `EmbeddingFunction`：一台翻译机要会做什么

模块的第一根支柱，定义在 `embeddings.rs:45`：

```rust
pub trait EmbeddingFunction: std::fmt::Debug + Send + Sync {   // ①
    fn name(&self) -> &str;                                                  // :46
    fn source_type(&self) -> Result<Cow<DataType>>;                         // :48 输入是什么类型
    fn dest_type(&self) -> Result<Cow<DataType>>;                           // :51 输出是什么类型
    fn compute_source_embeddings(&self, source: Arc<dyn Array>)             // :53 写入时：一列文字→一列向量
        -> Result<Arc<dyn Array>>;
    fn compute_query_embeddings(&self, input: Arc<dyn Array>)               // :55 查询时：查询文字→查询向量
        -> Result<Arc<dyn Array>>;
}
```

- ① `: std::fmt::Debug + Send + Sync` —— **supertrait 约束**：想当翻译机，先得能 `{:?}` 打印（调试用）、且能跨线程安全传递/共享（`Send + Sync`）。后者是因为同一个翻译机实例会被塞进 `Arc` 给多个表、多个线程共享（见 §3）。

五个方法是翻译机的"规格书"。逐个看：

| 行号 | 方法 | 作用 | WHY |
|---|---|---|---|
| `:46` | `name(&self) -> &str` | 翻译机的注册名（如 `"openai"`） | 注册表按这个名字登记/查找它（§3） |
| `:48` | `source_type() -> Cow<DataType>` | 能吃什么输入。三家都返回 `DataType::Utf8`（字符串） | 上层据此校验"这列类型对不对" |
| `:51` | `dest_type() -> Cow<DataType>` | 吐出什么输出。三家都是 `FixedSizeList<Float32, ndims>` 且**非空** | **注释 :50 强调：必须永远等于 `compute_*` 实际产出的类型**，否则 schema 对不上 |
| `:53` | `compute_source_embeddings(source)` | **写入路径**：把整列源数据算成整列向量 | 这是 `WithEmbeddings` 每批调用的那个方法（§5） |
| `:55` | `compute_query_embeddings(input)` | **查询路径**：把用户查询文字算成查询向量 | 注意：本 crate 的 `src` 里**没有调用点**——查询侧的触发在更上层（Python/Node 绑定或 query builder），这里只是三家实现了它 |

> **Rust 知识点：`Cow<DataType>`（Clone-On-Write）**。`source_type`/`dest_type` 返回的是 `Cow<DataType>`，意思是"这个值**可能借自别处、也可能是我新建的**，调用方不用关心"。三家实现里都返回 `Cow::Owned(...)`（新建的）。类比：Python 里返回值你不会区分它是引用还是拷贝；Rust 用 `Cow` 把"借还是拥有"这个选择显式留给实现者，零额外开销。

> **Rust 知识点：`Arc<dyn Array>`**。`dyn Array` 是 Arrow 数组的 trait 对象（"任意一种数组，运行时才知道具体是 Float32 还是 Utf8"，类似 Java 接口/Python 鸭子类型）。`Arc` 是引用计数指针，让数据能被廉价共享而不深拷贝。整列数据在 Arrow 里就是一个 `Arc<dyn Array>`，所以"一列文字"和"一列向量"都用它表示。

**为什么把 `source_type`/`dest_type` 单列成方法，而不是写死？** 因为不同模型维度不同：OpenAI ada-002 是 1536 维、3-large 是 3072 维、Bedrock Cohere 是 1024 维。`dest_type` 让每台翻译机自报家门"我吐 N 维向量"，上层就能据此构造正确的目标列 schema。

---

## 3. `EmbeddingRegistry` 与 `MemoryRegistry`：设备名册

第二根支柱，`embeddings.rs:81`：

```rust
pub trait EmbeddingRegistry: Send + Sync + std::fmt::Debug {
    fn functions(&self) -> HashSet<String>;                                 // :83 列出所有已注册名字
    fn register(&self, name: &str, function: Arc<dyn EmbeddingFunction>)    // :86 登记一台翻译机
        -> Result<()>;
    fn get(&self, name: &str) -> Option<Arc<dyn EmbeddingFunction>>;        // :88 按名字领取
}
```

**为什么需要注册表，不能直接传函数？** 因为翻译机的"声明"和"使用"被时空隔开了：

- 用户**建表时**只声明一个**名字**字符串：`EmbeddingDefinition { embedding_name: "openai", ... }`（§4），这个名字会被**序列化进磁盘**（schema 元数据），活得比进程长。
- 真正**写入时**（可能是另一次进程启动），需要凭这个名字**找回**一个能干活的翻译机实例。

名字是可序列化的，而翻译机实例（持有 API key、HTTP client、几百 MB 的 BERT 模型）不可序列化。注册表就是这道**"名字 ↔ 实例"的查找表**。

> **对照 Python**：`EmbeddingRegistry` 就是一个 `dict[str, EmbeddingFunction]`。`register` = `d[name] = fn`，`get` = `d.get(name)`，`functions` = `set(d.keys())`。

### 3.1 `MemoryRegistry`：墙上的白板

默认实现 `embeddings.rs:93`：

```rust
#[derive(Debug, Default, Clone)]
pub struct MemoryRegistry {
    functions: Arc<RwLock<HashMap<String, Arc<dyn EmbeddingFunction>>>>,    // :94 ← 唯一字段
}

impl EmbeddingRegistry for MemoryRegistry {
    fn functions(&self) -> HashSet<String> {
        self.functions.read().unwrap().keys().cloned().collect()           // :99 读锁
    }
    fn register(&self, name: &str, function: Arc<dyn EmbeddingFunction>) -> Result<()> {
        self.functions.write().unwrap().insert(name.to_string(), function); // :103 写锁
        Ok(())
    }
    fn get(&self, name: &str) -> Option<Arc<dyn EmbeddingFunction>> {
        self.functions.read().unwrap().get(name).cloned()                  // :111 读锁
    }
}
```

唯一的字段就是一个 `HashMap`，但裹了三层包装，各司其职：

| 包装层 | 作用 | 类比 |
|---|---|---|
| `HashMap<String, Arc<dyn EmbeddingFunction>>` | 真正的数据：名字 → 翻译机 | Python 的 `dict` |
| `RwLock<...>` | **多读单写**并发控制：`read()` 拿读锁（可多个并发）、`write()` 拿写锁（独占） | 读写锁 |
| `Arc<...>` | 让这把锁能被多处共享（多个表、多个连接共用同一本名册） | 共享指针 |

> **Rust 陷阱：`register`/`get` 都是 `&self`（不可变借用），怎么还能改内容？** 这叫**内部可变性（interior mutability）**——对外签名是"我只是读自己"，对内通过 `RwLock` 拿到可变访问偷偷改 `HashMap`。这是 Rust 并发的核心套路（第 09 篇讲 `DatasetConsistencyWrapper` 时也是同一招）。如果 `register` 写成 `&mut self`，注册表就没法被 `Arc` 共享了——共享指针只给你 `&self`。

> **Rust 陷阱：`.read().unwrap()` / `.write().unwrap()` 里的 `unwrap`**。`RwLock::read()` 返回 `Result`，只有在锁"中毒"（别的线程持锁时 panic 了）时才是 `Err`。这里 `unwrap` 等于说"中毒了就跟着 panic"。这在 Rust 里很常见，但要知道它**能 panic**——贡献者别把可能频繁中毒的逻辑塞这后面。

`MemoryRegistry::new()`（`:117`）直接返回 `Self::default()`——即一个空白板。

### 3.2 谁来兜底这本名册

当你建连接（`ConnectBuilder`）不指定自定义注册表时，`connection.rs` 在三处用 `Arc::new(MemoryRegistry::new())` 兜底（`connection.rs:832`、`:855`、`:928`）。想换成自己的注册表，调 `ConnectBuilder::embedding_registry`（`connection.rs:739`）：

```rust
pub fn embedding_registry(mut self, registry: Arc<dyn EmbeddingRegistry>) -> Self {  // :739
    self.embedding_registry = Some(registry);
    self
}
```

---

## 4. `EmbeddingDefinition` 与 `ColumnKind::Embedding`：嵌入列随 schema 持久化

第 09 篇 §3.2 讲过：`TableDefinition` 给每列贴一个 `ColumnKind` 标签，序列化进 Arrow schema 的 metadata，"使用说明书跟着数据走"。嵌入列就是靠这个标签活下来的。

### 4.1 `ColumnKind`：盖在列上的章

`table.rs:88`：

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ColumnKind {
    Physical,                               // :90 用户直接写入的普通列(最常见)
    Embedding(EmbeddingDefinition),         // :92 ★由翻译机代工的列，携带"怎么算"的定义
}
```

`Embedding` 变体里**塞着一份 `EmbeddingDefinition`**——这就是那张"此列由翻译机代工"的章，章上写明"用哪台翻译机、从哪列算、算到哪列"。

### 4.2 `EmbeddingDefinition`：章上的内容

`embeddings.rs:60`：

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Hash)]   // :59 注意 Serialize/Deserialize
pub struct EmbeddingDefinition {
    pub source_column: String,              // :62 输入列名(如 "text")
    pub dest_column: Option<String>,        // :65 向量列名；None 时默认 `{source}_embedding`
    pub embedding_name: String,             // :67 用哪台已注册翻译机(如 "openai")
}
```

构造器 `new`（`:71`）签名是 `new<S: Into<String>>(source_column, embedding_name, dest)`——三个参数都接受任何能转成 `String` 的类型（`&str`、`String` 都行）。

> **关键点：为什么 `derive(Serialize, Deserialize)`？** 因为这份定义要**写进磁盘的 schema 元数据**（经 serde_json），跨进程、跨时间存活。注意它只存 `embedding_name`（字符串），**不存翻译机实例**——实例靠注册表在用时按这个名字找回（呼应 §3"名字 ↔ 实例"）。这就是为什么定义可序列化而翻译机不可序列化，两者必须分家。

> **比喻**：`EmbeddingDefinition` 是合同附页上的一行字："`text` 列的向量，由名叫 `openai` 的翻译机代工，成品放进 `text_embedding` 列。" 合同（schema）存档进金库，多年后还能照着这行字找对应的翻译机重新开工。

### 4.3 定义从哪里来

建表时调 `CreateTableBuilder::add_embedding`（`connection.rs:279`）把定义挂上去，且**当场校验名字有效**：

```rust
pub fn add_embedding(mut self, definition: EmbeddingDefinition) -> Result<Self> {   // :279
    let embedding_func = self
        .embedding_registry
        .get(&definition.embedding_name)                       // :281 立刻去名册查
        .ok_or_else(|| Error::EmbeddingFunctionNotFound {      // :284 查不到，当场报错
            name: definition.embedding_name.clone(),
            reason: "No embedding function found in the connection's embedding_registry".to_string(),
        })?;
    self.embeddings.push((definition, embedding_func));        // :290 配对存好
    Ok(self)
}
```

> **设计点：早校验（fail fast）**。如果你声明用 `"opanai"`（打错字）这台不存在的翻译机，`add_embedding` 在**建表那一刻**就报 `Error::EmbeddingFunctionNotFound`（定义在 `error.rs:24`，display 文本 `"Embedding function '{name}' was not found. : {reason}"`），而不是等到第一次写入时才炸。报错越早，越好排查。

---

## 5. 写入时的按需计算（放大第 04 篇的一瞬）

现在把前四节的零件串起来：**一列向量到底是怎么凭空多出来的**。这是第 04 篇写入链路里被一笔带过的那一步，我们在这里架上放大镜。

### 5.1 接入点：写入流水线的"分流闸"

第 04 篇讲过，`NativeTable::add` 收到一个 `data: Box<dyn RecordBatchReader>`（餐盘传送带）。它做的第一件事就是把传送带包进分流闸（`table.rs:1957`）：

```rust
async fn add(
    &self,
    add: AddDataBuilder<NoData>,
    data: Box<dyn RecordBatchReader + Send>,
) -> Result<()> {
    let data = Box::new(MaybeEmbedded::try_new(                 // :1962 ← 包一层分流闸
        data,
        self.table_definition().await?,                        // 把"使用说明书"(含 ColumnKind 标签)传进去
        add.embedding_registry,                                // 把"设备名册"传进去
    )?);
    // ... 继续把 data 交给下游 lance write（第 04 篇）
}
```

注意：**"是否要算向量"在传送带被包裹的阶段就决定好了**，下游 lance 拿到的就是"可能已经加宽的传送带"，根本不知道向量是临时算的。

### 5.2 `MaybeEmbedded::try_new`：要不要加料？

`embeddings.rs:131` 先看这个 enum 本身——它是个**双态包装器**：

```rust
pub enum MaybeEmbedded<R: RecordBatchReader> {
    Yes(WithEmbeddings<R>),   // :133 有嵌入列 → 走加料工位
    No(R),                    // :136 没有 → 原样透传
}
```

`try_new`（`embeddings.rs:143`）的选择逻辑：

```rust
pub fn try_new(
    inner: R,
    table_definition: TableDefinition,
    registry: Option<Arc<dyn EmbeddingRegistry>>,
) -> Result<Self> {
    if let Some(registry) = registry {                                       // :148 没名册直接走 No
        let mut embeddings = Vec::with_capacity(...);
        for cd in table_definition.column_definitions.iter() {               // :150 扫每一列的标签
            if let ColumnKind::Embedding(embedding_def) = &cd.kind {         // :151 这列盖了"代工"章?
                match registry.get(&embedding_def.embedding_name) {          // :152 凭名字去名册领翻译机
                    Some(func) => embeddings.push((embedding_def.clone(), func)),  // :154 配对存好
                    None => return Err(Error::EmbeddingFunctionNotFound { ... }),  // :157 领不到→报错
                }
            }
        }
        if !embeddings.is_empty() {
            return Ok(Self::Yes(WithEmbeddings { inner, embeddings }));      // :170 有嵌入列→Yes
        }
    };
    Ok(Self::No(inner))                                                      // :175 否则原样透传
}
```

> **设计点：零开销的 `No` 分支**。没有嵌入列（或没传名册）时返回 `MaybeEmbedded::No(inner)`，它的 `next()`（`:245`）直接调 `inner.next()` 透传，**没有任何额外拷贝/计算**。这是 enum 双态包装器的经典优化——不用为"普通表"付一分钱嵌入开销。贡献者改这块时务必保留这条 fast path。

> **Rust 知识点：`if let ColumnKind::Embedding(embedding_def) = &cd.kind`**。这是**模式匹配 + 解构**：判断 `cd.kind` 是不是 `Embedding` 变体，**同时**把里面的 `EmbeddingDefinition` 绑定到 `embedding_def`。一行同时干"判断类型"和"取出内容"两件事，类似 Python 的 `match case Embedding(d):`。

### 5.3 `WithEmbeddings`：加料工位本身就是一条传送带

`embeddings.rs:125`：

```rust
pub struct WithEmbeddings<R: RecordBatchReader> {
    inner: R,                                                          // :126 被包裹的原传送带
    embeddings: Vec<(EmbeddingDefinition, Arc<dyn EmbeddingFunction>)>, // :127 列定义↔翻译机 配对
}
```

魔法在于它**同时实现了 `Iterator` 和 `RecordBatchReader`**（`embeddings.rs:259` 和 `:301`）——所以它"既拥有一条传送带，又自己是一条传送带"，能无缝替换掉原传送带插回流水线。这是**装饰器模式**。

> **Rust 知识点：`impl<R: RecordBatchReader> Iterator for WithEmbeddings<R>`**。这是"为泛型类型实现 trait"。`<R: RecordBatchReader>` 表示"对任意实现了 `RecordBatchReader` 的 `R`，下面这套实现都成立"。`RecordBatchReader` 在 Arrow 里本质就是 `Iterator<Item = Result<RecordBatch>>` + 一个 `schema()` 方法，所以同时实现两者就让 `WithEmbeddings` 变成一个完整的、带 schema 的传送带。

核心是 `next()`（`embeddings.rs:262`）——**每读一盘餐，就加宽一盘**：

```rust
fn next(&mut self) -> Option<Self::Item> {
    let batch = self.inner.next()?;                                   // :263 从原传送带取一盘(RecordBatch)
    match batch {
        Ok(mut batch) => {
            // todo: parallelize this                                 // :266 ← 目前串行，见 §7
            for (fld, func) in self.embeddings.iter() {               // :267 每个嵌入定义来一遍
                let src_column = batch.column_by_name(&fld.source_column).unwrap();  // :268 取源列(文字)
                let embedding = match func.compute_source_embeddings(src_column.clone()) {  // :269 算向量!
                    Ok(embedding) => embedding,
                    Err(e) => return Some(Err(ArrowError::ComputeError(...))),       // :272 出错包成 Arrow 错误
                };
                let dst_field_name = fld.dest_column.clone()
                    .unwrap_or_else(|| format!("{}_embedding", &fld.source_column)); // :281 目标列名(默认规则)
                let dst_field = Field::new(dst_field_name, embedding.data_type().clone(), ...); // :283
                match batch.try_with_column(dst_field.clone(), embedding) {          // :289 把向量列追加进这盘
                    Ok(b) => batch = b,
                    Err(e) => return Some(Err(e)),
                };
            }
            Some(Ok(batch))                                          // :294 吐出已加宽的盘
        }
        Err(e) => Some(Err(e)),
    }
}
```

一句话：**取源列 → `compute_source_embeddings` 算向量 → `try_with_column` 追加成新列 → 把加宽的 batch 吐回传送带**。下游 lance 写入时看到的就是已经带向量列的数据。

> **目标列名默认规则**：`dest_column` 是 `None` 时，列名 = `format!("{}_embedding", source_column)`（`:281`）。这和 `dest_fields()` 里推断 schema 时的规则（`:199`）**严格一致**——否则"声明的列名"和"实际产出的列名"会对不上。改其中一处必须同步改另一处。

### 5.4 建表时一次性嵌入（非流式分支）

写入是流式（每 batch 算一次）的，但**建表带嵌入**走的是稍不同的分支（`connection.rs:179`）：直接 `WithEmbeddings::new(data, self.embeddings)` 把整个数据 reader 包成带嵌入的 reader 写入。

> **限制**：建表带嵌入**不支持流式输入**——`connection.rs:176` 在输入是 streaming 时直接返回 `Error::NotSupported`（文本 `"Creating a table with embeddings is currently not support when the input is streaming"`）。这是一处明确的能力边界，可能是潜在的贡献点。

---

## 6. 三家实现对比：远程 API vs 本地模型

三家都实现了同一个 `EmbeddingFunction` trait，所以能互换。但内部"后厨设备"截然不同。

### 6.1 三家的"门面"对比

| 维度 | OpenAI (`openai.rs`) | Bedrock (`bedrock.rs`) | sentence-transformers (`sentence_transformers.rs`) |
|---|---|---|---|
| `name()` | `"openai"` (`:144`) | `"bedrock"` (`:75`) | `"sentence-transformers"` (`:407`) |
| 后厨类型 | 外卖：async-openai HTTP | 外卖：AWS SDK `invoke_model` | 自家灶台：candle 本地跑 BERT |
| 模型 enum | `EmbeddingModel` (`:22`) ada-002/3-small/3-large | `BedrockEmbeddingModel` (`:20`) Titan/CohereLarge | 无 enum，模型是字符串(默认 `all-MiniLM-L6-v2`) |
| 维度 | 1536/1536/3072 (`:30`) | 1536/1024 (`:26`) | 运行时探测(`compute_ndims_and_dtype` `:208`) |
| `source_type` | `Utf8` | `Utf8` | `Utf8` |
| `dest_type` | `FixedSizeList<Float32,N>` 非空 | 同左 | `FixedSizeList<dtype,N>` 非空(dtype 也探测) |
| 持有的"凭证" | `api_key`/`api_base`/`org_id` (`:71`) | `client: BedrockClient` (`:55`) | `model: BertModel`/`tokenizer`(`:47`) |
| 跨异步 | `block_in_place`+`block_on` (`:245`) | `block_in_place`+`block_on` (`:170`) | 纯本地 CPU，无 `block_on`(但同步阻塞) |
| 构造 | `new(api_key)` / `new_with_model` / 链式 `api_base()`/`org_id()` | `new(client)` / `with_model` | 点菜单式 builder (`:23`)，链式 9 个方法 |

> **核心洞见**：三个 struct 字段、构造方式、内部逻辑天差地别，但**对外都是 `Arc<dyn EmbeddingFunction>`**。注册进同一本名册、走同一条 `WithEmbeddings` 流水线，`MaybeEmbedded` 完全不在乎背后是 HTTP 还是本地推理。这正是 **trait object（`Arc<dyn EmbeddingFunction>`）的价值**：让"调用方"和"具体实现"彻底解耦。

### 6.2 OpenAI：打电话点外卖

真正干活的私有方法 `compute_inner`（`openai.rs:184`）：先校验非空 + Utf8（`:186`/`:193`）→ 构造 `OpenAIConfig`（`:199`）→ 把 Arrow 字符串列收集成 `Vec<String>` 包成 `EmbeddingInput::StringArray`（`:208`）→ 构造请求（`:236`）→ **跨进异步运行时调 HTTP**（`:245`）→ 把返回的每个向量 append 进 `Float32Builder`（`:253`）。

```rust
// TODO: request batching and retry logic                            // :244 ← 上手任务，见 §7
task::block_in_place(move || {                                       // :245
    Handle::current().block_on(async {
        let res = embed.create(req).await.map_err(...)?;             // :249 真正的 HTTP 请求在这
        for Embedding { embedding, .. } in res.data.iter() {
            builder.append_slice(embedding);                        // :254
        }
        Ok(builder.finish())
    })
})
```

> **Rust 陷阱（本篇最易绊倒的点）：`task::block_in_place` + `Handle::current().block_on`**。`Iterator::next` 是**同步**签名（不能 `await`），但底层要调**异步**的 HTTP 客户端。这两行就是"在同步函数里跑异步代码"的桥：
> - `block_in_place`：告诉 tokio"我这个 worker 线程要长时间阻塞了，请把它身上其他待办任务挪到别的线程去"，避免饿死整个运行时。
> - `block_on`：同步地等这个 async future 跑完，把结果拿回来。
>
> **类比**：Python 里在同步函数中 `asyncio.run(coro)`；JS 里把 `await` 包成同步等待。**陷阱**：这要求"当前正运行在 tokio 多线程运行时里"，否则 `block_in_place` 会 panic。

### 6.3 Bedrock：逐条点外卖

`compute_inner`（`bedrock.rs:122`）同样先校验，再收集 texts（`:137`），然后 **`for text` 逐条**（`:151`，非批量）按模型构造请求体（Titan 用 `{inputText}`、Cohere 用 `{texts:[..], input_type:"search_document"}`），同样 `block_in_place`+`block_on` 调 `client.invoke_model()...send().await`（`:170`），解析 JSON 取字段（Titan 取 `response["embedding"]`、Cohere 取 `response["embeddings"][0]`，`:189`）。

> **注意一处糙代码**：`bedrock.rs:182` 对 `send()` 的结果直接 `.unwrap()`（HTTP 失败会 panic 而非返回 `Err`）。这和 OpenAI 用 `map_err` 优雅传播错误的写法不一致——是个可以改进的点。

### 6.4 sentence-transformers：自己开火

这家最重（474 行），因为它真的在本地跑神经网络。

**先下货（builder.build()，`sentence_transformers.rs:142`）**：默认模型 `all-MiniLM-L6-v2`，拼成 `sentence-transformers/{}`（`:143`），从 hf-hub 下载 `config.json`/`tokenizer.json`/`model.safetensors`（`:159`），然后用 `unsafe VarBuilder::from_mmaped_safetensors`（`:184`）把权重文件 **mmap** 进内存，加载成 `BertModel`（`:185`）。

**再推理（`compute_inner`，`:240`）**：分词成 `Tensor` → `Tensor::stack` 成一批（`:310`）→ `model.forward` 跑 BERT（`:313`）→ 搬到 CPU（`:316`）→ **mean pooling**：`embeddings.sum(1) / n_tokens`（`:324`，把每个 token 的向量取平均得到句向量）→ 按 storage 的 dtype（U8/U32/I64/F16/F32/F64）分派 `from_cpu_storage` 转成 Arrow 数组（`:330`）。

> **Rust 陷阱：`unsafe VarBuilder::from_mmaped_safetensors`（`:184`）**。`mmap` 把权重文件直接映射进内存（省去读进堆的拷贝，几百 MB 的模型秒加载）。`unsafe` 是因为 mmap 的内存安全**编译器保证不了**（文件可能被外部进程改动）。**`unsafe` 不等于"有 bug"**，而是"编译器让你自己对这块内存安全负责"。

> **几个会 panic 的边角**：BF16 storage 直接 `panic!`（`:394`）、非 2 维输出 `todo!()`（`:399`）、`Utf8View` 输入返回 `Error::Runtime`（`:298`，尚未实现）。这些都是"还没填的坑"，贡献者要留意别让它们出现在用户能轻易触发的热路径。

### 6.5 三家的一个共同细节：手工拼 FixedSizeList

三家的 `compute_source_embeddings` 都**不用** `FixedSizeListBuilder`，而是手工 `ArrayData::builder(fsl).len(len).add_child_data(inner.into_data()).build()`（`openai.rs:169`、`bedrock.rs:98`、`sentence_transformers.rs:432`）。

> **WHY（注释 `openai.rs:167` 说明）**：`FixedSizeListBuilder` **总会加一个 null bitmap**（标记哪些元素是 null 的位图），但向量列要求**非空**（`dest_type` 里 `false`）。所以这里绕过高层 builder，直接拼 Arrow 底层 `ArrayData`：一个 `FixedSizeList` 由"子数组 + 固定长度"组成，把内层 1D 浮点数组当 child、套上固定长度 N，就得到一个非空的定长向量列。这是 Arrow 内存布局的细节。

---

## 7. 如何贡献一个新 embedding provider

要加一家新供应商（比如 Cohere 直连、本地 ONNX），**照抄三家模板即可**，五步：

1. **新建 `embeddings/yourprovider.rs`**，定义：
   - 一个 model `enum`（带 `ndims()` / 可选 `model_id()` / `impl FromStr`）——参考 `openai.rs:22` 或 `bedrock.rs:20`。
   - 一个 struct，持有凭证/client（API 型）或模型句柄（本地型）。**如果有不可打印或敏感字段，手写 `Debug`**——参考 OpenAI 对 api_key 脱敏（`openai.rs:78`）、Bedrock 跳过不实现 Debug 的 client（`bedrock.rs:112`）、sentence-transformers 跳过巨大的 model（`sentence_transformers.rs:54`）。
2. **`impl EmbeddingFunction`**：实现 `name`/`source_type`/`dest_type`/`compute_source_embeddings`/`compute_query_embeddings` 五个方法。
3. **一个私有 `compute_inner`** 做真正的 IO/推理。API 型记得用 `block_in_place`+`block_on` 把 async 调用塞进同步上下文。
4. **在 `embeddings.rs` 顶部加 feature gate**（参考 `embeddings.rs:4-11`）：

```rust
#[cfg(feature = "yourprovider")]
pub mod yourprovider;
```

5. **在 `Cargo.toml` 加 feature**，列出依赖（参考 `openai = [dep:async-openai, dep:reqwest]`、`bedrock = [dep:aws-sdk-bedrockruntime]`、`sentence-transformers = [dep:hf-hub, dep:candle-core, ...]`）。

> **Rust 知识点：`#[cfg(feature = "openai")]`（条件编译）**。三家实现各自躲在一个 cargo feature 后面（`embeddings.rs:4/7/10`）。不开 `openai` feature，`openai.rs` 整个文件就不参与编译，async-openai 也不会被拉进依赖。**好处**：用本地模型的人不必背 AWS SDK 的编译开销，反之亦然。类比：可选的 npm `peerDependencies` / Python 的 `extras_require`。

### 现成的"上手任务"（good first issue 信号）

源码里散落着标好的 TODO，是绝佳的练手切入点：

| 位置 | TODO | 难度提示 |
|---|---|---|
| `embeddings.rs:266` | `todo: parallelize this`——`WithEmbeddings::next` 目前串行算每个嵌入定义 | 中：引入并行需注意线程安全 |
| `openai.rs:244` | `request batching and retry logic`——OpenAI 调用无批处理/无重试 | 中：网络重试是经典需求 |
| `sentence_transformers.rs:264` | 分词可用 rayon 并行 | 中 |
| `sentence_transformers.rs:298` | `Utf8View not yet implemented` | 小：补一个输入类型分支 |
| `connection.rs:176` | 建表带嵌入不支持流式输入 | 大：涉及流式语义 |
| `bedrock.rs:182` | `send()` 直接 `unwrap()`，应改成 `map_err` 传播 | 小：错误处理改进 |

---

## 8. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `trait T: A + B` | 接口 + supertrait 约束 | `EmbeddingFunction` `embeddings.rs:45` |
| `Send + Sync` | 跨线程安全的"通行证"，编译器强制 | `:45`、`:81` |
| `Arc<dyn Trait>` | 引用计数 + 运行时多态；trait object | 全篇 |
| `Arc<RwLock<HashMap>>` | 共享 + 多读单写锁 + 数据，三层各司其职 | `MemoryRegistry` `:94` |
| 内部可变性 | `&self` 也能改内容，可变性藏在 `RwLock` 里 | `register`/`get` `:101`/`:110` |
| `Cow<DataType>` | Clone-On-Write：值可能借来也可能拥有 | `source_type`/`dest_type` `:48`/`:51` |
| `enum Yes(...)/No(...)` | 双态包装器，`No` 分支零开销透传 | `MaybeEmbedded` `:131` |
| `if let Pattern(x) = &val` | 模式匹配 + 解构，一行判类型 + 取内容 | `try_new` `:151` |
| `impl<R: Bound> Trait for T<R>` | 为泛型类型实现 trait（装饰器） | `WithEmbeddings` `:259`/`:301` |
| `block_in_place` + `block_on` | 在同步函数里跑异步（最易绊倒点） | `openai.rs:245`、`bedrock.rs:170` |
| `#[cfg(feature = "x")]` | 条件编译，把可选实现关在 feature 后 | `embeddings.rs:4/7/10` |
| `unsafe { ... }` | 编译器保证不了内存安全，由你负责 | `from_mmaped_safetensors` `sentence_transformers.rs:184` |
| `derive(Serialize, Deserialize)` | 自动生成序列化代码，让定义能存进磁盘 | `EmbeddingDefinition` `:59` |
| `todo!()` / `panic!()` / `.unwrap()` | 三个会 panic 的宏/方法，热路径慎用 | `sentence_transformers.rs:394/399`、`bedrock.rs:182` |

---

## 9. 动手验证（建议亲手做一遍）

1. **数清两个 trait**：打开 `embeddings.rs`，跳到 `:45` 和 `:81`，对照 §2/§3，确认全模块的"抽象"就这两个 trait + 一个默认注册表实现。问自己：为什么翻译机实例不能序列化，而 `EmbeddingDefinition` 能？
2. **追"向量凭空多出来"**：从 `table.rs:1957`（`NativeTable::add`）出发，跟到 `MaybeEmbedded::try_new`（`embeddings.rs:143`），再到 `WithEmbeddings::next`（`:262`）。把 §5 那条链路在源码里逐跳走一遍，重点看 `:289` 那行 `try_with_column` 是怎么把向量列追加进 batch 的。
3. **验证零开销 No 分支**：读 `MaybeEmbedded` 的 `Iterator::next`（`:242`）和 `RecordBatchReader::schema`（`:250`），确认 `No(inner)` 变体只是原样转发，没有任何嵌入开销。
4. **对比三家的 `block_on`**：并排打开 `openai.rs:245` 和 `bedrock.rs:170`，再看 `sentence_transformers.rs:240`（无 `block_on`）。理解为什么前两家需要"同步里跑异步"而本地推理不需要。
5. **找一个 TODO 上手**：挑 §7 表格里的一个（比如 `sentence_transformers.rs:298` 的 `Utf8View`），读懂上下文，想想要补哪个分支——这就是你的第一个 PR 雏形。

---

## 10. 小结 & 下一篇

- **整个模块只有两个抽象**：`EmbeddingFunction`（翻译机规格书，`embeddings.rs:45`）+ `EmbeddingRegistry`（设备名册，`:81`，默认实现 `MemoryRegistry` `:93`）。其余都是胶水。
- **"自动算向量"靠装饰器缝进流水线**：`WithEmbeddings`（`:125`）既拥有又是一条 `RecordBatchReader`，`next()` 每读一盘就 `compute_source_embeddings` 加一列向量（`:262`）；`MaybeEmbedded`（`:131`）是零开销分流闸。**没改 lance 一行**——"组装而非重造"的标本。
- **定义随 schema 持久化**：`ColumnKind::Embedding(EmbeddingDefinition)`（`table.rs:92`）把"哪列用哪台翻译机"序列化进 schema 元数据，写入时 `MaybeEmbedded::try_new` 据此凭名字从注册表找回翻译机。
- **同一 trait，三种后厨**：OpenAI/Bedrock 是 HTTP 外卖（`block_in_place`+`block_on` 跨异步），sentence-transformers 是本地 candle 灶台。互不依赖，各藏在自己的 feature 后。
- **贡献新 provider** = 照抄模板：model enum + struct + `impl EmbeddingFunction` + 私有 `compute_inner` + feature gate + Cargo.toml。源码里现成的 TODO 是绝佳起手式。

**下一篇**将沿着"请求的一生"链路视角，看查询侧如何触发 `compute_query_embeddings`、以及向量进入查询计划后的去向，把本篇刻意留白的"查询路径触发点"补上。
