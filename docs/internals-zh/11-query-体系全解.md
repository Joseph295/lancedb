# 11 · Query 体系全解：点菜单的三层 trait 与两种订单

> **本篇解剖的源码**：`rust/lancedb/src/query.rs`（全 1735 行）
> **覆盖区间**：整个文件 —— `Select` / `IntoQueryVector` / `QueryBase` / `HasQuery` / `ExecutableQuery` 三层 trait，`QueryRequest` / `VectorQueryRequest` 两种订单，`Query` / `VectorQuery` 两种 Builder，以及 `execute_hybrid` 的手工编排
> **前置阅读**：`01 Rust 垫脚石`（trait / `dyn` / `Arc` / async 四节）、`09 Table 解剖（上）`（`BaseTable` 合同、Builder 模式、`Arc<dyn BaseTable>`）
> **与哪些篇互补**：本篇是**模块深潜**，把"点菜单怎么搭"讲透；`06 查询链路`是**链路篇**，讲"点完单后 `execute()` 怎么一跳跳进 lance scanner 出结果"。两篇在 `create_plan` 这个交界处握手——本篇讲到 `self.parent.create_plan(...)` 就收尾，`06` 从那里接着往后厨钻。
>
> 本篇所有论断都带真实行号，形如 `query.rs:457`，可点击跳转。

---

## 0. 这一篇要回答的问题

`query.rs` 是用户构建查询的**全部公共 API**所在地。但读它最容易犯的错，是把它当成"查询引擎"——以为距离计算、索引检索都在这里。**全错。** 这个文件 1735 行，**一行真正的扫描代码都没有**。

如果你只带走四句话，就是这四句：

1. **`query.rs` 是「点菜单」，不是「厨房」**——它只收集参数、提供链式 API，真正下厨（扫描 / 索引 / 距离计算）全在 lance。
2. **查询拆成两种「订单」**：普通扫描用 `QueryRequest`，向量搜索用 `VectorQueryRequest`（内嵌前者 + 向量专属字段）。
3. **配置能力用三层 trait 切分**：`QueryBase`（通用旋钮）+ `HasQuery`（桥）+ `ExecutableQuery`（执行入口），靠一个 blanket impl 让两种 Builder **零重复**地共享通用配置。
4. **`nearest_to` 是一道单向门**：普通查询一旦升级成向量查询，就解锁 `nprobes`/`refine_factor` 等旋钮，但回不去了。

把这些画成一张图，就是整个 Query 体系的骨架：

```
                你的代码
         table.query().limit(10).nearest_to(vec)?.nprobes(30).execute()
                   │
        ┌──────────┴───────────┐
        ▼                      ▼
 ┌─────────────┐  nearest_to  ┌──────────────────┐
 │   Query     │ ───────────▶ │   VectorQuery    │   ← 单向门（query.rs:734）
 │ (普通点菜单) │   单向升级    │ (向量搜索点菜单)  │
 │ query.rs:680│              │  query.rs:852    │
 └──────┬──────┘              └────────┬─────────┘
        │ parent: Arc<dyn BaseTable>   │ parent: Arc<dyn BaseTable>
        │ request: QueryRequest        │ request: VectorQueryRequest
        ▼                              ▼
 ┌─────────────┐              ┌──────────────────────────────┐
 │QueryRequest │◀─── base ────│ VectorQueryRequest           │
 │ 10 个字段    │   内嵌       │  base: QueryRequest          │
 │ query.rs:612│              │  + 9 个向量字段(nprobes...)   │
 └─────────────┘              │  query.rs:790                │
   「普通订单」                 └──────────────────────────────┘
                                        「向量订单」

   三层 trait 横切在两种 Builder 上：
   ┌────────────────────────────────────────────────────┐
   │ QueryBase     通用旋钮 limit/offset/only_if/select… │ query.rs:334
   │   ▲ blanket impl<T: HasQuery>（query.rs:457）       │
   │ HasQuery      只需暴露 mut_query() -> &mut QueryRequest │ query.rs:453
   │ ExecutableQuery  execute/explain_plan/analyze_plan │ query.rs:546
   └────────────────────────────────────────────────────┘
                          │ execute()
                          ▼
            AnyQuery::Query / AnyQuery::VectorQuery   ← 打包成统一枚举
                          │  (table.rs:398)
                          ▼
            self.parent.query(&any, options)   ← 递给后厨 BaseTable
                          ▼
                    lance scanner  (链路篇 06 接手)
```

> **比喻总钥匙**（延续 `09` 的家族）：`Query`/`VectorQuery` 是两张**点菜单**（Builder），客人在上面勾选（`limit`/`filter`/`nearest_to`），但菜单本身不做菜。`QueryRequest`/`VectorQueryRequest` 是**订单小票**（纯数据，可复印、可传真到云端分店）。`QueryBase` 是摆在公共台面的**通用调料台**，任何点菜单都能取用。`IntoQueryVector` 是**翻译官**，把客人的母语（`Vec<f32>`）译成后厨听得懂的 Arrow 餐盘。最后菜单把勾好的订单（`AnyQuery`）递进后厨（`BaseTable`），真正下厨在 lance（后厨灶台）。

本篇按"先讲能选什么列（`Select`）→ 再讲三层 trait → 再讲两种 Builder 的升级与旋钮 → 最后讲执行如何落到后厨"的顺序展开。

---

## 1. 总览：为什么查询要拆成三层 trait + 两种订单

在动手剖任何类型之前，先理解**这个设计要解决什么矛盾**——否则你会觉得"为什么不直接写一个 `Query` struct 把所有方法塞进去"。

### 1.1 矛盾一：普通扫描和向量搜索，公共参数太多

`SELECT * WHERE x > 10 LIMIT 5` 是普通扫描；"找最近的 10 个向量，但只在 `x > 10` 的行里找"是向量搜索。两者都需要 `limit`、`offset`、`filter`、`select`（选哪些列）、`with_row_id` 这些**通用配置**；但只有向量搜索需要 `nprobes`、`refine_factor`、`distance_type` 这些**向量专属旋钮**。

如果写成两个独立 struct，通用配置要复制两遍（`Query::limit` 和 `VectorQuery::limit` 各写一份）。LanceDB 的解法是：把通用配置抽成 `QueryBase` trait，用 **blanket impl** 一次性套到两种 Builder 上（§4）。**贡献者加一个通用配置项，只需改三处：`QueryRequest` 加字段 + `QueryBase` 加方法签名 + blanket impl 加一行**，两种 Builder 自动都有了。

### 1.2 矛盾二：「参数数据」和「执行能力」要能分开

`RemoteTable`（云端表）需要把查询参数**序列化成 JSON 发到云端**；测试代码需要直接**断言查询字段**（"我设了 `nprobes=30`，字段里是不是 30"）。这两件事都只关心"参数是什么"，不关心"怎么执行"。

于是 LanceDB 把每种查询拆成**两层**：

| 层 | 类型 | 角色 | 能力 |
|---|---|---|---|
| 数据层 | `QueryRequest`(:612) / `VectorQueryRequest`(:790) | 「订单小票」 | 纯数据，`Clone`、可 `into_request()` 拿出来检视 / 序列化 / 测试断言 |
| 包装层 | `Query`(:680) / `VectorQuery`(:852) | 「点菜单」 | 数据 + 一根 `Arc<dyn BaseTable>` 后厨指针，能 `execute()` |

> **WHY 分两层？** 把"参数"和"后厨指针"分开，订单小票（Request）就能脱离具体的表独立存在——序列化发往云端、跨进程传递、被测试直接构造。`Query`/`VectorQuery` 则在小票外面套了一根后厨电话（`parent`），只有它能下单执行。

### 1.3 三层 trait 各管什么

| trait | 行号 | 职责 | 一句话 |
|---|---|---|---|
| `QueryBase` | :334 | 通用配置（11 个链式方法） | "公共调料台" |
| `HasQuery` | :453 | 只暴露 `mut_query()` 一个方法 | "把调料台接到具体菜单上的桥" |
| `ExecutableQuery` | :546 | 执行入口（`create_plan`/`execute`/`explain_plan`/`analyze_plan`） | "喊一声下单" |

三者的协作：`Query` 和 `VectorQuery` 各自实现 `HasQuery`（暴露自己内部的 `QueryRequest`）和 `ExecutableQuery`（各自的执行逻辑），而 `QueryBase` 通过 blanket impl **自动**套到任何 `HasQuery` 实现者身上。这就是为什么你在 `query.rs` 里**找不到 `Query::limit` 的定义**——它来自 `:457` 那个泛型 impl（详见 §4）。

---

## 2. `Select`：查询能选什么列

最简单的热身，先看列投影枚举 `Select`（`query.rs:38`）：

```rust
pub enum Select {
    /// Select all columns
    All,                              // ① query.rs:42 —— 选全部列
    /// Select the provided columns
    Columns(Vec<String>),            // ② query.rs:44 —— 指定列名清单
    /// Advanced selection which allows for dynamic column calculations
    Dynamic(Vec<(String, String)>), // ③ query.rs:51 —— (输出列名, SQL 表达式) 元组
}
```

三个变体的含义：

| 变体 | 行号 | 含义 | 等价 SQL |
|---|---|---|---|
| `All` | :42 | 选所有列。**注释明确警告**：永远比按需选慢（`query.rs:41`） | `SELECT *` |
| `Columns(vec)` | :44 | 只选这几列 | `SELECT a, b` |
| `Dynamic(vec)` | :51 | 元组 `(输出名, SQL 表达式)`，可做动态计算列 | `SELECT a+b AS combined, c` |

`Dynamic` 是最有意思的：元组第一项是输出列名，第二项是 SQL 表达式。文档注释（`query.rs:402-403`）给的例子是 `SELECT a + b AS combined, c` 对应 `&[("combined", "a + b"), ("c", "c")]`。

两个便捷构造器把"传 `&str` 还是 `String`"的烦恼抹平了：

```rust
pub fn columns(columns: &[impl AsRef<str>]) -> Self { ... }      // query.rs:59
pub fn dynamic(columns: &[(impl AsRef<str>, impl AsRef<str>)]) -> Self { ... }  // query.rs:66
```

> **Rust 知识点：`impl AsRef<str>`**。`AsRef<str>` 是"能借出一个 `&str` 的任意类型"——`String`、`&str`、`&String` 都满足。`columns(&["a", "b"])` 和 `columns(&[s1, s2])`（`s1`/`s2` 是 `String`）都能编译。这是 Rust 处理"我不在乎你给我字符串的具体所有权形式"的惯用法，类比 TS 里接受 `string | String对象`。

`Select::All` 是默认值（`QueryRequest` 的 `Default` 里 `select: Select::All`，`query.rs:657`）。最终这个枚举会在后厨被 `match` 成三条 lance scanner 调用（`table.rs:2160-2168`，§7 详述）。

---

## 3. `QueryBase`：一切查询共有的配置

`QueryBase`（`query.rs:334`）是"通用调料台"，定义了 11 个**所有查询都能用**的链式配置方法。先看它的签名特征——**全部是 self-consuming（消费自身）**：

```rust
pub trait QueryBase {
    fn limit(self, limit: usize) -> Self;            // ① query.rs:342 —— 注意是 self 不是 &self
    fn offset(self, offset: usize) -> Self;          //   query.rs:348
    fn only_if(self, filter: impl AsRef<str>) -> Self;  // query.rs:362
    fn full_text_search(self, query: FullTextSearchQuery) -> Self;  // query.rs:383
    fn select(self, selection: Select) -> Self;      //   query.rs:407
    fn fast_search(self) -> Self;                    //   query.rs:417
    fn postfilter(self) -> Self;                     //   query.rs:437
    fn with_row_id(self) -> Self;                    //   query.rs:440
    fn rerank(self, reranker: Arc<dyn Reranker>) -> Self;  // query.rs:445
    fn norm(self, norm: NormalizeMethod) -> Self;    //   query.rs:450
}
```

> **Rust 陷阱：self-consuming builder（`fn limit(self) -> Self`）**。这个方法拿走 `self` 的**所有权**（不是借用 `&self`），改完字段再把整个 `self` **还回去**。这就实现了 `q.limit(10).offset(5).select(...)` 的链式调用：每一步"消费上一个、产生下一个"。类比 JS 里 `return this`，但 Rust 是**真的移动所有权**——所以一条链用完，原变量就被"移动"走了，不能再用（除非 `Clone`）。这也是为什么文档注释说"this query object can be reused"（`query.rs:677`）时，强调的是先 `clone` 再用。

逐方法表（注意：这里列的是 trait 上的**声明**，方法体在 §4 的 blanket impl 里）：

| 方法 | 行号 | 配什么 | 默认行为 / 注意点 |
|---|---|---|---|
| `limit(n)` | :342 | 最多返回几行 | 普通扫描**无 limit**（返回全表）；向量 / FTS 搜索默认 `DEFAULT_TOP_K`(=10, `query.rs:34`) |
| `offset(n)` | :348 | 跳过前 n 行 | 默认从第一行开始 |
| `only_if(sql)` | :362 | SQL 过滤字符串，如 `"x > 10"` | 只构造 `QueryFilter::Sql` 变体（见 §8）。建标量索引可加速 |
| `full_text_search(q)` | :383 | 全文检索（BM25 打分） | 需表上有 FTS 索引。**若当前 limit 为空会补 `DEFAULT_TOP_K`**（`query.rs:474-476`） |
| `select(sel)` | :407 | 列投影（§2 的 `Select`） | 默认 `All`。**最佳实践：永远只选你要的列** |
| `fast_search()` | :417 | 只查**已建索引**的数据 | 默认 false。弱一致性快路径——新写入但还没并进索引的数据查不到 |
| `postfilter()` | :437 | 过滤改到向量搜索**之后** | **方法名与字段名相反**（见下方陷阱） |
| `with_row_id()` | :440 | 返回 `_rowid` 元列 | 默认 false。reranking 时常需要 |
| `rerank(r)` | :445 | 指定 reranker | **仅 Hybrid 搜索支持**（`query.rs:444`） |
| `norm(m)` | :450 | 分数归一化方法（rank / score） | 默认 None。Hybrid 用 |

> **Rust 陷阱（也是读代码的大坑）：`postfilter()` 设的是 `prefilter = false`**。`QueryBase` trait 里**根本没有 `prefilter()` 方法**，只有 `postfilter()`。而它在 blanket impl 里的实现是 `self.mut_query().prefilter = false`（`query.rs:491-494`）。字段名（`prefilter`）和方法名（`postfilter`）语义相反——读源码时务必当心。默认 `prefilter = true`（先过滤再向量搜索，`query.rs:660`），调 `postfilter()` 把它翻成 false（先向量搜索再过滤）。

> **概念区分：`fast_search` vs 后面会讲的 `use_index`**。两者正交，极易混：
> - `fast_search`（`QueryBase`，:417）= 只查**已索引的数据**，跳过尚未并入索引的增量数据。换"快"与"弱一致"。
> - `use_index`（`VectorQuery` 专属，§6）= 是否用**向量索引**做 ANN 近似搜索（关掉就暴力 flat search）。
>
> 一个管"查不查增量数据"，一个管"用不用向量索引"，互不相干。

### 3.1 一个细节：`full_text_search` 的自动补 limit

注意 `full_text_search` 不是裸赋值，它有个小逻辑（`query.rs:473-479`）：

```rust
fn full_text_search(mut self, query: FullTextSearchQuery) -> Self {
    if self.mut_query().limit.is_none() {            // ① 如果用户没设 limit
        self.mut_query().limit = Some(DEFAULT_TOP_K); //   补默认 10，防止全表 BM25
    }
    self.mut_query().full_text_search = Some(query);
    self
}
```

为什么 FTS 要强制补 limit？因为全文检索若不限量，会对全表算 BM25 分数——代价巨大。所以"没说就给 10"是一个保护性默认。

---

## 4. `HasQuery` 与 blanket impl：为什么两种 Builder 能共享配置

这是整个模块**最精妙的设计**，也是非 Rust 读者最容易卡住的地方。

### 4.1 `HasQuery`：只有一个方法的「桥」

```rust
pub trait HasQuery {                                  // query.rs:453
    fn mut_query(&mut self) -> &mut QueryRequest;     // query.rs:454
}
```

它只要求实现者干一件事：**交出内部那个 `QueryRequest` 的可变引用**。`Query` 和 `VectorQuery` 各自实现它：

```rust
impl HasQuery for Query {                             // query.rs:755
    fn mut_query(&mut self) -> &mut QueryRequest {
        &mut self.request                             // ① Query 的 request 就是 QueryRequest
    }
}

impl HasQuery for VectorQuery {                       // query.rs:1134
    fn mut_query(&mut self) -> &mut QueryRequest {
        &mut self.request.base                        // ② VectorQuery 要钻进 .base（内嵌的 QueryRequest）
    }
}
```

注意第 ② 行的关键差异：`VectorQuery` 的 `mut_query` 返回的是 `&mut self.request.base`——也就是向量订单里**内嵌的那个普通订单**。这有个重要后果：**`QueryBase` 的 11 个方法只能改 `base` 里的通用字段**（`limit`/`filter`/...），改不到 `nprobes` 等向量专属字段。向量字段只能用 `VectorQuery` 自己的固有方法改（§6）。

### 4.2 blanket impl：一次实现，全员享用

```rust
impl<T: HasQuery> QueryBase for T {                   // query.rs:457
    fn limit(mut self, limit: usize) -> Self {
        self.mut_query().limit = Some(limit);         // ① 通过桥拿到 QueryRequest，改字段
        self                                          // ② 还回去
    }
    fn offset(mut self, offset: usize) -> Self { ... }
    // ……其余 9 个方法同款套路
}
```

这一行 `impl<T: HasQuery> QueryBase for T` 是关键。它说："**为所有实现了 `HasQuery` 的类型 `T`，一次性实现 `QueryBase`**。" 于是 `Query` 和 `VectorQuery`（都实现了 `HasQuery`）**自动获得**全部 11 个通用配置方法，无需各写一遍。

> **Rust 陷阱：blanket impl（覆盖式实现）**。`impl<T: Trait> OtherTrait for T` 是"给所有满足约束的类型批量实现"。类比 TS 里给一个接口的所有实现者批量加 mixin 方法，或 Python 里给一个 ABC 的所有子类统一注入方法。**这就是为什么 `Query::limit` 在 `query.rs` 里搜不到独立定义**——它来自这个泛型 impl，编译期对每个 `T` 单态化（monomorphize）出一份。

每个方法体都是同款三步：`self.mut_query()` 拿引用 → 改字段 → `return self`。**零额外分配**——所有权在链条里流动，没有堆分配、没有拷贝。

> **给贡献者的操作清单**：要加一个通用配置项（比如 `with_fragment_ids()`），完整路径是：
> 1. `QueryRequest`（:612）加一个字段 + `Default`（:650）给默认值；
> 2. `QueryBase`（:334）加方法签名；
> 3. blanket impl（:457）加方法体（一行赋值）；
> 4. 如果该字段要真正生效，还得去 `table.rs` 把它写进 lance scanner（§7 末尾强调，**漏了这步参数就是摆设**）。

---

## 5. `Query` → `VectorQuery`：`nearest_to` 与 `IntoQueryVector`

### 5.1 `Query` 的结构与升级方法

```rust
pub struct Query {                                    // query.rs:680
    parent: Arc<dyn BaseTable>,                       // ① 后厨电话（同 09 篇 Table.inner）
    request: QueryRequest,                            // ② 订单小票
}
```

两个字段，和 `09` 篇里 `NativeTable` 一样"薄"。`Query` 的固有方法：

| 方法 | 行号 | 干什么 |
|---|---|---|
| `new(parent)` | :686 | 构造（`request` 取 `Default`）。`pub(crate)`——用户从 `table.query()` 拿，不直接 new |
| `into_vector()` | :696 | 内部辅助：转成 `VectorQuery`（查询向量为空） |
| `nearest_to(vector)` | :734 | **plain → vector 单向升级**（下详） |
| `into_request()` | :746 | 拿出 `QueryRequest`（消费 self） |
| `current_request()` | :750 | 借出 `&QueryRequest`（检视用） |

### 5.2 `nearest_to`：单向门的内部

```rust
pub fn nearest_to(self, vector: impl IntoQueryVector) -> Result<VectorQuery> {  // query.rs:734
    let mut vector_query = self.into_vector();                          // ① 升级成 VectorQuery
    let query_vector = vector.to_query_vector(&DataType::Float32, "default")?;  // ② 翻译用户输入→Arrow
    vector_query.request.query_vector.push(query_vector);              // ③ push 进向量清单
    if vector_query.request.base.limit.is_none() {                    // ④ 没设 limit
        vector_query.request.base.limit = Some(DEFAULT_TOP_K);        //   补默认 10
    }
    Ok(vector_query)
}
```

四步走：① `into_vector()`（:696）调 `VectorQuery::new`（:858），把 `Query.parent` 搬走、`request` 经 `VectorQueryRequest::from_plain_query`（:834）包成 `base`；② 把用户传的 `Vec`/slice 翻译成 Arrow Array；③ push 进 `query_vector`；④ 向量搜索必须有 limit，没设就补 10。

> **比喻：升级套餐（单向门）**。普通点单 `Query` 一旦调 `nearest_to`，就升级成"向量搜索套餐" `VectorQuery`，解锁 `nprobes`/`refine_factor` 等高级旋钮。虽然有 `into_plain()`（:873）能降回去，但**会丢掉所有向量字段**。理解这个单向性，就能解释为什么 `nprobes` 只在 `VectorQuery` 上、不在 `Query` 上——因为只有"已经升级的菜单"才需要它。

> **注意 ②：硬编码 `Float32` 和 `"default"`**。`nearest_to` 写死了用 `DataType::Float32` 和 label `"default"` 调翻译（`query.rs:736`）。这是"没注册 embedding 模型时"的默认假设——用户直接传浮点向量。`add_query_vector`（:903）也同样硬编码。如果将来支持 sentence-transformer（输入是字符串），这里的 `Float32` 假设就要改。

### 5.3 `IntoQueryVector`：哪些类型能当查询向量

`IntoQueryVector`（`query.rs:87`）是"翻译官"trait，唯一方法：

```rust
fn to_query_vector(
    self,
    data_type: &DataType,           // ① 后厨期望的目标类型（通常 Float32）
    embedding_model_label: &str,    // ② 仅用于报错信息
) -> Result<Arc<dyn Array>>;        // ③ 译成 Arrow 数组
```

它存在的意义（注释 `query.rs:78-86`）：让**不懂 Arrow 的用户**直接传 `Vec<f32>` 这样的原生类型，而不必手搓 Arrow 数组；同时也是接入其他库（如 polars）的扩展点。

实现清单（全文件最密集的 trait 实现群）：

| 实现 for | 行号 | 行为 |
|---|---|---|
| `Arc<dyn Array>` | :122 | 已经是 Arrow 数组。类型不符时**对 Float16/32/64 尝试 `arrow_cast::cast`**（:129-142），其余报错 |
| `&dyn Array` | :157 | **严格不转**：类型不符直接报错（:163-169），相符则 `to_data` + `make_array` |
| `&[f16]` | :177 | 按目标 `DataType` 分三支转 Float16/32/64 |
| `&[f32]` | :207 | 同上，三支 |
| `&[f64]` | :237 | 同上，三支 |
| `&[f16; N]` / `&[f32; N]` / `&[f64; N]` | :267/:278/:289 | 定长数组，`as_slice()` 委托给上面三个 slice 实现 |
| `Vec<f16>` / `Vec<f32>` / `Vec<f64>` | :300/:311/:322 | 同样 `as_slice()` 委托 |

> **设计观察：几乎所有实现都委托到 `&[fXX]` 三个核心实现**。`Vec<f32>`、`&[f32; 128]` 都只是 `self.as_slice().to_query_vector(...)`（如 :306、:284）。三个 slice 实现（:177/:207/:237）才是真正做 `f16↔f32↔f64` 转换的地方。

> **`Arc<dyn Array>` 与 `&dyn Array` 的不对称**：前者类型不符会**尝试 cast**（宽容），后者**严格报错**（不转）。差异在 :129-142 vs :163-169。为什么？`Arc<dyn Array>` 持有所有权能安全地产出新数组；`&dyn Array` 只是借用，cast 会涉及新分配的所有权问题，于是干脆不转。

> **给贡献者**：想支持新输入类型（polars `Series`、`float8`），只需为它 `impl IntoQueryVector`，并尽量委托到现有的 slice 实现。这是开放给生态的扩展点。

> **Rust 知识点：`const N: usize` 泛型（const generics）**。`impl<const N: usize> IntoQueryVector for &[f16; N]`（:267）是"对所有定长数组长度统一实现"。`&[f32; 128]` 和 `&[f32; 768]` 共用这一份代码，编译期按 `N` 单态化。类比 C++ 模板按数组大小特化。

---

## 6. `VectorQuery` 的向量专属旋钮

`VectorQuery`（`query.rs:852`）结构同 `Query`，只是 `request` 换成 `VectorQueryRequest`：

```rust
pub struct VectorQuery {                              // query.rs:852
    parent: Arc<dyn BaseTable>,
    request: VectorQueryRequest,                      // ← 内嵌 base + 9 个向量字段
}
```

它的**固有方法**（inherent impl，不是 trait 方法）就是向量专属旋钮。逐方法表：

| 方法 | 行号 | 调什么 | 默认值 | 仅对哪种索引有效 |
|---|---|---|---|---|
| `column(name)` | :887 | 指定向量列（多向量列时必填） | 自动探测 | — |
| `add_query_vector(v)` | :902 | 加第二个查询向量，输出多 `query_index` 列 | 单向量 | — |
| `nprobes(n)` | :928 | IVF 探查的分区数 | **20**（:822） | **仅 IVF PQ**（:910） |
| `distance_range(lo, hi)` | :935 | 距离范围 `[lower, upper)` 过滤 | None | — |
| `ef(n)` | :948 | HNSW refine 候选数 | None（即 1.5×limit，:806） | **仅 HNSW**（:943） |
| `refine_factor(n)` | :980 | 取 `limit × n` 个候选再用真实距离重排 | None | **仅 IVF PQ**（:955） |
| `distance_type(t)` | :997 | 距离度量（L2/Cosine/Dot） | None（默认 L2，:996） | — |
| `bypass_vector_index()` | :1009 | 设 `use_index=false`，强制暴力 flat 搜索 | `use_index=true`（:828） | — |

每个旋钮的"为什么"（WHY，源码文档注释里都讲了）：

- **`nprobes`**（:908-927）：IVF 索引把向量分成若干分区（簇）。`nprobes` 控制搜索几个最近的分区。**越大召回越高、延迟越大**。默认 20 适合很多场景，但最佳值要 benchmark。
- **`refine_factor`**（:953-979）：IVF PQ 存的是**压缩（量化）**向量，比较不精确。`refine_factor=3` + `limit=10` → 先 ANN 取 30 个候选，再取它们的**完整未压缩值**算真实距离，重排后留前 10。**注意**：调用此方法（哪怕传 1）就会触发取完整值的开销；**不调用时返回的 `_distance` 是基于量化向量的近似距离**，可能与真实距离差很多。
- **`ef`**（:941-947）：HNSW 专用，refine 阶段的候选数，默认 1.5×limit。
- **`bypass_vector_index`**（:1002-1008）：跳过向量索引做暴力搜索。**用途**：得到 ground truth，用来计算召回率、给 `nprobes` 选合适的值。

`VectorQueryRequest` 的字段全表见 §8。这里看它的两个固有辅助方法：

```rust
pub fn into_plain(self) -> Query {                   // query.rs:873 —— 降级回普通查询（丢向量字段）
    Query { parent: self.parent, request: self.request.base }
}
```

> **`add_query_vector` 与多向量搜索**：调用多次会把多个查询向量塞进 `query_vector: Vec<...>`（:798）。执行时会派发多次搜索，结果带一个额外的 `query_index` 列标明来自哪个查询向量（:899-901）。测试 `test_multiple_query_vectors`（:1555）验证了它产生 `UnionExec` 计划（:1567）并输出 `query_index` 列（:1578）。**注意**：注释说这"不比并发发多个查询更快"（:896-897）——只是便利封装。

---

## 7. `ExecutableQuery`：execute / explain_plan / analyze_plan 如何落到后厨

配完所有旋钮，喊一声 `execute()` 才真正下单。执行能力在 `ExecutableQuery`（`query.rs:546`）：

```rust
pub trait ExecutableQuery {
    fn create_plan(&self, options: QueryExecutionOptions)
        -> impl Future<Output = Result<Arc<dyn ExecutionPlan>>> + Send;   // ① :551

    fn execute(&self)                                                     // ② :559 带默认实现
        -> impl Future<Output = Result<SendableRecordBatchStream>> + Send {
        self.execute_with_options(QueryExecutionOptions::default())
    }

    fn execute_with_options(&self, options: QueryExecutionOptions)        // ③ :580
        -> impl Future<Output = Result<SendableRecordBatchStream>> + Send;

    fn explain_plan(&self, verbose: bool)                                 // ④ :585
        -> impl Future<Output = Result<String>> + Send;

    fn analyze_plan(&self)                                                // ⑤ :587 带默认实现
        -> impl Future<Output = Result<String>> + Send {
        self.analyze_plan_with_options(QueryExecutionOptions::default())
    }

    fn analyze_plan_with_options(&self, options: QueryExecutionOptions)   // ⑥ :591
        -> impl Future<Output = Result<String>> + Send;
}
```

> **Rust 陷阱：RPITIT（return-position `impl Trait` in trait）**。注意签名是 `fn execute(&self) -> impl Future<...> + Send`，**不是 `async fn`**。这是较新的 Rust 特性，等价于"返回一个匿名的 Future 类型"。非 Rust 读者可以读成"返回一个 Promise，且这个 Promise 能在任意线程 `await`（`+ Send` 保证跨线程安全）"。为什么不用 `async fn`？async fn in trait 当时对 `+ Send` 约束的表达有限制，RPITIT 显式写出 `+ Send` 更可控。`execute`(:559) 和 `analyze_plan`(:587) 是**带默认实现**的——只是转调带 options 的版本。

### 7.1 `Query` 的执行：打包成 `AnyQuery` 递给后厨

```rust
impl ExecutableQuery for Query {                      // query.rs:761
    async fn create_plan(&self, options: QueryExecutionOptions) -> Result<Arc<dyn ExecutionPlan>> {
        let req = AnyQuery::Query(self.request.clone());        // ① clone 订单，包成 AnyQuery
        self.parent.clone().create_plan(&req, options).await   // ② 委托给后厨 BaseTable
    }
    async fn execute_with_options(&self, options) -> Result<SendableRecordBatchStream> {
        let query = AnyQuery::Query(self.request.clone());     // query.rs:771
        Ok(SendableRecordBatchStream::from(
            self.parent.clone().query(&query, options).await?, // ③ 委托 BaseTable::query
        ))
    }
    // explain_plan(:777) / analyze_plan_with_options(:782) 同理
}
```

**关键**：`Query` 执行时**不碰任何 lance scanner**。它做的全部事情是：① `clone` 自己的 `request`，包进 `AnyQuery::Query` 枚举（`table.rs:398`）；② 把这个统一枚举交给 `self.parent`（那根 `Arc<dyn BaseTable>` 后厨电话）。后厨（`NativeTable` 或 `RemoteTable`）才把字段写进 lance scanner 真正执行。

> **比喻：把订单递进后厨**。点菜单（`Query`）勾完单子，把它复印一份（`clone`）装进标准信封（`AnyQuery::Query`），从窗口递给后厨（`parent.query(...)`）。菜单自己一勺菜都不炒。这就是 `09` 篇"组装而非重造"在查询侧的体现——**`query.rs` 编排参数，lance 做活**。

> **Rust 知识点：`AnyQuery` enum 做多态**。`AnyQuery`（`table.rs:398`）有两个变体 `Query(QueryRequest)` / `VectorQuery(VectorQueryRequest)`。后厨用 `match` 分发，无需为普通查询和向量查询各开一个 `BaseTable` 方法。把 enum 想成"带标签的联合体 / sealed class"。

### 7.2 `VectorQuery` 的执行分叉：普通向量 vs Hybrid

`VectorQuery` 的 `execute_with_options`（`query.rs:1109`）多了一个分叉：

```rust
async fn execute_with_options(&self, options) -> Result<SendableRecordBatchStream> {
    if self.request.base.full_text_search.is_some() {          // ① 同时设了 FTS → Hybrid
        let hybrid_result = async move { self.execute_hybrid(options).await }
            .boxed()                                            // ② 必须装箱（见陷阱）
            .await?;
        return Ok(hybrid_result);
    }
    self.inner_execute_with_options(options).await             // ③ 否则走普通向量执行
}
```

而 `inner_execute_with_options`（:1088）是普通向量查询的真正执行：

```rust
async fn inner_execute_with_options(&self, options) -> Result<SendableRecordBatchStream> {
    let plan = self.create_plan(options.clone()).await?;       // ① 经 BaseTable 造 DataFusion 计划
    let inner = execute_plan(plan, Default::default())?;       // ② lance_datafusion 执行计划
    let inner = if let Some(timeout) = options.timeout {       // ③ 可选超时包裹
        TimeoutStream::new_boxed(inner, timeout)
    } else { inner };
    Ok(DatasetRecordBatchStream::new(inner).into())            // ④ 包成结果流
}
```

> **Rust 陷阱：`.boxed()` 装箱 Future**（:1115）。`execute_hybrid` 内部又会调用查询 Future，类型大小在编译期无法确定（潜在无限递归类型）。`.boxed()` 把 Future 装进堆指针（`Box<dyn Future>`），打破"类型无限大"的编译错误。类比把闭包放进堆指针。

### 7.3 Hybrid 搜索：lancedb 层手工编排（不是一条 lance 计划）

`execute_hybrid`（`query.rs:1014`）是本模块**唯一一处真正"做事"的逻辑**——但它做的不是扫描，而是**编排两条独立查询 + Rust 侧融合**。完整控制流（带行号调用栈）：

```
VectorQuery::execute_hybrid  (query.rs:1014)
├─ ① 从 base 克隆出一个纯 FTS Query，并 with_row_id()       :1019-1021
├─ ② clone 自身为 vector_query.with_row_id()                :1023
│     并清掉 full_text_search 防止递归回 hybrid 分支         :1025  ★关键
├─ ③ try_join! 并发执行 FTS 与 向量 两条查询                 :1026-1029
├─ ④ try_join! 各自 try_collect 成 Vec<RecordBatch>         :1031-1034
├─ ⑤ hybrid::query_schemas 取 schema                        :1038
│     concat_batches 各自合并                                :1041-1042
├─ ⑥ 若 norm == Rank：hybrid::rank 先转秩                    :1044-1047
├─ ⑦ hybrid::normalize_scores 归一化两边分数                 :1049-1050
├─ ⑧ reranker = base.reranker 或默认 RRFReranker            :1052-1057  ★默认 RRF
├─ ⑨ reranker.rerank_hybrid(fts查询串, vec结果, fts结果)     :1068-1070
├─ ⑩ check_reranker_result 校验                             :1072
│     slice 到 limit                                        :1074-1077
│     若未要求 row_id 则 drop _rowid 列                      :1079-1081
└─ ⑪ 单批包成 stream 返回                                    :1083-1085
```

```rust
let mut vector_query = self.clone().with_row_id();
vector_query.request.base.full_text_search = None;       // ★ query.rs:1025 —— 防递归
let (fts_results, vec_results) = try_join!(              // query.rs:1026
    fts_query.execute_with_options(options.clone()),
    vector_query.inner_execute_with_options(options)     // 注意：走 inner，绕开 hybrid 分叉
)?;
```

> **比喻：双厨房并行 + 调度长融合**。全文检索厨房和向量搜索厨房**同时开火**（`try_join!` 并发，:1026），各自出菜后由调度长（`Reranker`，默认 `RRFReranker`，:1057）按评分重新摆盘、截到客人要的份数。

> **Hybrid 触发条件（易错点）**：只有在 **`VectorQuery` 上**设了 `full_text_search` 才走 hybrid（`query.rs:1113`）。在普通 `Query` 上设 `full_text_search` 不会触发 hybrid——那走 `BaseTable` 的**纯 FTS 路径**。换句话说，hybrid = "既 `nearest_to` 又 `full_text_search`"。

> **Rust 陷阱：`try_join!` 宏**（:1026）。并发跑多个 Future，**任一出错则整体出错并取消其余**。类比 `Promise.all`，但带早退错误传播。这里用了两层 `try_join!`：先并发执行两条查询拿到流（:1026），再并发把两个流 collect 成 Vec（:1031）。

> **给贡献者**：想改融合算法（默认 RRF），就实现 `Reranker` trait 然后用 `rerank()` 传入，或改 `:1057` 的默认值。整个 hybrid 是纯 lancedb 层逻辑，**不下沉到 lance**——这是"编排"而非"重造"的极端案例。

### 7.4 explain / analyze 也只是转发

`Query::explain_plan`（:777）/ `VectorQuery::explain_plan`（:1123）都只是打包 `AnyQuery` 后 `self.parent.explain_plan(...)` 转发。真正的格式化在 `BaseTable::explain_plan`（`table.rs:430`）：`create_plan` → `DisplayableExecutionPlan::new`（:432）→ `indent(verbose)` 渲染成文本。**链路细节见 `06`**。

---

## 8. `QueryRequest` / `VectorQueryRequest` 字段全表

两种"订单小票"是纯数据结构，值得逐字段核对。

### 8.1 `QueryRequest`（`query.rs:612`）—— 10 个 `pub` 字段

| 字段 | 行号 | 类型 | Default | 含义 |
|---|---|---|---|---|
| `limit` | :614 | `Option<usize>` | `None`(:653) | 最多返回行数 |
| `offset` | :617 | `Option<usize>` | `None`(:654) | 跳过前 n 行 |
| `filter` | :620 | `Option<QueryFilter>` | `None`(:655) | 过滤器（SQL/Substrait/Datafusion） |
| `full_text_search` | :623 | `Option<FullTextSearchQuery>` | `None`(:656) | 全文检索 |
| `select` | :626 | `Select` | `All`(:657) | 列投影 |
| `fast_search` | :632 | `bool` | `false`(:658) | 只查已索引数据 |
| `with_row_id` | :637 | `bool` | `false`(:659) | 返回 `_rowid` |
| `prefilter` | :640 | `bool` | **`true`**(:660) | true=先过滤；`postfilter()` 翻成 false |
| `reranker` | :644 | `Option<Arc<dyn Reranker>>` | `None`(:661) | Hybrid 用 |
| `norm` | :647 | `Option<NormalizeMethod>` | `None`(:662) | Hybrid 分数归一化 |

`QueryFilter`（`query.rs:599`）三变体：`Sql(String)`(:601) / `Substrait(Arc<[u8]>)`(:603) / `Datafusion(Expr)`(:605)。`only_if()` 只构造 `Sql` 变体（:469）。

### 8.2 `VectorQueryRequest`（`query.rs:790`）—— `base` + 9 个向量字段

| 字段 | 行号 | 类型 | Default | 含义 |
|---|---|---|---|---|
| `base` | :792 | `QueryRequest` | `Default`(:819) | **内嵌普通订单**（继承全部 §8.1 字段） |
| `column` | :796 | `Option<String>` | `None`(:820) | 向量列名（多列时必填） |
| `query_vector` | :798 | `Vec<Arc<dyn Array>>` | 空 Vec(:821) | 查询向量（**支持多个**） |
| `nprobes` | :800 | `usize` | **20**(:822) | IVF 探查分区数 |
| `lower_bound` | :802 | `Option<f32>` | `None`(:823) | 距离下界（含） |
| `upper_bound` | :804 | `Option<f32>` | `None`(:824) | 距离上界（不含） |
| `ef` | :807 | `Option<usize>` | `None`(:825) | HNSW refine 候选数 |
| `refine_factor` | :809 | `Option<u32>` | `None`(:826) | IVF PQ 精排倍数 |
| `distance_type` | :811 | `Option<DistanceType>` | `None`(:827) | 距离度量 |
| `use_index` | :813 | `bool` | **`true`**(:828) | false=暴力 flat 搜索 |

> **核对要点**：`VectorQueryRequest` 的 `Default`（:816）里只有 `nprobes=20` 和 `use_index=true` 是非平凡默认，其余全是 `None` / 空 Vec。`from_plain_query`（:834）用 `base: query, ..Default::default()` 把一个 `QueryRequest` 塞进 `base`、其余取默认——这正是 `nearest_to` 升级的内部机制。

> **Rust 知识点：`..Default::default()`（结构体更新语法）**。`from_plain_query`（:834）写 `Self { base: query, ..Default::default() }`，意思是"`base` 用我给的，**其余字段全从 `Default` 拷过来**"。类比 JS 的 `{...defaults, base: query}`，但顺序相反（Rust 里显式字段优先）。

### 8.3 `QueryExecutionOptions`（`query.rs:515`）—— 执行选项

```rust
#[non_exhaustive]                                     // query.rs:513
pub struct QueryExecutionOptions {
    pub max_batch_length: u32,                        // :528 单个 RecordBatch 最大行数，默认 1024(:536)
    pub timeout: Option<Duration>,                    // :530 超时
}
```

> **Rust 陷阱：`#[non_exhaustive]`**（:513）。标记此结构体**未来可能加字段**，外部 crate 不能用 `QueryExecutionOptions { max_batch_length, timeout }` 全字段构造，必须 `..Default::default()`。这是保护 API 演进的惯用法——加字段不算 breaking change。

---

## 9. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `fn limit(self) -> Self` | self-consuming builder：消费自身再还回，链式调用每步移动所有权 | `QueryBase` :342 |
| `impl<T: HasQuery> QueryBase for T` | blanket impl：给所有满足约束的类型批量实现 | :457 |
| `fn execute(&self) -> impl Future<...> + Send` | RPITIT：trait 返回匿名 Future，非 `async fn`，`+Send` 保跨线程 | :559 |
| `Arc<dyn Array>` / `Arc<dyn BaseTable>` | 引用计数 + 动态分发；`Vec<Arc<dyn Array>>` = 一组共享的任意 Arrow 数组 | :798/:681 |
| `enum` + `match` 做多态 | `Select`/`QueryFilter`/`AnyQuery` 用带标签联合体分发，无继承 | :38/:599/table.rs:398 |
| `impl<const N: usize>` | const generics：对所有定长数组长度统一实现 | :267 |
| `impl AsRef<str>` | "能借出 `&str` 的任意类型"，抹平 `String`/`&str` | `Select::columns` :59 |
| `..Default::default()` | 结构体更新语法：指定部分字段，其余从 Default 拷 | `from_plain_query` :837 |
| `#[non_exhaustive]` | 结构体未来可加字段，外部必须用 `..Default::default()` 构造 | :513 |
| `try_join!` | 并发多个 Future，任一出错整体早退 | `execute_hybrid` :1026 |
| `.boxed()` | 装箱 Future 到堆，打破无限递归类型 | :1115 |
| `Option<T>` + `unwrap_or(...)` | 取值或回退默认 | `limit.unwrap_or(DEFAULT_TOP_K)` :1074 |

---

## 10. 动手验证（建议亲手做一遍）

1. **数清三层 trait 的边界**：打开 `query.rs`，确认 `QueryBase`（:334）有 11 个方法签名、但**没有任何方法体**——方法体全在 `:457` 的 blanket impl 里。问自己：为什么 `Query::limit` 在文件里搜不到独立定义？（答案：blanket impl 替它生成。）
2. **抓 `postfilter` 的反直觉**：跳到 `:491`，确认 `postfilter()` 设的是 `prefilter = false`。再搜整个 trait，确认**没有 `prefilter()` 方法**。这是读代码的经典坑。
3. **跟一次升级**：从 `Query::nearest_to`（:734）手动追到 `into_vector()`（:696）→ `VectorQuery::new`（:858）→ `VectorQueryRequest::from_plain_query`（:834）。看清"普通订单怎么被包进向量订单的 `base`"。
4. **验证执行只是转发**：在 `query.rs` 里搜 `self.parent.clone()` 和 `AnyQuery::`，统计 `Query`/`VectorQuery` 的执行方法（:763/:771/:1105/...），确认它们全是"clone request → 包 AnyQuery → 递给 parent"。`query.rs` 里**没有一行 lance scanner 代码**。
5. **读懂 hybrid 的并发**：跳到 `execute_hybrid`（:1014），找到 `:1025` 那行 `full_text_search = None`，理解它为什么是防递归的关键——少了它，向量子查询会再次命中 hybrid 分叉（:1113）造成无限递归。再看两层 `try_join!`（:1026/:1031）。
6.（可选）**跑测试看计划**：`test_create_execute_plan`（:1448）断言无索引时计划含 `KNNFlatSearch`（:1458，flat 暴力搜索）；`test_fast_search_plan`（:1495）验证 `fast_search` 后计划不含 `Take`；`test_multiple_query_vectors`（:1555）验证多向量产生 `UnionExec`（:1567）+ `query_index` 列。`cargo test -p lancedb query::` 跑一遍。

---

## 11. 小结 & 下一篇

- **`query.rs` 是点菜单，不是厨房**：1735 行里没有一行扫描代码。它收集参数（`QueryRequest`/`VectorQueryRequest`）、提供链式 API，执行时把订单打包成 `AnyQuery` 递给 `Arc<dyn BaseTable>` 后厨。这是"组装而非重造"在查询侧的字面证据。
- **两种订单 + 两种菜单的双层结构**：`Request`（纯数据，可序列化 / 测试 / 检视）与 `Query`（数据 + 后厨指针，能执行）分离；`VectorQueryRequest` 内嵌 `QueryRequest` 复用全部通用字段。
- **三层 trait 零重复共享配置**：`QueryBase`（通用旋钮）通过 blanket impl（:457）自动套到任何 `HasQuery` 实现者（`Query`/`VectorQuery`）上；向量专属旋钮（`nprobes`/`refine_factor`...）放 `VectorQuery` 的固有方法。
- **`nearest_to` 是单向门**：plain → vector 升级解锁向量旋钮，理解这点就理解了为什么 `nprobes` 只在 `VectorQuery` 上。
- **Hybrid 是纯 lancedb 层手工编排**：`execute_hybrid` 并发跑 FTS + 向量两条查询，Rust 侧用 `Reranker`（默认 RRF）融合——不下沉到 lance。

**下一篇会继续往后厨钻**：本篇讲到 `self.parent.create_plan(...)` / `self.parent.query(...)` 就收尾，把订单递进了 `BaseTable`。`06 查询链路`（链路篇）从这里接手，讲 `AnyQuery` 在 `NativeTable` 里如何被 `match` 拆开、字段如何逐个写进 lance scanner（`table.rs:2149-2178` 那段旋钮映射）、最终如何出结果流。**点菜单的故事到此为止，下厨的故事在 `06`。**
