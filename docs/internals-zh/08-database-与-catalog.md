# 08 · Database 与 Catalog 抽象层：连接层的两级"用工合同"

> **本篇解剖的源码**：
> - `rust/lancedb/src/database.rs`（168 行，`Database` trait 与请求类型）
> - `rust/lancedb/src/database/listing.rs`（627 行，`ListingDatabase` 落地实现）
> - `rust/lancedb/src/catalog.rs`（87 行，`Catalog` trait 与请求类型）
> - `rust/lancedb/src/catalog/listing.rs`（前 285 行，`ListingCatalog` 落地实现）
>
> **覆盖区间**：连接层的两级抽象 `Catalog → Database → Table` 的全部 trait 定义、请求类型、以及 listing 后端的逐方法实现。
>
> **前置阅读**：`01 Rust 垫脚石`（尤其 trait / `dyn` / `Arc` / async 四节）、`03 连接建立链路`（`connect()` 怎么走到这一层）。本篇所有论断都带真实行号，形如 `database.rs:151`，可点击跳转。
>
> **与哪些篇互补**：`09/10 Table 解剖`讲 `Database::create_table` 返回的 `Arc<dyn BaseTable>` 内部长什么样——本篇只到"把活儿甩给 `NativeTable`"为止，不往下钻。`04 写入链路`讲一次 `create_table` 的端到端时序——本篇专注连接层这一段的结构本身。

---

## 0. 这一篇要回答的三个问题

如果你只带走三句话，就是这三句：

1. **连接层也是一份"用工合同"**——`Database` 是个 trait（接口），规定"想当数据库必须会做哪 7 件事"。本地目录、S3、云端各签同一份合同，上层永不写 `if 本地 else 云端`。
2. **抽象有两级，结构完全对称**——`Catalog` 管多个 `Database`，`Database` 管多个 `Table`。`Catalog : Database` 这层关系，长得跟 `Database : Table` 那层一模一样。
3. **listing 实现土到家，但够用**——一个目录就是一个数据库，目录里每个 `xxx.lance` 文件夹就是一张表。"列表"（list directory）这个动作就是它名字的来历，列目录 + 过滤后缀 + 排序 + 分页，全在客户端做。

把这三句话画成一张图，就是整个连接层的骨架：

```
                你的代码
       connect("/data").execute()
                   │
                   ▼
        ┌────────────────────────┐
        │  Catalog (连锁总部)      │  catalog.rs:68   ← 管"有哪些 database"
        │  6 个方法的 trait        │
        └───────────┬────────────┘
            create_database / open_database
                    │  返回 Arc<dyn Database>
                    ▼
        ┌────────────────────────┐
        │  Database (用工合同)     │  database.rs:151  ← 管"有哪些 table"
        │  7 个方法的 trait        │
        └───────────┬────────────┘
            create_table / open_table
                    │  返回 Arc<dyn BaseTable>
                    ▼
        ┌────────────────────────┐
        │  BaseTable (见 09 篇)    │  table.rs:408     ← 管"表里的数据"
        └────────────────────────┘

   两种后端各自实现这两份 trait：
   listing（本地/S3）         remote（LanceDB Cloud）
   ListingCatalog            （无 Catalog，直连 Database）
   ListingDatabase           RemoteDatabase
       │ 委托                      │ 发 HTTP
       ▼                          ▼
   NativeTable / lance        云端服务
```

> **比喻总钥匙**：`Database` trait 是**后厨用工合同**——规定一家店必须会做 7 件事（列菜单、上新菜、开张、改名、撤菜……）。`Catalog` trait 则是**连锁总部**——它不管单道菜，只管"旗下有哪些门店"。`ListingDatabase` 是个**靠 `ls` 认菜的土经理**：不记账本，每次要列菜单就跑去厨房 `ls` 一遍，看到 `.lance` 结尾的就当作一道菜。

本篇顺序：先看为什么连接层要 trait（§1）→ 读 `Database` 合同条款（§2）→ 看围绕它的"点菜单"请求类型（§3）→ 进 `ListingDatabase` 后厨看怎么落地（§4）→ 上一层看 `Catalog` 怎么嵌套委托（§5）→ 三层一图回顾（§6）。

---

## 1. 为什么连接层也要 trait？

`database.rs` 顶部的模块注释（`database.rs:4-15`）把动机说得很直白：

> 一个"数据库"是个通用概念，表示某种"管理表及其元数据"的东西。我们提供一个最简实现：基于"列目录"。但用户可能想自己实现，因为——表可能在 S3 上以不同方式排布、表可能由某个独立应用（如 Postgres）管理、或者要用自定义表实现（如远程表）。

翻译成给贡献者的话：**LanceDB 不想把"表存在哪、怎么发现表"这件事写死**。它把这件事抽象成一个 trait，于是支持三种后端而互不打架：

| 后端 | 实现类型 | "表在哪" | 在哪个文件 |
|---|---|---|---|
| 本地文件系统 | `ListingDatabase` | 本地目录里的 `.lance` 文件夹 | `database/listing.rs:203` |
| 云对象存储（S3/GCS/Azure） | `ListingDatabase`（同上） | 桶里的 `.lance` 前缀 | 同上，靠 `object_store` 适配 |
| LanceDB Cloud | `RemoteDatabase` | 远程服务，发 HTTP 问 | `remote/db.rs:176`（`cfg(feature = "remote")`） |

注意：**前两种共用同一个 `ListingDatabase`**——本地和 S3 的区别被更底层的 `object_store` 抹平了，`ListingDatabase` 根本不关心自己面对的是磁盘还是对象桶。第三种 `RemoteDatabase` 才真正换了实现。

> **多态隔离的价值**：上层代码（`Connection`）只持有 `Arc<dyn Database>`（`connection.rs:849` 把 `ListingDatabase` 装进 `Arc` 后就只当它是 `dyn Database`）。要从本地切到云端，只需换掉躲在 `Arc<dyn Database>` 后面的实现，上层一行不改。这跟 `09 篇`里 `Table` 持有 `Arc<dyn BaseTable>` 是**同一个套路在更高一层重演**。

> **Rust 知识点：`dyn Trait` 是什么？**（写给不懂 Rust 的你）
> `dyn Database` 表示"某个实现了 `Database` 这个接口的具体类型，但编译期不知道是哪个"。运行时通过一张"虚函数表"找到真正该调的方法——和 Java 接口引用、Go interface、Python 鸭子类型是一回事。`Arc<dyn Database>` 则在此之上再加"原子引用计数的共享指针"，让多个线程能安全地共享同一个数据库对象。

---

## 2. `Database` trait：连接层的"用工合同"

`database.rs:150-167`：

```rust
#[async_trait::async_trait]                                    // ① 让 trait 里能写 async fn 的宏
pub trait Database:
    Send + Sync + std::any::Any + std::fmt::Debug + std::fmt::Display + 'static  // ② supertrait 约束
{
    async fn table_names(&self, request: TableNamesRequest) -> Result<Vec<String>>;        // :155
    async fn create_table(&self, request: CreateTableRequest) -> Result<Arc<dyn BaseTable>>; // :157
    async fn open_table(&self, request: OpenTableRequest) -> Result<Arc<dyn BaseTable>>;     // :159
    async fn rename_table(&self, old_name: &str, new_name: &str) -> Result<()>;             // :161
    async fn drop_table(&self, name: &str) -> Result<()>;                                    // :163
    async fn drop_all_tables(&self) -> Result<()>;                                           // :165
    fn as_any(&self) -> &dyn std::any::Any;                                                  // :166
}
```

整个合同就 **7 个方法**（前 6 个是 `async`，最后一个 `as_any` 是同步的）。理解了它，你就理解了 LanceDB 连接层为什么能"本地/云端一套代码"。

### 2.1 先拆 `trait` 那一行的每个零件

`database.rs:151-153` 那行 supertrait 约束（冒号后那一串）比 `09 篇`的 `BaseTable` 还多两个，逐个拆：

| 零件 | 含义 | 类比你熟悉的语言 |
|---|---|---|
| `Send + Sync` | 能安全地跨线程传递/共享，并发的"通行证" | "线程安全标记" |
| `std::any::Any` | 携带运行时类型信息，给"向下转型"用（配合 `as_any`） | Java 的 `Object` + `instanceof` |
| `std::fmt::Debug` | 能被 `{:?}` 打印（给程序员调试看） | 调试用的 `repr()` |
| `std::fmt::Display` | 能被 `{}` 打印（给人看，如 `ListingDatabase(uri=...)`） | `toString()` |
| `'static` | 不借用任何短命引用，能活到"程序需要它多久就多久" | 无直接对应；理解成"不挂任何会过期的外部引用" |
| `#[async_trait::async_trait]` | 一个宏，把 trait 里的 `async fn` 改写成返回 `Box<dyn Future>` 的普通方法 | 无直接对应；理解成"让接口方法能 async 的胶水" |

> **对照 `09 篇`的一个细节**：`BaseTable`（`table.rs:408`）的约束是 `Display + Debug + Send + Sync`，而 `Database` 多了 `Any` 和 `'static`。多出来的 `Any` 正是为了支持最后那个 `as_any` 方法（见 §2.3）。

> **Rust 陷阱：`#[async_trait]` 宏到底干了啥？**
> Rust 的原生 trait 当年对 `async fn` 支持有限。这个宏把每个 `async fn foo(&self) -> T` 偷偷改写成 `fn foo(&self) -> Pin<Box<dyn Future<Output = T> + Send>>`——也就是把"异步函数"变成"返回一个装在堆上的 Future 对象的普通函数"。类比 Java 的 `CompletableFuture<T>`：方法立刻返回一个"将来会有结果的盒子"，你 `.await`（≈`.get()`）时才真正等结果。代价是每次调用有一次堆分配，但换来了"接口里能写 async"的便利。

### 2.2 7 个方法逐条解读

| 行号 | 方法 | 作用 | 返回什么 |
|---|---|---|---|
| `:155` | `table_names(req)` | 列出库里所有表名，支持分页 | `Vec<String>` |
| `:157` | `create_table(req)` | 建表（或按 mode 处理重名），写入初始数据 | **`Arc<dyn BaseTable>`** |
| `:159` | `open_table(req)` | 打开已有表 | **`Arc<dyn BaseTable>`** |
| `:161` | `rename_table(old, new)` | 改表名 | `()` |
| `:163` | `drop_table(name)` | 删一张表 | `()` |
| `:165` | `drop_all_tables()` | 清空整库 | `()` |
| `:166` | `as_any()` | 取 `&dyn Any`，给向下转型用 | `&dyn Any` |

**两个最关键的方法是 `create_table` 和 `open_table`，因为它们的返回类型是 `Arc<dyn BaseTable>`**——这就是两级抽象的"接缝"：`Database` 不返回某个具体的 `NativeTable`，而是返回"某个实现了 `BaseTable` 合同的东西"。上层（`Connection`）拿到后包成 `09 篇`讲的 `Table` 门面，至此连接层的 trait 和表层的 trait 严丝合缝地接上了。

> **设计要点：返回 trait 对象，不返回具体类型**。如果 `create_table` 直接返回 `NativeTable`，那 `RemoteDatabase` 就没法实现这个签名了（它要返回的是远程表）。返回 `Arc<dyn BaseTable>` 让两种后端都能各自返回自己的表实现，而调用方完全无感。**这是"组装而非重造"在接口设计上的体现：合同只规定"还你一张表"，不规定"还你哪种表"。**

### 2.3 `as_any`：为什么合同里要塞一个"露底"方法

`as_any`（`database.rs:166`）看起来很怪——一个抽象层的接口，为什么要提供"把自己变回具体类型"的能力？

因为有时上层确实需要拿到具体的 `ListingDatabase`（比如读它的 `uri` 这种 listing 特有的字段）。但手里是 `Arc<dyn Database>`，要"拆开看里面到底是不是 ListingDatabase"，唯一安全途径就是 `as_any()` 拿到 `&dyn Any`，再 `downcast_ref::<ListingDatabase>()` 试着转回去——成功给 `Some`，失败给 `None`，绝不崩溃。

`ListingDatabase` 的实现就一行（`database/listing.rs:623-625`）：

```rust
fn as_any(&self) -> &dyn std::any::Any {
    self                                  // self 已是 &Self，自动当成 &dyn Any 返回
}
```

> **Rust 知识点：`as_any` + `downcast_ref` 配对**。这是 trait 对象（`dyn`）想变回具体类型的标准套路，和 `09 篇`里 `BaseTable::as_any` 完全一样。supertrait 约束里那个 `Any`（§2.1）就是为了让这一步成立——没有 `Any` 约束，编译器不让你 `downcast`。

---

## 3. 围绕 `Database` 的请求类型：把参数打包成"点菜单"

`Database` 的几个方法不直接收一堆散参数，而是收一个**请求结构体**（`...Request`）。这是 LanceDB 反复使用的设计：**用结构体 + 枚举打包参数，避免布尔参数地狱和函数重载**。逐个看。

### 3.1 `TableNamesRequest`：分页请求

`database.rs:38-48`：

```rust
#[derive(Clone, Debug, Default)]
pub struct TableNamesRequest {
    pub start_after: Option<String>,   // :45 只返回字典序在此之后的名字（分页游标）
    pub limit: Option<u32>,            // :47 最多返回几个
}
```

`#[derive(Default)]` 让你能写 `TableNamesRequest::default()` 拿到"两个字段都是 `None`"的实例——也就是"不分页，全要"。`drop_all_tables` 内部就是这么用的（见 §4.6）。

### 3.2 `OpenTableRequest`：打开表的参数

`database.rs:51-56`：

```rust
#[derive(Clone, Debug)]
pub struct OpenTableRequest {
    pub name: String,                              // :53 表名
    pub index_cache_size: Option<u32>,             // :54 索引缓存大小
    pub lance_read_params: Option<ReadParams>,     // :55 直接透传给 lance 的读参数
}
```

注意 `lance_read_params` 是 `Option<ReadParams>`——`ReadParams` 是 lance 引擎的类型。LanceDB 在这里**开了个"逃生口"**：高级用户可以绕过 LanceDB 的封装，直接把 lance 的读参数塞进来。这个字段在 §4.5 的 `open_table` 里有"用户提供的优先，否则用默认值现搭"的取舍逻辑。

### 3.3 `CreateTableMode`：用枚举消灭"布尔参数地狱"

建表时遇到"表已存在"怎么办？有三种策略。LanceDB 不用 `bool overwrite, bool error_if_exists` 这种容易传错的参数，而是用一个枚举一次说清（`database.rs:62-85`）：

```rust
pub enum CreateTableMode {
    Create,                              // :64 已存在就报错（默认）
    ExistOk(TableBuilderCallback),       // :68 已存在就打开它；忽略提供的数据
    Overwrite,                           // :70 已存在就覆盖
}

impl CreateTableMode {
    pub fn exist_ok(                     // :74 一个便捷构造器，帮你把闭包装进 Box
        callback: impl FnOnce(OpenTableRequest) -> OpenTableRequest + Send + 'static,
    ) -> Self {
        Self::ExistOk(Box::new(callback))
    }
}

impl Default for CreateTableMode {       // :81
    fn default() -> Self { Self::Create }  // 默认 = 报错，最安全
}
```

最有意思的是 `ExistOk` 还**带一个回调** `TableBuilderCallback`（`database.rs:58`）：

```rust
pub type TableBuilderCallback = Box<dyn FnOnce(OpenTableRequest) -> OpenTableRequest + Send>;
```

它的意思是："如果表已存在、要改成打开它，那打开时怎么配置，由你这个闭包说了算"。§4.4 会看到它被真正调用。

> **Rust 陷阱：`Box<dyn FnOnce(...) -> ...>` 是什么？**
> 这是"装在堆上的、只能调用一次的闭包"。`FnOnce` 表示"调一次就消耗掉"（类比一张一次性兑换券），`Box<dyn ...>` 表示"我不知道你传的闭包具体是什么类型，先装箱起来"。类比 Java：传一个 `Function<OpenTableRequest, OpenTableRequest>` lambda 进来，延迟到需要时才执行。`Send` 约束保证这个闭包能跨线程搬。

> **设计要点：Mode 枚举 vs 布尔参数**。`CreateTableMode` 把"三选一"编码成一个不可能传出矛盾组合的枚举（你没法同时"报错"又"覆盖"）。这比 `create_table(overwrite=true, error_if_exists=true)` 这种自相矛盾都能编译过的布尔签名安全得多。**加新后端时，只要 `match` 这个枚举的三个分支，编译器会强制你处理完整。**

### 3.4 `CreateTableData`：三种食材入口，统一成一条传菜流水线

建表的数据可能有三种来源（`database.rs:88-105`）：

```rust
pub enum CreateTableData {
    Data(Box<dyn RecordBatchReader + Send>),    // :90 一个同步迭代器（schema 从数据推断）
    StreamingData(SendableRecordBatchStream),   // :92 一个异步流（schema 从数据推断）
    Empty(TableDefinition),                      // :94 空表，只给 schema，没有数据行
}

impl CreateTableData {
    pub fn schema(&self) -> Arc<arrow_schema::Schema> {   // :98 三种来源都能问出 schema
        match self {
            Self::Data(reader) => reader.schema(),
            Self::StreamingData(stream) => stream.schema(),
            Self::Empty(definition) => definition.schema.clone(),
        }
    }
}
```

精彩的是下面这段：三种异构入口被统一成 DataFusion 的流（`database.rs:107-122`）：

```rust
#[async_trait]
impl StreamingWriteSource for CreateTableData {              // :108 实现 lance 的"可流式写入源"
    fn into_stream(self) -> ...SendableRecordBatchStream {
        match self {
            Self::Data(reader) => reader.into_stream(),                       // 同步迭代器 → 流
            Self::StreamingData(stream) => stream.into_df_stream(),           // 已是流，转一下类型
            Self::Empty(table_definition) => {                               // :116 空表造个"空流"
                let schema = table_definition.schema.clone();
                Box::pin(RecordBatchStreamAdapter::new(schema, stream::empty()))
            }
        }
    }
}
```

> **比喻**：`CreateTableData` 是**三种食材入口**——预切好的盒装菜（`Data`）、传送带上源源不断的菜（`StreamingData`）、空盘子只贴了菜名标签（`Empty`）。但厨房后段只认一种格式：**DataFusion 的传菜流水线（`SendableRecordBatchStream`，即 Arrow 餐盘流）**。`into_stream` 就是那个把三种入口统一接到流水线上的转接头。特别地，空表用 `stream::empty()` 造一个"零行但有 schema"的空流——schema 还在（盘子标签还在），只是没有任何 Arrow 餐盘流过。

### 3.5 `CreateTableRequest`：把上面四样打包

`database.rs:125-145`：

```rust
pub struct CreateTableRequest {
    pub name: String,                  // :127 新表名
    pub data: CreateTableData,         // :129 §3.4 那三选一
    pub mode: CreateTableMode,         // :131 §3.3 那三选一
    pub write_options: WriteOptions,   // :133 写参数（仅在有数据时用到）
}

impl CreateTableRequest {
    pub fn new(name: String, data: CreateTableData) -> Self {   // :137 便捷构造：mode 和 write_options 取默认
        Self { name, data, mode: CreateTableMode::default(), write_options: WriteOptions::default() }
    }
}
```

`new` 只要"名字 + 数据"两个必填项，其余取默认。这是 Rust 表达"可选参数"的惯用法之一（另一个是 `09 篇`的 Builder）。

---

## 4. `ListingDatabase`：一个目录就是一个数据库

现在进后厨。`ListingDatabase` 是 `Database` trait 的唯一 OSS 实现，定义在 `database/listing.rs:203-220`：

```rust
#[derive(Debug)]
pub struct ListingDatabase {
    object_store: ObjectStore,                                  // :204 底层存储抽象（磁盘 or S3，统一接口）
    query_string: Option<String>,                              // :205 URI 上的查询串，要透传给 lance
    pub(crate) uri: String,                                    // :207 数据库根 uri
    pub(crate) base_path: object_store::path::Path,            // :208 根路径（object_store 的路径类型）
    pub(crate) store_wrapper: Option<Arc<dyn WrappingObjectStore>>,  // :211 写路径的存储包装器（如镜像写）
    read_consistency_interval: Option<std::time::Duration>,    // :213 读一致性刷新间隔（透传给每张表）
    storage_options: HashMap<String, String>,                  // :216 存储配置，被本库所有表继承
    new_table_config: NewTableConfig,                          // :219 本库建新表时的默认配置
}
```

它的 `Display`（`database/listing.rs:222-238`）只打印 `uri` 和 `read_consistency_interval`，兑现 §2.1 那个 `Display` 约束。

> **比喻**：`ListingDatabase` 是个**不记账本的土经理**。它没有"表清单"这种内部状态——`object_store` 和 `base_path` 加起来只是告诉它"厨房在哪"。每次有人问"有哪些菜"，它现跑去 `ls` 一遍。好处是无需维护一致性（账本不会和实际脱节）；代价是目录一大就慢（见 §4.2 的提醒）。

### 4.1 `table_uri`：把表名拼成路径

几乎所有方法第一步都是把表名拼成完整 uri。`table_uri`（`database/listing.rs:390-412`）：

```rust
fn table_uri(&self, name: &str) -> Result<String> {
    validate_table_name(name)?;                                       // :391 ① 先校验表名合法性
    let path = Path::new(&self.uri);
    let table_uri = path.join(format!("{}.{}", name, LANCE_FILE_EXTENSION));  // :393 ② 拼成 "<root>/<name>.lance"
    let mut uri = table_uri.as_path().to_str()
        .context(InvalidTableNameSnafu { name, reason: "Name is not valid URL" })?  // :396 ③ 转字符串，失败给上下文错误
        .to_string();
    if let Some(query) = self.query_string.as_ref() {                // :405 ④ 把连接级查询串追加回去
        uri.push('?');
        uri.push_str(query.as_str());
    }
    Ok(uri)
}
```

第 ① 步 `validate_table_name`（`utils.rs:83`）用一个正则 `TABLE_NAME_REGEX = ^[a-zA-Z0-9_\-\.]+$`（`utils.rs:22`）把关：表名只能含字母数字、下划线、连字符、点，且不能为空。否则返回 `Error::InvalidTableName`（`error.rs:13`）。

> **Rust 陷阱：`.context(XxxSnafu { ... })?`**
> 这是 `snafu` 错误库的惯用法。`to_str()` 返回 `Option`，`.context(...)` 把"`None`"翻译成一个带上下文（表名、原因）的具体错误，再用 `?` 上抛。类比：给一个可能为空的结果"附上失败时该报什么错"，空了就抛出去。`InvalidTableNameSnafu` 是 `#[derive(Snafu)]`（`error.rs:9`）为 `Error::InvalidTableName` 自动生成的"上下文选择器"。

### 4.2 `table_names`：列目录 + 过滤 + 排序 + 客户端分页

这是"listing"名字的来历。`database/listing.rs:448-476`：

```rust
async fn table_names(&self, request: TableNamesRequest) -> Result<Vec<String>> {
    let mut f = self.object_store
        .read_dir(self.base_path.clone()).await?                    // :451 ① 列出根目录下所有条目
        .iter().map(Path::new)
        .filter(|path| {                                            // :455 ② 只保留 .lance 后缀的
            let is_lance = path.extension().and_then(|e| e.to_str()).map(|e| e == LANCE_EXTENSION);
            is_lance.unwrap_or(false)
        })
        .filter_map(|p| p.file_stem().and_then(|s| s.to_str().map(String::from)))  // :462 ③ 取文件名去掉 .lance
        .collect::<Vec<String>>();
    f.sort();                                                       // :464 ④ 字典序排序
    if let Some(start_after) = request.start_after {                // :465 ⑤ 分页游标：丢掉 <= start_after 的
        let index = f.iter().position(|name| name.as_str() > start_after.as_str()).unwrap_or(f.len());
        f.drain(0..index);
    }
    if let Some(limit) = request.limit {                            // :472 ⑥ 截断到 limit 个
        f.truncate(limit as usize);
    }
    Ok(f)
}
```

六步：列目录 → 过滤 `.lance` → 去后缀取表名 → 排序 → 丢掉游标之前的 → 截断。

> **给贡献者的提醒：分页是纯客户端的**。注意第 ⑤⑥ 步——`read_dir` 一次性把**整个目录都拉回来**了，分页（`start_after`/`limit`）只是在内存里 `drain` 和 `truncate`。对几十张表的目录毫无问题，但对**超大目录（成千上万张表）不高效**：每次列表都要拉全量再丢弃。如果你将来要优化这块，这就是切入点——但要注意 `object_store::read_dir` 本身是否支持服务端分页。

> **Rust 知识点：迭代器链（iterator chain）**。`.iter().map(...).filter(...).filter_map(...).collect()` 是 Rust 的"流式处理"惯用法，类比 Java Stream / Python 生成器表达式。`filter_map` 是"过滤 + 映射二合一"：闭包返回 `Option`，`None` 的被丢、`Some(x)` 的留下 `x`。

### 4.3 `create_table`：继承补缺 + 委托 `NativeTable` + 处理重名

这是本篇最长的方法（`database/listing.rs:478-562`），但拆开看只有四段。

**第一段：storage_options 继承补缺（不覆盖）** `database/listing.rs:481-493`：

```rust
let storage_options = request.write_options.lance_write_params
    .get_or_insert_with(Default::default)        // :484 ① 一路 get_or_insert_with 钻进嵌套结构，缺啥补啥
    .store_params.get_or_insert_with(Default::default)
    .storage_options.get_or_insert_with(Default::default);
for (key, value) in self.storage_options.iter() {   // :489 ② 把连接级的存储配置补进去
    if !storage_options.contains_key(key) {         // :490 ★ 只补缺，不覆盖：用户在 request 里设的优先
        storage_options.insert(key.clone(), value.clone());
    }
}
```

`:490` 那个 `if !contains_key` 是**铁律**：连接级的 `storage_options`（建连接时设的）只用来填补 request 里**没设**的键，绝不覆盖用户在本次 request 里显式设的值。`open_table`（§4.5）也是同样规则（`:576`）。

> **Rust 知识点：`get_or_insert_with(Default::default)` 链**。`Option::get_or_insert_with` 的意思是"如果是 `None` 就用闭包造一个默认值塞进去，然后返回里面那个值的可变引用"。一路链下来，相当于"沿着一条可能处处为空的嵌套路径，缺哪一层补哪一层默认值，最后拿到最里层的可变引用"。类比 Python 的 `dict.setdefault` 串成一条链。

**第二段：决定 new_table 配置，带向后兼容回退** `database/listing.rs:499-517`：

```rust
if let Some(storage_version) = &self.new_table_config.data_storage_version {
    write_params.data_storage_version = Some(*storage_version);   // :500 优先用结构化配置
} else if let Some(v) = storage_options.get(OPT_NEW_TABLE_STORAGE_VERSION) {  // :503 否则回退到旧的字符串 key
    write_params.data_storage_version = Some(v.parse()?);
}
// enable_v2_manifest_paths 同理（:507-517）
```

这段是**向后兼容**：早期版本通过 storage_options 里的字符串 key（如 `new_table_data_storage_version`，常量在 `database/listing.rs:32`）来配置，新版本改用结构化的 `NewTableConfig`。代码优先读结构化配置，读不到才回退翻旧 key。

**第三段：Overwrite 翻译成 lance 的 WriteMode** `database/listing.rs:519-521`：

```rust
if matches!(&request.mode, CreateTableMode::Overwrite) {
    write_params.mode = WriteMode::Overwrite;     // :520 把 LanceDB 的 mode 翻译成 lance 的 WriteMode
}
```

**第四段：委托 `NativeTable::create`，然后 `match` 错误处理重名** `database/listing.rs:525-561`：

```rust
match NativeTable::create(&table_uri, &request.name, request.data,
    self.store_wrapper.clone(), Some(write_params), self.read_consistency_interval).await   // :525 ★ 真正建表甩给 NativeTable
{
    Ok(table) => Ok(Arc::new(table)),                                  // :535 成功，装进 Arc<dyn BaseTable>
    Err(Error::TableAlreadyExists { name }) => match request.mode {    // :536 已存在？看 mode 怎么办
        CreateTableMode::Create => Err(Error::TableAlreadyExists { name }),  // :537 报错（默认）
        CreateTableMode::ExistOk(callback) => { /* 见 §4.4 */ }              // :538
        CreateTableMode::Overwrite => unreachable!(),                        // :558 不可能到这
    },
    Err(err) => Err(err),                                              // :560 其它错误原样上抛
}
```

> **"组装而非重造"的字面证据**：`:525` 这一行把建表的真活儿整个甩给了 `NativeTable::create`（即 lance）。`ListingDatabase` 自己只干了"拼 uri、补配置、翻译 mode、处理重名"这些**编排**工作，存储引擎一行没写。

> **为什么 `Overwrite => unreachable!()`？**（`:558`）因为第三段已经把 `Overwrite` 翻译成 `WriteMode::Overwrite` 了——lance 在覆盖模式下根本不会抛 `TableAlreadyExists`（它会直接覆盖）。所以"已存在错误 + Overwrite 模式"这个组合在控制流上不可能出现。`unreachable!()` 是 Rust 标注"此处控制流不变量保证到不了"的宏，真到了会 panic（说明上游逻辑有 bug）。

### 4.4 `ExistOk` 回调的真正调用点

`database/listing.rs:538-557` 展开第四段里 `ExistOk` 那个分支，看 §3.3 的回调怎么被用：

```rust
CreateTableMode::ExistOk(callback) => {
    let req = OpenTableRequest {                       // :539 ① 造一个默认的打开请求
        name: request.name.clone(),
        index_cache_size: None,
        lance_read_params: None,
    };
    let req = (callback)(req);                          // :544 ② 让用户的闭包改写这个请求
    let table = self.open_table(req).await?;            // :545 ③ 打开已存在的表
    let table_schema = table.schema().await?;
    if table_schema != data_schema {                   // :549 ④ 校验：已存在表的 schema 必须和你想建的一致
        return Err(Error::Schema {
            message: "Provided schema does not match existing table schema".to_string(),
        });
    }
    Ok(table)                                          // :556 ⑤ 一致就返回这张已有表
}
```

这正是 §3.3 说的：表已存在时，"打开它"的配置由用户那个 `FnOnce` 闭包说了算（`:544` 调用它），但 LanceDB 额外加了一道"schema 必须匹配"的安全检查（`:549`），防止你以为在建 A 表却悄悄拿到了一张 schema 不同的旧表。

### 4.5 `open_table`：继承补缺 + 读参数取舍 + 委托 `NativeTable`

`database/listing.rs:564-606`，结构和 `create_table` 前半段对称：

```rust
async fn open_table(&self, mut request: OpenTableRequest) -> Result<Arc<dyn BaseTable>> {
    let table_uri = self.table_uri(&request.name)?;
    // 同样的 storage_options 继承补缺（:568-579，规则同 §4.3，只补缺不覆盖）
    let read_params = request.lance_read_params.unwrap_or_else(|| {   // :587 ★ 用户给了 ReadParams 就用，没给才现搭默认
        let mut default_params = ReadParams::default();
        if let Some(index_cache_size) = request.index_cache_size {
            default_params.index_cache_size = index_cache_size as usize;
        }
        default_params
    });
    let native_table = Arc::new(
        NativeTable::open_with_params(&table_uri, &request.name,      // :596 ★ 打开真活儿甩给 NativeTable
            self.store_wrapper.clone(), Some(read_params), self.read_consistency_interval).await?,
    );
    Ok(native_table)
}
```

`:587` 的取舍是 §3.2 那个"逃生口"的兑现：**用户直接给的 `lance_read_params` 优先级最高**；没给，才用 `OpenTableRequest` 里的 `index_cache_size` 等高层选项现搭一个默认 `ReadParams`。`:596` 又一次"甩给 `NativeTable`"。

### 4.6 `rename_table` / `drop_table` / `drop_all_tables`

`database/listing.rs:608-621`：

```rust
async fn rename_table(&self, _old_name: &str, _new_name: &str) -> Result<()> {
    Err(Error::NotSupported {                                          // :609 listing 后端不支持改名
        message: "rename_table is not supported in LanceDB OSS".to_string(),
    })
}
async fn drop_table(&self, name: &str) -> Result<()> {
    self.drop_tables(vec![name.to_string()]).await                     // :615 转调批量删
}
async fn drop_all_tables(&self) -> Result<()> {
    let tables = self.table_names(TableNamesRequest::default()).await?;  // :619 先列全表
    self.drop_tables(tables).await                                      // :620 再批量删
}
```

`drop_all_tables` 复用了 `table_names`（用 `default()` 拿全量，§3.1）。真正的删除在 `drop_tables`（`database/listing.rs:414-443`）：它先用 `commit_handler.delete(...)`（`:428`，让 lance 的提交处理器删提交记录）再 `object_store.remove_dir_all(...)`（`:430`，删目录），并把 lance 的 `NotFound` 翻译成 `Error::TableNotFound`（`:436`）。

> **关键洞察：`NotSupported` 是实现选择，不是接口限制**。`rename_table` 在 listing 后端返回 `NotSupported`（`database/listing.rs:608`），但**同一个 `Database` 合同**在 `RemoteDatabase` 里被真正实现了（`remote/db.rs:381` 有真实的 `rename_table` 方法体，整个 `impl<S: HttpSend> Database for RemoteDatabase<S>` 在 `remote/db.rs:249`，受 `cfg(feature = "remote")` 控制）。这证明：**trait 把"能改名"写进了合同，但每个后端可以选择"我做不到"**——能力的差异藏在实现里，不在接口里。贡献者要给 listing 加 rename 支持，只需替换这个方法体，合同一字不改。

---

## 5. `Catalog` trait：比 `Database` 更高一层

`Database` 管"一堆表"。再往上，`Catalog` 管"一堆 database"。结构和 `Database` **几乎镜像对称**。

`catalog.rs:67-86`：

```rust
#[async_trait]
pub trait Catalog: Send + Sync + std::fmt::Debug + 'static {   // :68 ← 注意：比 Database 少了 Any 和 Display
    async fn database_names(&self, request: DatabaseNamesRequest) -> Result<Vec<String>>;       // :70
    async fn create_database(&self, request: CreateDatabaseRequest) -> Result<Arc<dyn Database>>; // :73
    async fn open_database(&self, request: OpenDatabaseRequest) -> Result<Arc<dyn Database>>;     // :76
    async fn rename_database(&self, old_name: &str, new_name: &str) -> Result<()>;               // :79
    async fn drop_database(&self, name: &str) -> Result<()>;                                      // :82
    async fn drop_all_databases(&self) -> Result<()>;                                             // :85
}
```

把它和 §2 的 `Database` 并排看，对称关系一目了然：

| `Database`（管表） | `Catalog`（管库） | 返回类型 |
|---|---|---|
| `table_names` | `database_names` | `Vec<String>` |
| `create_table` → `Arc<dyn BaseTable>` | `create_database` → `Arc<dyn Database>` | **trait 对象** |
| `open_table` → `Arc<dyn BaseTable>` | `open_database` → `Arc<dyn Database>` | **trait 对象** |
| `rename_table` | `rename_database` | `()` |
| `drop_table` | `drop_database` | `()` |
| `drop_all_tables` | `drop_all_databases` | `()` |

> **教学锚点：两层对称**。`Catalog : Database` 这层关系，和 `Database : Table` 那层结构完全一样——上层 trait 的 `create_X` 返回下层的 `Arc<dyn Y>`。理解了其中一层，另一层免费送。这种"自相似"的分层正是 LanceDB 连接层好懂的原因。

> **两个不对称的小差异**（贡献时容易踩）：
> 1. `Catalog` 的 supertrait 约束（`catalog.rs:68`）是 `Send + Sync + Debug + 'static`，**比 `Database` 少了 `Any` 和 `Display`**——因为目前没有"把 `dyn Catalog` 向下转型"或"`{}` 打印 catalog"的需求。
> 2. `CreateDatabaseMode`（`catalog.rs:42-49`）的 `ExistOk` 变体**不带回调**，而 `CreateTableMode::ExistOk`（`database.rs:68`）带。开库比开表简单，不需要"已存在时怎么打开"的自定义逻辑。

### 5.1 `Catalog` 的请求类型

和 `Database` 那套平行（`catalog.rs:21-65`）：

```rust
#[derive(Clone, Debug, Default)]
pub struct DatabaseNamesRequest {              // :21  对应 TableNamesRequest
    pub start_after: Option<String>,           // :23  分页游标
    pub limit: Option<u32>,                    // :25
}
#[derive(Clone, Debug)]
pub struct OpenDatabaseRequest {               // :30  对应 OpenTableRequest
    pub name: String,                          // :32
    pub database_options: HashMap<String, String>,  // :36  库级选项（透传给底层 Database 实现）
}
pub enum CreateDatabaseMode { Create, ExistOk, Overwrite }   // :42  对应 CreateTableMode（但 ExistOk 不带回调）
pub struct CreateDatabaseRequest {             // :58  对应 CreateTableRequest
    pub name: String,                          // :59
    pub mode: CreateDatabaseMode,              // :62
    pub options: HashMap<String, String>,      // :64
}
```

### 5.2 `ListingCatalog`：一个目录 = 一堆 database，每个子目录 = 一个 database

`catalog/listing.rs:82-91`：

```rust
#[derive(Debug)]
pub struct ListingCatalog {
    object_store: ObjectStore,             // :84 同 ListingDatabase 的套路
    uri: String,                           // :86
    base_path: ObjectStorePath,            // :88
    options: ListingCatalogOptions,        // :90 内含 db_options（见下）
}
```

它的实现思路（`catalog/listing.rs:77-81` 的文档注释说得很清楚）：**基础目录下的每个子文件夹就是一个 database**，会被开成一个 `ListingDatabase`。于是层级是：

```
/catalog_root            ← ListingCatalog 的 base_path
  /db1                   ← 一个 database（ListingDatabase）
    /tableA.lance        ← db1 里的一张表
    /tableB.lance
  /db2                   ← 另一个 database
    /tableC.lance
```

`database_names`（`catalog/listing.rs:162-184`）和 `ListingDatabase::table_names` 几乎逐行一样：`read_dir` → 取 `file_name`（不过滤后缀，因为子目录就是库名）→ 排序 → `start_after` drain → `limit` truncate。**同一套"列目录 + 排序 + 客户端分页"，只是过滤规则不同**。

### 5.3 嵌套委托：`create_database` 怎么"填一张标准申请表，交给开店流程"

最能体现"两层对称"的是 `create_database`（`catalog/listing.rs:186-229`）——它创建一个 database 的方式，是**给 `ListingDatabase` 填一张 `ConnectRequest` 申请表，再走 `ListingDatabase::connect_with_options` 这个"开店流程"**：

```rust
async fn create_database(&self, request: CreateDatabaseRequest) -> Result<Arc<dyn Database>> {
    let db_path = self.database_path(&request.name);                  // :187 算出子目录路径（database_path 在 :155）
    let exists = Path::new(&to_local_path(&db_path)).exists();        // :189 这个子目录在不在
    match request.mode {                                              // :191 ① 按 mode 处理重名
        CreateDatabaseMode::Create if exists =>
            return Err(Error::DatabaseAlreadyExists { name: request.name }),  // :192/:193 已存在就报错
        CreateDatabaseMode::Create => { create_dir_all(db_path.to_string()).unwrap(); }  // :195/:196 建目录
        CreateDatabaseMode::ExistOk => { if !exists { create_dir_all(...).unwrap(); } }  // :198
        CreateDatabaseMode::Overwrite => {                            // :203 先删旧的再建
            if exists { self.drop_database(&request.name).await?; }
            create_dir_all(db_path.to_string()).unwrap();
        }
    }
    let db_uri = format!("/{}/{}", self.base_path, request.name);     // :211 ② 拼出新库的 uri
    let mut connect_request = ConnectRequest {                       // :213 ③ 填一张"连接申请表"
        uri: db_uri,
        read_consistency_interval: None,
        options: Default::default(),
        #[cfg(feature = "remote")] client_config: Default::default(),
    };
    self.options.db_options.serialize_into_map(&mut connect_request.options);  // :222 ④ 把 catalog 的库级选项灌进表
    Ok(Arc::new(
        ListingDatabase::connect_with_options(&connect_request).await?,        // :226 ⑤ ★ 嵌套委托：走开店流程
    ))
}
```

> **比喻**：`ListingCatalog` 是**连锁总部**，要开新店时它不亲自垒墙——它先按 mode 处理"这地址已有店了吗"（`:191`），然后**填一张标准的开店申请表（`ConnectRequest`，`:213`），把总部规定的标准配置（`db_options`）抄进表里（`:222`），最后交给开店流程（`ListingDatabase::connect_with_options`，`:226`）去真正开张**。总部自己只管"批地、填表、传达标准"，不碰具体施工。

`open_database`（`catalog/listing.rs:231-256`）是同样的嵌套委托，只是先检查目录存在性，不存在就报 `Error::DatabaseNotFound`（`:236/:237`），存在则同样填 `ConnectRequest` 交给 `connect_with_options`。

> **`ConnectRequest`：连接层的"万能申请表"**（`connection.rs:603-630`）。它有 4 个字段：`uri`（:611）、`options`（:617，库/catalog 特定选项）、`read_consistency_interval`（:629），以及 `cfg(remote)` 下的 `client_config`。无论是顶层 `connect()`（`connection.rs:846-857`）、还是 catalog 内部开库（`:226`），都填这同一张表交给同一个 `connect_with_options`——**一张表通吃所有连接入口**。

> **给贡献者的提醒：这里有几处 `unwrap()` 潜在 panic**。`:196`、`:200`、`:207`、`:108`（`open_path` 里的 `ObjectStore::from_path(path).unwrap()`）都直接 `unwrap`，意味着如果建目录/解析路径失败会 panic 而非返回 `Result`。这块有完整单测覆盖（`catalog/listing.rs:287-616`），但若你要加固错误处理，这些 `unwrap` 是候选。

### 5.4 `ListingCatalogOptions`：选项直接委托给 `ListingDatabaseOptions`

`catalog/listing.rs:30-53`，catalog 的选项干脆**整个套住** database 的选项：

```rust
#[derive(Clone, Debug, Default)]
pub struct ListingCatalogOptions {
    pub db_options: ListingDatabaseOptions,    // :35 catalog 的选项 = 包一层 database 的选项
}
impl CatalogOptions for ListingCatalogOptions {
    fn serialize_into_map(&self, map: &mut HashMap<String, String>) {
        self.db_options.serialize_into_map(map);   // :40 序列化直接转交 db_options
    }
}
impl ListingCatalogOptions {
    pub(crate) fn parse_from_map(map: &HashMap<String, String>) -> Result<Self> {
        let db_options = ListingDatabaseOptions::parse_from_map(map)?;   // :50 解析也直接转交
        Ok(Self { db_options })
    }
}
```

> **加一个新 option 要动几处？**（贡献者必读）以 `ListingDatabaseOptions` 为例，一个 option 从定义到生效要打通**五处**，漏一处就"传不通"：
> 1. **字段**：在 `NewTableConfig`（`database/listing.rs:37`）或 `ListingDatabaseOptions` 加字段；
> 2. **`parse_from_map`**（`database/listing.rs:69`）：从 `HashMap<String,String>` 解析出来；
> 3. **`serialize_into_map`**（`database/listing.rs:104`）：序列化回去（这样能跨 `ConnectRequest` 传递）；
> 4. **Builder**（`database/listing.rs:121` 起的 `ListingDatabaseOptionsBuilder`）：给用户一个链式设置入口；
> 5. **消费点**：在 `create_table`/`open_table` 里真正读这个字段去影响行为。
>
> `parse_from_map`（`:69`）有个值得记的约定：它先抽走两个已知 key（`OPT_NEW_TABLE_STORAGE_VERSION` `:32`、`OPT_NEW_TABLE_V2_MANIFEST_PATHS` `:33`），**剩下的 map 项一律当作 storage_options**（`:87-95`）。所以你加的新 key 如果不在这两个常量里，会被当成存储配置透传给 `object_store`。

---

## 6. 三层抽象一图回顾

把 Catalog / Database / Table 三层各自"管什么"画在一起：

```
┌──────────────────────────────────────────────────────────────────┐
│  Catalog  (连锁总部)                              catalog.rs:68      │
│  管: 有哪些 DATABASE                                                 │
│  方法: database_names / create_database / open_database / ...        │
│  实现: ListingCatalog（子目录=database）/ catalog/listing.rs:83       │
│  create_database 返回 ──────────────┐                               │
└─────────────────────────────────────┼──────────────────────────────┘
                                       │ Arc<dyn Database>
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  Database (用工合同/门店)                          database.rs:151    │
│  管: 有哪些 TABLE                                                    │
│  方法: table_names / create_table / open_table / drop_table / ...    │
│  实现: ListingDatabase（.lance 文件夹=table）/ database/listing.rs:203│
│        RemoteDatabase（HTTP）         / remote/db.rs:176             │
│  create_table 返回 ─────────────────┐                               │
└─────────────────────────────────────┼──────────────────────────────┘
                                       │ Arc<dyn BaseTable>
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  BaseTable (见 09/10 篇)                            table.rs:408     │
│  管: 表里的 数据 / 索引 / 版本                                        │
│  实现: NativeTable（lance）/ RemoteTable（HTTP）                      │
└──────────────────────────────────────────────────────────────────┘
```

三层的共性，是读懂整个连接层的钥匙：

| 层 | trait | "上层造下层"的方法 | 返回 | listing 实现的核心动作 |
|---|---|---|---|---|
| Catalog | `catalog.rs:68` | `create_database` | `Arc<dyn Database>` | `read_dir` 子目录 + 嵌套委托 `connect_with_options` |
| Database | `database.rs:151` | `create_table` | `Arc<dyn BaseTable>` | `read_dir` `.lance` + 甩给 `NativeTable::create` |
| Table | `table.rs:408` | （见 09 篇） | 数据流 | 拿锁 + 甩给 lance |

> **一句话总结整篇**：连接层是**两级对称的 trait 抽象**（Catalog 对 Database = Database 对 Table），listing 实现用"**列目录 + 过滤 + 排序 + 客户端分页**"落地"有哪些下级"，用"**填申请表嵌套委托 / 甩给 NativeTable**"落地"创建下级"。加一个新后端，你要做的只是实现 `Database` trait 的 7 个方法（或 `Catalog` 的 6 个），**编排现成零件，不重造存储引擎**。

---

## 7. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `trait T: A + B + ...` | 接口 + supertrait 约束（实现前必须先满足 A、B） | `Database` `database.rs:151` |
| `#[async_trait]` | 让 trait 能写 `async fn` 的宏（改写成返回 `Box<dyn Future>`） | `database.rs:150` |
| `Send + Sync + 'static` | 跨线程安全 + 不挂短命引用，编译器强制 | `database.rs:152` |
| `Arc<dyn Trait>` | 引用计数 + 运行时多态；`clone` 只 +1 计数 | 全篇 |
| `as_any()` + `downcast_ref` | `dyn` 变回具体类型的唯一安全途径（需 `Any` 约束） | `database.rs:166` |
| `enum` 编码"多选一" | 用枚举消灭布尔参数地狱（`CreateTableMode` 三态） | `database.rs:62` |
| `match self { ... }` | 对枚举穷尽分支，编译器强制处理完整 | `database.rs:99`、`listing.rs:536` |
| `Box<dyn FnOnce(..) -> ..>` | 装箱的一次性闭包（延迟执行的回调） | `TableBuilderCallback` `database.rs:58` |
| `Option::get_or_insert_with(Default::default)` | 缺则补默认值，返回内部可变引用（类比 `setdefault`） | `listing.rs:484` |
| `.context(XxxSnafu { .. })?` | 把 `Option`/`Err` 翻译成带上下文的错误再上抛 | `listing.rs:396` |
| `matches!(x, Pat)` | 判断某值是否匹配某模式，返回 `bool` | `listing.rs:519` |
| `unreachable!()` | 标注控制流到不了；真到了就 panic | `listing.rs:558` |
| 迭代器链 `.filter().filter_map().collect()` | 流式处理（类比 Java Stream） | `listing.rs:455-463` |
| `#[cfg(feature = "remote")]` | 条件编译：仅当启用某 feature 才编译这段 | `connection.rs:613`、`remote/db.rs:249` |

---

## 8. 动手验证（建议亲手做一遍）

1. **数一数两份合同**：打开 `database.rs:151` 和 `catalog.rs:68`，对照 §6 的表格，确认 `Database` 是 7 个方法、`Catalog` 是 6 个，并找出 `Catalog` 的 supertrait 约束比 `Database` 少了哪两个（`Any` 和 `Display`）。问自己：为什么 catalog 不需要这两个？
2. **抓"两层对称"**：把 `database.rs:155-165` 的 6 个 async 方法和 `catalog.rs:70-85` 的 6 个并排，逐个找出"管表"对"管库"的镜像方法名。
3. **跟一次 `table_names`**：从 `database/listing.rs:448` 读到 `:476`，在脑中模拟一个含 `["b.lance", "a.lance", "notes.txt"]` 的目录、`start_after=Some("a")`、`limit=Some(5)` 时，返回值是什么。（答案：过滤掉 `notes.txt`，排序成 `["a","b"]`，`start_after="a"` 丢掉 `a`，得 `["b"]`。）
4. **找"组装而非重造"的字面证据**：在 `database/listing.rs` 搜 `NativeTable::`，确认 `create_table`（`:525`）和 `open_table`（`:596`）都把真活儿甩给了 `NativeTable`。再去 `catalog/listing.rs:226` 看 `create_database` 怎么甩给 `ListingDatabase::connect_with_options`。
5. **验证"接口不限制能力"**：对比 `database/listing.rs:608`（listing 的 `rename_table` 返回 `NotSupported`）和 `remote/db.rs:381`（remote 真实现了 `rename_table`）。理解：同一份合同，两种后端可以做出不同的能力选择。

---

## 9. 小结 & 下一篇

- **连接层是两级对称的 trait 抽象**：`Catalog`（`catalog.rs:68`，管多个 database）→ `Database`（`database.rs:151`，管多个 table）→ `Table`（`table.rs:408`，管数据）。每层的 `create_X` 都返回下层的 `Arc<dyn Y>`，这就是层与层的"接缝"。
- **请求类型用结构体 + Mode 枚举打包参数**：`CreateTableRequest` / `CreateTableMode`（三态：Create/ExistOk/Overwrite）/ `CreateTableData`（三种食材入口统一成 DataFusion 流），消灭布尔参数地狱。
- **`ListingDatabase` / `ListingCatalog` 是"靠 `ls` 认菜的土实现"**：列目录 + 过滤 + 排序 + **纯客户端**分页落地"有哪些下级"；创建则**填 `ConnectRequest` 嵌套委托 / 甩给 `NativeTable`**——组装而非重造。
- **能力差异在实现里，不在接口里**：`rename` 在 listing 返回 `NotSupported`，在 `RemoteDatabase` 真实现。加新后端只需实现 trait 的那几个方法。

**接下来**：本篇在 `create_table` 返回 `Arc<dyn BaseTable>` 处止步。想知道那个 `BaseTable` 内部长什么样、`NativeTable::create` 拿到数据后怎么拿锁、怎么甩给 lance 落盘——请读 **`09 Table 解剖（上）`** 和 **`10 Table 解剖（下）`**。想看一次 `create_table` 调用从用户代码穿到磁盘的完整时序，请读 **`04 写入链路`**。
