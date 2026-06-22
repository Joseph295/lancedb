# 07 · Connection 与 Builder 体系：餐厅经理与点菜单王国

> **本篇解剖的源码**：`rust/lancedb/src/connection.rs`（约 1330 行，含测试）
> **覆盖区间**：第 1–912 行 —— `Connection` 门面、`ConnectBuilder`/`connect`、`CreateTableBuilder<HAS_DATA>` 双套方法、`OpenTableBuilder`/`TableNamesBuilder`、`CatalogConnectBuilder`/`connect_catalog`
> **配角源码**：`database.rs:33-167`（`Database`/`DatabaseOptions` 契约与各 Request 类型）、`remote/db.rs:61-122`（远程选项的 key 与序列化）、`arrow.rs:119-155`（`IntoArrow`/`IntoArrowStream`）
> **前置阅读**：`01 Rust 垫脚石`（trait / `dyn` / `Arc` / async）、`02 三大基石`。Builder「消费自身」的套路与 `03 连接建立链路`、`04 写入链路` 是同一条路的两个视角——本篇讲**结构**，那两篇讲**链路**。
> **与哪些篇互补**：`09/10 Table 解剖` 讲 `Table`/`BaseTable` 那一侧的门面；本篇讲它的「上游」——你拿到 `Table` 之前，得先有 `Connection`，而 `Connection` 全靠 Builder 喂养。两边共用同一套「门面 + `Arc<dyn ...>` + Builder」骨架。
>
> 本篇所有论断都带真实行号，形如 `connection.rs:455`，可点击跳转。

---

## 0. 这一篇要回答的三个问题

如果你只带走三句话，就是这三句：

1. **`Connection` 是「餐厅经理」**——它手里只攥着一部「后厨电话」`Arc<dyn Database>`，所有请求都转手发给后厨，自己不干活。
2. **进餐厅的每一道门都是 Builder**——建表、开表、列表名、连接，全都先递给你一张「点菜单」，你勾完选项喊 `.execute().await` 才下单。
3. **「有数据建表」和「无数据建表」是两种编译期不同的类型**——`CreateTableBuilder<true>` 和 `CreateTableBuilder<false>` 靠一个**编译期 bool**（const generic）区分，用错方法**编译不过**。这是本篇最高级、也最优雅的一招。

把这三句话画成一张图，就是整个 Builder 王国的骨架：

```
   你的代码
      │  connect(uri).api_key("…").execute().await
      ▼
 ┌──────────────────────────┐
 │   connect(uri)            │  connection.rs:867   ← 工厂函数，返回点菜单
 │   → ConnectBuilder        │  connection.rs:633
 └─────────┬────────────────┘
           │  .execute().await   按 uri 路由
           ▼
 ┌──────────────────────────┐
 │   Connection (餐厅经理)    │  connection.rs:455
 │   uri / internal:Arc<dyn Database> / embedding_registry │
 └─────────┬────────────────┘
           │  每个入口方法都「现造一张点菜单」
   ┌───────┼─────────┬──────────────┬─────────────┐
   ▼       ▼         ▼              ▼             ▼
create_table  create_empty   open_table   table_names   drop_table
   │ →CreateTableBuilder<true> <false>  OpenTableBuilder  TableNamesBuilder  (直接 async)
   │  :493        :531         :551          :483          :576
   ▼  .execute().await
 ┌──────────────────────────┐
 │  Database (trait/后厨合同) │  database.rs:151
 │  ListingDatabase│RemoteDatabase  ← 两家后厨，签同一份合同
 └──────────────────────────┘
```

> **比喻总钥匙**：`Connection` 是**餐厅经理**，手里那部 `internal: Arc<dyn Database>` 是直通**后厨**（数据库实现）的电话。每个 Builder 是一张**点菜单**，`.execute()` 是把单子塞进后厨窗口。`CreateTableBuilder<true/false>` 是**两本不同的点菜单**：一本带食材（数据），一本只写菜名（空 schema）；服务员发错本，经理（编译器）当场打回。`CatalogConnectBuilder` 则是**连锁总部前台**——它管的是「数据库」这一级，比单个餐厅高一层。

---

## 1. 总览：`Connection` 是「前台」，一切入口都是 Builder

### 1.1 `Connection` 的三个字段

`connection.rs:454-459`：

```rust
#[derive(Clone)]                              // ① 可廉价克隆
pub struct Connection {
    uri: String,                              // ② 连接字符串，纯展示/回溯用
    internal: Arc<dyn Database>,              // ③ ★核心：后厨电话，真正干活的人
    embedding_registry: Arc<dyn EmbeddingRegistry>,  // ④ embedding 函数登记处
}
```

和 `Table` 一样，`Connection` **薄得惊人**——只有 3 个字段，真正的能力全在 `internal` 这个 `Arc<dyn Database>` 后面（`connection.rs:457`）。`Connection` 自己只是个转发台。

| 字段 | 行号 | 作用 | 为什么是这个类型 |
|---|---|---|---|
| `uri` | `:456` | 连接 URI 字符串 | 纯 `String`，只用于 `uri()` 回显（`:469`）和测试断言（`:961`） |
| `internal` | `:457` | 后厨：实际的数据库实现 | `Arc<dyn Database>` —— 运行时多态，本地（`ListingDatabase`）/云端（`RemoteDatabase`）都塞这里 |
| `embedding_registry` | `:458` | embedding 函数登记处 | `Arc<dyn ...>`，建表/开表时复制一份给下游的 `Table`（见 §3.2） |

> **Rust 知识点：`#[derive(Clone)]` + `Arc` = 廉价克隆**。三个字段里两个是 `Arc`（原子引用计数指针），一个是 `String`。克隆一个 `Connection` 只是把两个 `Arc` 的计数 +1、复制一个短字符串，**不复制后面的数据库**。所以你会看到源码里到处 `self.internal.clone()`——那是廉价的指针复制。
>
> **比喻**：`Arc<dyn Database>` 像经理手里那部后厨电话。`clone` 是再配一部分机，但拨通的还是**同一个后厨**。配 100 部分机，后厨还是那一个。

### 1.2 `Display` 也是「甩锅」给后厨

`connection.rs:461-465`：

```rust
impl std::fmt::Display for Connection {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.internal)   // 打印请求直接转交 internal
    }
}
```

`write!(f, "{}", self.internal)` 里的 `{}` 会触发 `internal` 的 `Display`。这行能编译通过，正是因为 `Database` trait 被约束实现了 `std::fmt::Display`（`database.rs:152`：`... + std::fmt::Display + ...`）——和 `Table`/`BaseTable` 那侧（09 篇 §2.3）一模一样的套路。

### 1.3 入口方法地图：要么「现造 Builder」，要么「直接 async 转发」

`impl Connection`（`connection.rs:467-599`）的方法泾渭分明分成两类，分清它们是读懂这个文件的关键：

**(a) Builder 工厂型 —— 不是 `async`，只发「点菜单」**

| 方法 | 行号 | 返回的点菜单 |
|---|---|---|
| `table_names()` | `:483` | `TableNamesBuilder` |
| `create_table(name, data)` | `:493` | `CreateTableBuilder<true>` ← **带数据** |
| `create_table_streaming(name, data)` | `:512` | `CreateTableBuilder<true>` ← 带数据（流式） |
| `create_empty_table(name, schema)` | `:531` | `CreateTableBuilder<false>` ← **无数据** |
| `open_table(name)` | `:551` | `OpenTableBuilder` |

这五个方法**都不是 `async`、都不碰 IO**，只是 `new` 出一个 Builder 返回给你。注意它们的共同动作是把 `self.internal.clone()` 塞进 Builder——把「后厨电话」复制一份交给点菜单（如 `:498-503`）。

**(b) 立即转发型 —— `async`，调用即委托后厨**

| 方法 | 行号 | 一行委托 |
|---|---|---|
| `rename_table(old, new)` | `:562` | `self.internal.rename_table(...).await` |
| `drop_table(name)` | `:576` | `self.internal.drop_table(...).await` |
| `drop_all_tables()` | `:589` | `self.internal.drop_all_tables().await` |
| `drop_db()` | `:583` | 已废弃（`#[deprecated]`），转调 `drop_all_tables` |

加上几个纯 getter：`uri()`（`:469`）、`database()`（`:474`，把 `&Arc<dyn Database>` 直接借出去）、`embedding_registry()`（`:596`）。

> **为什么有的入口造 Builder、有的直接 async？** 凡是**有可选配置**的操作（建表能选 mode、写选项、storage 选项、embedding……）就用 Builder 把可选项摊开让你慢慢勾；凡是**没什么可配**的操作（删表只需要一个名字）就直接 `async` 转发，没必要套个空壳。这是一条贯穿全库的设计直觉。

---

## 2. `ConnectBuilder` / `connect`：URI 三形态路由

### 2.1 入口函数 `connect`

`connection.rs:867-869`：

```rust
pub fn connect(uri: &str) -> ConnectBuilder {
    ConnectBuilder::new(uri)
}
```

它**不是 `async`**——只是造一张连接点菜单 `ConnectBuilder` 返回。真正建连要等你 `.execute().await`。

### 2.2 `ConnectRequest`：连接请求的「配置载体」

`ConnectBuilder` 内部包着一个 `ConnectRequest`（`connection.rs:602-630`），所有链式方法其实都在改这个 request 的字段：

```rust
#[derive(Clone, Debug)]
pub struct ConnectRequest {
    pub uri: String,                                       // :611 三形态 URI
    #[cfg(feature = "remote")]
    pub client_config: ClientConfig,                       // :614 仅 remote 编译时存在
    pub options: HashMap<String, String>,                  // :617 ★万能配置信封
    pub read_consistency_interval: Option<std::time::Duration>,  // :629 读一致性刷新间隔
}
```

文档注释（`connection.rs:606-610`）写明了 URI 的**三种形态**，这是本节的核心：

| URI 形态 | 例子 | 路由到的后厨 |
|---|---|---|
| 本地文件系统路径 | `/path/to/database` | `ListingDatabase` |
| 云对象存储 | `s3://bucket/...`、`gs://bucket/...` | `ListingDatabase`（带 storage 选项） |
| LanceDB Cloud | `db://dbname` | `RemoteDatabase` |

> **Rust 陷阱：`#[cfg(feature = "remote")]` 条件编译字段**。`client_config` 这个字段（`:613-614`）只有在开启 `remote` feature 时才**存在于结构体里**。这是 Rust 的编译期裁剪：没开 remote 的二进制连这个字段都没有，体积更小。后面你会看到 `execute_remote` 也有两个版本——开/不开 remote 各一个（`:806` vs `:837`）。Python/JS 没有等价物（最接近的是构建时的 tree-shaking，但那是运行时之后的事）。

> **`options: HashMap<String, String>` 是「万能配置信封」**（`:617`）。无论是 API key、region、还是任意 storage 选项，最后都被塞进这一个字符串 map。下游再用 `parse_from_map` 把它解读出来（见 §2.4）。这种「先一律塞进 map、用时再解析」的设计在 listing 与 remote 两条路上通用——是本库统一配置的关键套路。

### 2.3 链式配置方法：每个都在往 `options` 里塞键值

`ConnectBuilder` 的链式方法（`connection.rs:653-803`）几乎都是同一个形状：`mut self → 改 request → 返回 Self`。挑几个有代表性的：

| 方法 | 行号 | 干什么 | remote 专属？ |
|---|---|---|---|
| `api_key(key)` | `:662` | 往 `options` 插 `remote_database_api_key` | 是（`#[cfg(feature="remote")]`） |
| `region(r)` | `:678` | 插 `remote_database_region` | 是 |
| `host_override(h)` | `:694` | 插 `remote_database_host_override` | 是 |
| `database_options(opts)` | `:709` | 调 `opts.serialize_into_map(&mut options)` 批量塞 | 否 |
| `client_config(cfg)` | `:733` | 设 `request.client_config` | 是 |
| `embedding_registry(reg)` | `:739` | 设 builder 上的 `embedding_registry` | 否 |
| `storage_option(k,v)` | `:764` | 往 `options` 插一对 | 否 |
| `storage_options(pairs)` | `:772` | 批量插 | 否 |
| `read_consistency_interval(d)` | `:797` | 设一致性刷新间隔 | 否 |

注意 `api_key`/`region`/`host_override` 三个 key 的常量定义在远程模块（`remote/db.rs:62-64`）：

```rust
pub const OPT_REMOTE_API_KEY: &str = "remote_database_api_key";       // remote/db.rs:62
pub const OPT_REMOTE_REGION: &str = "remote_database_region";         // remote/db.rs:63
pub const OPT_REMOTE_HOST_OVERRIDE: &str = "remote_database_host_override";  // remote/db.rs:64
```

`database_options` 的精髓在 `serialize_into_map`（`connection.rs:710`），它来自 `DatabaseOptions` trait（`database.rs:33-35`）：

```rust
pub trait DatabaseOptions {
    fn serialize_into_map(&self, map: &mut HashMap<String, String>);
}
```

`RemoteDatabaseOptions` 实现了它（`remote/db.rs:110-122`），把自己的字段一个个 `insert` 进那张万能 map。**`api_key()` 这种快捷方法只是 `database_options(...)` 的「单条版」——殊途同归，都往 `options` 里塞 key。**

> **Rust 陷阱：`mut self`（消费自身）的链式 Builder**。看 `region`（`:678`）：`pub fn region(mut self, region: &str) -> Self`。它接收的是 `self` 的**所有权**（不是 `&self` 借用），改完 `self.request.options` 后把 `self` 整个**还回去**。这就实现了 `connect(uri).api_key(k).region(r).execute()` 的链式。代价是：调用一次后，原变量被「移动」走，不能再用。这是 Rust Builder 的标准形态，09 篇也反复出现。

### 2.4 `execute`：URI 一字定生死

终于到 `execute`（`connection.rs:844-858`）——所有配置在这里兑现：

```rust
pub async fn execute(self) -> Result<Connection> {
    if self.request.uri.starts_with("db") {                          // ① uri 以 "db" 开头 → 云端
        self.execute_remote()
    } else {                                                          // ② 否则 → 本地/对象存储
        let internal = Arc::new(ListingDatabase::connect_with_options(&self.request).await?);
        Ok(Connection {
            internal,                                                 // ③ 包成 Arc<dyn Database>
            uri: self.request.uri,
            embedding_registry: self
                .embedding_registry
                .unwrap_or_else(|| Arc::new(MemoryRegistry::new())), // ④ 没给 registry 就造个默认空的
        })
    }
}
```

分支逻辑一句话：**`uri.starts_with("db")` 就走云端，否则一律走本地 `ListingDatabase`**（`connection.rs:846`）。本地分支调 `ListingDatabase::connect_with_options(&self.request)`（实现在 `database/listing.rs:252`），把整个 request（含那张 options map）传过去，由它自己解析 s3/gs/本地路径与 storage 选项。

云端分支 `execute_remote` 有**两个编译版本**：

```rust
#[cfg(feature = "remote")]
fn execute_remote(self) -> Result<Connection> {                      // connection.rs:806
    let options = RemoteDatabaseOptions::parse_from_map(&self.request.options)?;  // ① 从 map 解析回来
    let region = options.region.ok_or_else(|| Error::InvalidInput { … })?;        // ② region 必填
    let api_key = options.api_key.ok_or_else(|| Error::InvalidInput { … })?;      // ③ api_key 必填
    let internal = Arc::new(RemoteDatabase::try_new(&self.request.uri, &api_key, &region, …)?);  // :819
    Ok(Connection { internal, uri: self.request.uri, embedding_registry: … })
}

#[cfg(not(feature = "remote"))]
fn execute_remote(self) -> Result<Connection> {                      // connection.rs:837
    Err(Error::Runtime {                                             // 没开 remote feature → 直接报错
        message: "cannot connect to LanceDb Cloud unless the 'remote' feature is enabled".to_string(),
    })
}
```

注意这里的**对称美学**：`api_key()` 在 `:665` 把 key **塞进** map，`execute_remote` 在 `:809` 用 `parse_from_map` 把它**取出来**（`remote/db.rs:92`）。塞与取分居建连两端，中间靠那张 `HashMap` 信封传递。

> **Rust 知识点：`unwrap_or_else(|| Arc::new(MemoryRegistry::new()))`**（`:830-832` 与 `:853-855`）。`embedding_registry` 字段在 builder 上是 `Option<Arc<dyn ...>>`。`unwrap_or_else` 的意思是「有就用你给的，没有就**惰性地**造一个默认 `MemoryRegistry`」。用 `_else`（接闭包）而非 `unwrap_or`（接现成值）是因为默认值的构造有成本，要懒到真正缺省时才执行。这正是 Rust 表达「可选参数 + 默认值」的惯用法。

> **给贡献者的信号**：注意 `connect_with_options` 是 `async`、要 `.await`，但 `execute_remote` **不是 async**（`:806` 没 async，`:847` 直接调用不 await）——远程连接的建立是同步的（只是构造 client，不实际发请求）。如果你要给连接加新选项，**改动点几乎总是「加一个链式方法往 options 塞 key」+「在下游 parse 出来」**，而不是动 execute 的分支逻辑。这就是「组装而非重造」。

---

## 3. const generics typestate：`CreateTableBuilder<HAS_DATA>` 的两本点菜单

这是本篇的技术高潮。**为什么建表 Builder 要用一个编译期 `bool` 当类型参数？** 因为「有初始数据建表」和「无数据只给 schema 建表」需要**不同的方法集**，而 LanceDB 想让你**用错方法时直接编译失败**，而不是运行时报错。

### 3.1 一个结构体，一个编译期开关

`connection.rs:91-100`：

```rust
pub struct CreateTableBuilder<const HAS_DATA: bool> {     // ① const generic：类型参数是个 bool
    parent: Arc<dyn Database>,                            // 后厨电话
    embeddings: Vec<(EmbeddingDefinition, Arc<dyn EmbeddingFunction>)>,  // 待算的 embedding 列
    embedding_registry: Arc<dyn EmbeddingRegistry>,
    request: CreateTableRequest,                          // 真正的请求载体
    // This is a bit clumsy but we defer errors until `execute` is called
    // to maintain backwards compatibility
    data: CreateTableBuilderInitialData,                  // ② 预存的初始数据（可能携带错误）
}
```

> **Rust 陷阱：const generic（`const HAS_DATA: bool`）**。一般泛型参数是**类型**（`Vec<T>` 里的 `T`）；const generic 让你把一个**编译期常量值**当参数。这里 `HAS_DATA` 是个 `bool`。关键后果：**`CreateTableBuilder<true>` 和 `CreateTableBuilder<false>` 是两个彻底不同的类型**，编译器把它们当陌生人。Python/JS/TypeScript **没有任何等价物**——TS 的泛型只能传类型，传不了 `true`/`false` 这种值。最接近的类比是 C++ 模板的非类型参数（`template<bool>`）。

### 3.2 三个 `impl` 块：`true` 套、`false` 套、共享套

这是整个设计的骨架——**三个 impl 块按 `HAS_DATA` 切成三份**：

```
impl CreateTableBuilder<true>  { … }   // connection.rs:103  ← 只有「有数据」时才有的方法
impl CreateTableBuilder<false> { … }   // connection.rs:189  ← 只有「无数据」时才有的方法
impl<const HAS_DATA: bool> CreateTableBuilder<HAS_DATA> { … }  // connection.rs:214  ← 两者共享
```

**(a) `<true>` 专属（`:103-186`）：带数据的构造与执行**

| 方法 | 行号 | 作用 |
|---|---|---|
| `new<T: IntoArrow>(...)` | `:104` | 从迭代器数据造 builder，`data.into_arrow()` 先尝试转换、错误暂存 |
| `new_streaming<T: IntoArrowStream>(...)` | `:123` | 从流式数据造 builder |
| `execute() -> Result<Table>` | `:143` | 真正下单，**带上 embedding_registry** |
| `into_request() -> Result<CreateTableRequest>` | `:153` | 把暂存数据解包成最终请求（核心，见 §3.4） |

`<true>::execute`（`connection.rs:143-151`）：

```rust
pub async fn execute(self) -> Result<Table> {
    let embedding_registry = self.embedding_registry.clone();   // ① 留一份 registry
    let parent = self.parent.clone();                           // ② 后厨电话
    let request = self.into_request()?;                         // ③ 解包数据（可能在此报暂存的错）
    Ok(Table::new_with_embedding_registry(                      // ④ 造 Table，带 registry
        parent.create_table(request).await?,                    //    委托后厨 Database::create_table
        embedding_registry,
    ))
}
```

**(b) `<false>` 专属（`:189-212`）：无数据的构造与执行**

| 方法 | 行号 | 作用 |
|---|---|---|
| `new(...)` | `:190` | 从 schema 造 builder，`data` 设为 `None` |
| `execute() -> Result<Table>` | `:207` | 下单，**裸 `Table::new`，不带 registry** |

`<false>::execute`（`connection.rs:207-211`）：

```rust
pub async fn execute(self) -> Result<Table> {
    Ok(Table::new(                                             // ★裸 Table::new，无 registry
        self.parent.clone().create_table(self.request).await?,
    ))
}
```

> **核对要点：两个 `execute` 的差别**。`<true>::execute`（`:147`）用 `Table::new_with_embedding_registry` 把 registry 传给下游 Table；`<false>::execute`（`:208`）用裸 `Table::new`，**不带 registry**。为什么？因为只有「有数据」时才可能需要现场计算 embedding 列，空表没数据可算，自然不需要 registry。这是 typestate 区分两套行为的实际收益。

**(c) 共享 `impl<const HAS_DATA: bool>`（`:214-350`）：两本菜单都有的配置**

| 方法 | 行号 | 作用 |
|---|---|---|
| `mode(mode)` | `:218` | 设建表模式（已存在时怎么办，见 §3.3） |
| `write_options(opts)` | `:224` | 写选项 |
| `storage_option(k,v)` | `:235` | 单条 storage 选项 |
| `storage_options(pairs)` | `:255` | 批量 storage 选项 |
| `add_embedding(def) -> Result<Self>` | `:279` | 加一个 embedding 列定义（带早期校验） |
| `enable_v2_manifest_paths(b)` | `:307` | 已废弃，设 manifest 路径版本 |
| `data_storage_version(v)` | `:333` | 已废弃，设文件版本 |

> **这就是 typestate 的妙处**：`mode`、`storage_option` 这些**两种建表都适用**的配置，写在共享 impl 里，`<true>` 和 `<false>` 都能调；而 `into_request`（解包数据）只对 `<true>` 有意义，就锁在 `<true>` 的 impl 里——`CreateTableBuilder<false>` 上**根本没有这个方法**，编译器不让你调。状态（有无数据）被编码进了类型，非法操作在编译期就被挡住。

### 3.3 `add_embedding` 的「早期校验」

`connection.rs:279-292`：

```rust
pub fn add_embedding(mut self, definition: EmbeddingDefinition) -> Result<Self> {  // ① 返回 Result！
    let embedding_func = self
        .embedding_registry
        .get(&definition.embedding_name)                    // ② 立刻去 registry 查这个名字
        .ok_or_else(|| Error::EmbeddingFunctionNotFound {   // ③ 查不到当场报错
            name: definition.embedding_name.clone(),
            reason: "No embedding function found …".to_string(),
        })?;
    self.embeddings.push((definition, embedding_func));     // ④ 找到了才存进 embeddings
    Ok(self)
}
```

注意它返回 `Result<Self>` 而非 `Self`——**链式 Builder 里少见的「会失败的一步」**。它在配置阶段就去 registry 查 embedding 名字（`:283`），查不到立即 `EmbeddingFunctionNotFound`（`:284`）。这叫**早期校验**：与其等到 `execute` 才发现 embedding 名拼错，不如在你勾菜单时就报。

### 3.4 `into_request`：暂存数据的「结账时核验」

const generic 还解决了一个**向后兼容**难题。看 `CreateTableBuilderInitialData`（`connection.rs:85-89`）：

```rust
enum CreateTableBuilderInitialData {
    None,                                                      // 无数据（<false> 用）
    Iterator(Result<Box<dyn RecordBatchReader + Send>>),      // 迭代器数据（注意：包着 Result！）
    Stream(Result<SendableRecordBatchStream>),                // 流式数据
}
```

关键在 `Iterator`/`Stream` 里包的是 `Result<...>`——`new` 时调 `data.into_arrow()`（`:119`）可能就失败了，但**错误被暂存**，不立即抛。注释（`:97-98`）写明：「为了向后兼容，把错误推迟到 `execute` 才抛」。

错误在 `into_request`（`connection.rs:153-185`）兑现：

```rust
fn into_request(self) -> Result<CreateTableRequest> {
    if self.embeddings.is_empty() {
        match self.data {
            CreateTableBuilderInitialData::Iterator(maybe_iter) => {
                let data = maybe_iter?;                        // ① 暂存的错误在这里 ? 抛出
                Ok(CreateTableRequest { data: CreateTableData::Data(data), ..self.request })
            }
            CreateTableBuilderInitialData::None => {
                unreachable!("No data provided for CreateTableBuilder<true>")  // ② 类型保证走不到这
            }
            CreateTableBuilderInitialData::Stream(maybe_stream) => {
                let data = maybe_stream?;
                Ok(CreateTableRequest { data: CreateTableData::StreamingData(data), ..self.request })
            }
        }
    } else {
        // 有 embedding：必须是迭代器，流式不支持
        let CreateTableBuilderInitialData::Iterator(maybe_iter) = self.data else {  // ③ let-else
            return Err(Error::NotSupported {                  // :176 流式 + embedding 不支持
                message: "Creating a table with embeddings is currently not support when the input is streaming".to_string(),
            });
        };
        let data = maybe_iter?;
        let data = Box::new(WithEmbeddings::new(data, self.embeddings));  // ④ 把数据包进 embedding 计算层
        Ok(CreateTableRequest { data: CreateTableData::Data(data), ..self.request })
    }
}
```

三个 Rust 知识点挤在这一个函数里：

> **`unreachable!()`（`:164`）**：标记「程序逻辑上不可能到达此处」，一旦真到了就 **panic**。这里 `<true>` 的 builder 永远有数据（`None` 只属于 `<false>`），所以 `Iterator`/`Stream` 之外的分支不可能发生。但 `match` 语法要求穷尽所有 enum 变体，于是用 `unreachable!` 填上。**typestate 在这里又一次发力**：是 const generic 保证了 `<true>` 永不为 `None`，否则这就是个真 bug。

> **`let-else`（`:175`）**：`let PATTERN = EXPR else { … }`。如果 `self.data` 不是 `Iterator(...)` 模式，就走 `else` 块。而 `else` 块**必须发散**（`return`/`panic`/`break` 等不返回值的东西）——这里是 `return Err(...)`。这是 Rust 1.65+ 的语法，**Python/JS 完全没有对应**，最接近的是「`if 不匹配 then 提前 return`」的早返回。

> **`..self.request`（结构体更新语法，`:160` 等）**：`CreateTableRequest { data: …, ..self.request }` 意思是「`data` 字段用新的，**其余字段全从 `self.request` 搬过来**」。类比 JS 的 `{ ...request, data }`。

> **比喻**：`CreateTableBuilderInitialData` 像建表时先收下的**一张预付小票**（数据已经尝试转成 Arrow，成败先记在票上）。直到 `into_request`（结账）才核验这张票——票没问题就放行，票上记着错误（`maybe_iter?`）就当场退单。而 typestate（`<true>`）保证了「带数据的菜单永远有票」，所以 `None` 分支才敢写 `unreachable!`。

---

## 4. `OpenTableBuilder` / `TableNamesBuilder`：另外两本点菜单

### 4.1 `OpenTableBuilder`：打开已有表

`connection.rs:352-357`：

```rust
#[derive(Clone, Debug)]
pub struct OpenTableBuilder {
    parent: Arc<dyn Database>,                       // 后厨电话
    request: OpenTableRequest,                       // 请求载体（database.rs:52）
    embedding_registry: Arc<dyn EmbeddingRegistry>,  // 传给下游 Table
}
```

它的 `request` 是 `OpenTableRequest`（`database.rs:52-56`），三个字段：`name`、`index_cache_size: Option<u32>`、`lance_read_params: Option<ReadParams>`。链式配置项：

| 方法 | 行号 | 作用 | 默认 |
|---|---|---|---|
| `index_cache_size(n)` | `:387` | 索引缓存条目数 | 256（文档注释 `:378`） |
| `lance_read_params(p)` | `:395` | 直接塞 lance 读参数（最高优先级） | `None` |
| `storage_option(k,v)` | `:406` | 单条 storage 选项 | — |
| `storage_options(pairs)` | `:425` | 批量 | — |
| `execute() -> Result<Table>` | `:445` | 打开，委托 `Database::open_table`，带 registry | — |

`execute`（`connection.rs:445-450`）和 `<true>` 建表一样，用 `Table::new_with_embedding_registry` 把 registry 传给打开的表：

```rust
pub async fn execute(self) -> Result<Table> {
    Ok(Table::new_with_embedding_registry(
        self.parent.clone().open_table(self.request).await?,   // 委托后厨
        self.embedding_registry,
    ))
}
```

> **Rust 陷阱：`get_or_insert(Default::default())` 的「惰性深层构造」**。看 `storage_option`（`:407-414`）：
> ```rust
> let storage_options = self.request
>     .lance_read_params.get_or_insert(Default::default())   // Option<ReadParams> → 没有就建一个
>     .store_options.get_or_insert(Default::default())       // 再钻一层 Option
>     .storage_options.get_or_insert(Default::default());    // 再钻一层 Option
> ```
> 这是一条「层层钻进嵌套 `Option`，缺哪层就**就地建哪层**」的链。`get_or_insert` 返回的是**可变引用**，所以能继续往里钻。类比 JS 的 `obj.a ??= {}; obj.a.b ??= {}; obj.a.b.c ??= {}` 然后操作 `c`。建表 Builder 的 `storage_option`（`:235-247`）是**一模一样**的三层钻法——记住这个形状，全库 storage 选项都长这样。

### 4.2 `TableNamesBuilder`：列表名（最简 Builder）

`connection.rs:40-43`：

```rust
pub struct TableNamesBuilder {
    parent: Arc<dyn Database>,
    request: TableNamesRequest,                  // database.rs:39
}
```

`TableNamesRequest`（`database.rs:39-48`）只有两个字段，对应两个分页配置：

| 方法 | 行号 | 作用 |
|---|---|---|
| `start_after(name)` | `:58` | 只返回字典序在此之后的名字（配合 limit 做分页） |
| `limit(n)` | `:64` | 最多返回几个 |
| `execute() -> Result<Vec<String>>` | `:70` | `self.parent.clone().table_names(request).await` |

这是全篇**最朴素的 Builder**——没有数据、没有 registry，就一个 request 加两个旋钮。测试 `test_table_names`（`connection.rs:986-1025`）演示了它的分页用法，是理解 Builder 链式调用的好例子。

### 4.3 `NoData`：一个「永不落地」的占位类型

`connection.rs:75-81`：

```rust
pub struct NoData {}

impl IntoArrow for NoData {
    fn into_arrow(self) -> Result<Box<dyn arrow_array::RecordBatchReader + Send>> {
        unreachable!("NoData should never be converted to Arrow")   // :79 一旦调用就 panic
    }
}
```

`NoData` 是个空结构体，给「不需要数据」的泛型场景占位（你会在 09 篇 `AddDataBuilder<NoData>` 见到它）。它假装实现了 `IntoArrow`，但 `into_arrow` 直接 `unreachable!`——意思是「类型系统上需要它能转 Arrow，但实际逻辑保证永不会真的调到」。又一处用 `unreachable!` 把「类型要求」和「运行时不可能」缝合的例子。

---

## 5. Catalog 连接：连锁总部前台

`Connection` 管的是「一个数据库里的表」。再往上一层是 **Catalog**——「一个目录里的多个数据库」。连接 Catalog 走另一条平行的路：`connect_catalog` / `CatalogConnectBuilder`。

### 5.1 平行结构

`connection.rs:910-912`：

```rust
pub fn connect_catalog(uri: &str) -> CatalogConnectBuilder {
    CatalogConnectBuilder::new(uri)
}
```

`CatalogConnectBuilder`（`connection.rs:872-875`）比 `ConnectBuilder` 更瘦——只裹一个 `ConnectRequest`，连 `embedding_registry` 都没有（catalog 这一层不碰 embedding）：

```rust
#[derive(Debug)]
pub struct CatalogConnectBuilder {
    request: ConnectRequest,                     // 复用同一个 ConnectRequest
}
```

| 方法 | 行号 | 作用 |
|---|---|---|
| `new(uri)` | `:879` | 造 builder，初始化 `ConnectRequest` |
| `catalog_options(opts)` | `:891` | `opts.serialize_into_map(&mut options)` —— 和 `database_options` 同款套路 |
| `execute() -> Result<Arc<ListingCatalog>>` | `:897` | 建连 |

`execute`（`connection.rs:897-900`）：

```rust
pub async fn execute(self) -> Result<Arc<ListingCatalog>> {
    let catalog = ListingCatalog::connect(&self.request).await?;   // :898 委托 ListingCatalog
    Ok(Arc::new(catalog))
}
```

注意它**只支持本地 `ListingCatalog`**（`:897` 返回类型写死 `Arc<ListingCatalog>`），没有 `db://` 云端分支——这是 catalog 与 database 两条路当前的差异。`catalog_options` 用的 `serialize_into_map` 来自 `CatalogOptions` trait（与 `DatabaseOptions` 同构），又是那张万能 options map 的复用。

### 5.2 `connect` vs `connect_catalog`：一对孪生入口

| 维度 | `connect` / `ConnectBuilder` | `connect_catalog` / `CatalogConnectBuilder` |
|---|---|---|
| 入口函数 | `connect`（`:867`） | `connect_catalog`（`:910`） |
| 管理粒度 | 一个数据库里的**表** | 一个目录里的**数据库** |
| 返回 | `Connection`（`:455`） | `Arc<ListingCatalog>`（`:897`） |
| 后厨 | `Database` trait（本地/云） | `ListingCatalog`（仅本地） |
| 带 embedding_registry | 是 | 否 |
| 选项注入 | `database_options`（`:709`） | `catalog_options`（`:891`） |

> **比喻**：如果说 `Connection` 是**一家餐厅的经理**，那 `Catalog` 就是**连锁品牌总部前台**——它管的是「有哪些分店（数据库）」，每家分店里再有自己的菜单（表）。两套入口结构高度平行，是同一套 Builder 设计在不同粒度上的复刻。测试 `test_connect_catalog`（`connection.rs:1246-1261`）和 `test_catalog_create_database`（`:1265`）展示了它的用法。

---

## 6. Builder 模式总结：为什么 Rust 数据库 API 爱用它

读到这里，你已见过本篇的 6 个 Builder：`ConnectBuilder`、`CreateTableBuilder<true/false>`、`OpenTableBuilder`、`TableNamesBuilder`、`CatalogConnectBuilder`。它们形态高度一致——这不是巧合，是 Rust 数据库 API 的标准答卷。

### 6.1 Builder 三段式

每个 Builder 都走同一条三段流水线：

```
① Connection/connect 返回一张 Builder        （不是 async，零 IO）
        ▼
② .xxx(...).yyy(...)  链式配置               （每步 mut self → Self，可选项摊开）
        ▼
③ .execute().await    下单                   （async，真正干活，返回 Result）
```

### 6.2 为什么是 Builder 而不是普通函数？

| Rust 的「缺陷」 | Builder 如何补救 |
|---|---|
| **没有可选参数** | 链式方法把每个可选项变成一次可选调用，不调就用默认 |
| **没有命名参数** | `.mode(x).limit(y)` 方法名即参数名，可读性拉满 |
| **没有函数重载** | `create_table`（带数据）/`create_empty_table`（无数据）用不同入口 + const generic 区分 |
| **链式中途可能失败** | `execute` 才返回 `Result`，中间步骤大多返回 `Self`，API 保持顺滑（**错误延迟**，见 §3.4） |

> **核心洞察：错误延迟（error deferral）**。Builder 的链式方法大多返回 `Self` 而非 `Result<Self>`，所以 `a.b().c().d()` 读起来一气呵成。真正可能失败的事（IO、数据转换）都攒到 `execute` 一次性抛。`CreateTableBuilderInitialData` 把 `into_arrow` 的错误暂存到 `execute` 才抛（§3.4），正是这一哲学的极致体现。**例外**是 `add_embedding`（§3.3）——它做的是配置期校验，所以破例返回 `Result<Self>`。

### 6.3 给贡献者的「改这个模块」指南

这是本篇最实用的一节。你想给 Connection/Builder 体系加功能，**90% 的情况落在这三类改动**，且都不碰执行逻辑：

1. **加一个连接配置项** → 在 `ConnectBuilder` 加一个 `mut self → Self` 的链式方法，往 `self.request.options` 塞一个新 key（仿 `:764` `storage_option`），再在下游 `parse_from_map` 取出来。
2. **加一个建表/开表配置项** → 在共享 `impl<const HAS_DATA>`（`:214`）或 `OpenTableBuilder`（`:359`）加链式方法，改 `self.request` 的字段。注意区分该选项是「两种建表都适用」（放共享 impl）还是「只对有/无数据适用」（放 `<true>`/`<false>` 专属 impl）。
3. **加一个数据库实现** → 实现 `Database` trait（`database.rs:151`），然后在 `execute`（`:845`）的路由里加分支。这是唯一会动执行逻辑的场景，也最少见。

> **反复强调：组装而非重造**。`Connection` 自己几乎不写业务逻辑——它把请求转发给 `Arc<dyn Database>`，把数据转发给 `Table`。Builder 也只是「攒配置 + 在 execute 时一次性委托」。**贡献者绝大多数时候是在「加一个旋钮」或「加一个 option key」，而不是重写执行流程。** 真正的存储/查询逻辑在 `ListingDatabase`/`RemoteDatabase`/lance 里，本模块只负责把用户意图整理成请求、选对后厨。

---

## 7. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `#[derive(Clone)]` + `Arc` | 克隆只 +1 引用计数，不搬数据 | `Connection` `:454` |
| `Arc<dyn Database>` | 引用计数 + 运行时多态，本地/云端通吃 | `internal` `:457` |
| `#[cfg(feature = "remote")]` | 条件编译：没开 feature 字段/方法都不存在 | `client_config` `:613`、`execute_remote` `:806`/`:837` |
| `const HAS_DATA: bool`（const generic） | 编译期常量当类型参数，`<true>`/`<false>` 是不同类型 | `CreateTableBuilder` `:92` |
| 三个 impl 块按 const 切分 | 状态编进类型，非法方法编译期被挡 | `:103` / `:189` / `:214` |
| `mut self -> Self` | 消费自身的链式 Builder，调用后原变量被移动 | 几乎所有链式方法，如 `:218` |
| `unreachable!()` | 标记「逻辑不可达」，真到了就 panic | `:79`、`:164` |
| `let PATTERN = EXPR else { … }`（let-else） | 不匹配就走发散的 else 块 | `into_request` `:175` |
| `..self.request`（结构体更新语法） | 其余字段从已有结构体搬来 | `:160` 等 |
| `get_or_insert(Default::default())` | 嵌套 Option 缺层就地建层，链式钻入 | `:236-247`、`:407-414` |
| `unwrap_or_else(\|\| …)` | 可选值缺省时惰性造默认 | `:830`、`:853` |
| `?` 运算符 | 出错就提前 return 这个错误 | `maybe_iter?` `:157` 等 |
| `HashMap<String,String>` 万能信封 | 任意配置先塞 map，用时再 parse | `options` `:617` |

---

## 8. 动手验证（建议亲手做一遍）

1. **数一数 Builder**：在 `connection.rs` 搜 `pub struct.*Builder`，确认本篇 6 个 Builder（`TableNames` `:40`、`CreateTable` `:92`、`OpenTable` `:353`、`Connect` `:633`、`CatalogConnect` `:873`）。对每个找出它的 `execute`，确认都返回 `Result`。
2. **验证 typestate 挡错**：在测试里写 `db.create_empty_table("t", schema).into_request()`，编译。你会发现**编译失败**——因为 `into_request`（`:153`）只属于 `<true>`，而 `create_empty_table` 返回 `<false>`（`:535`）。这就是 const generic typestate 在保护你。
3. **跟一次建连路由**：从 `connect(uri)`（`:867`）追到 `ConnectBuilder::execute`（`:845`），分别用 `/tmp/foo`、`s3://b/x`、`db://x` 三种 uri 走读分支，确认前两者走 `ListingDatabase`（`:849`）、第三者走 `execute_remote`（`:847`）。
4. **抓「万能 options 信封」**：搜 `self.request.options.insert`，统计有多少链式方法在往那张 map 塞 key。再看 `execute_remote`（`:809`）如何用 `parse_from_map`（`remote/db.rs:92`）把它们读回来——感受「塞与取分居两端」。
5. **对照孪生入口**：把 `ConnectBuilder`（`:633`）和 `CatalogConnectBuilder`（`:873`）并排读，列出它们的异同（提示：后者无 embedding_registry、无 remote 分支、返回类型不同）。理解「同一套 Builder 设计在不同粒度上的复刻」。

---

## 9. 小结 & 下一篇

- `Connection` 是**餐厅经理**：3 个字段（`uri`/`internal`/`embedding_registry`），核心是 `Arc<dyn Database>` 后厨电话，方法非「现造 Builder」即「立即 async 转发」。
- **一切入口都是 Builder**：`connect`→`ConnectBuilder`、`create_table`→`CreateTableBuilder<true>`、`create_empty_table`→`CreateTableBuilder<false>`、`open_table`→`OpenTableBuilder`、`table_names`→`TableNamesBuilder`。三段式：返回 Builder → 链式配置 → `execute().await` 下单。
- **`connect` 的 URI 三形态路由**：`db://` 走 `RemoteDatabase`，其余走 `ListingDatabase`；所有配置先塞进 `options: HashMap`，下游 `parse_from_map` 取出。
- **const generic typestate 是本篇技术高潮**：`CreateTableBuilder<true/false>` 用编译期 bool 把「有无数据」编进类型，三个 impl 块切分方法集，用错方法编译失败；`unreachable!`/`let-else` 与之配合处理「类型保证不可能发生」的分支。
- **Catalog 连接**是 database 连接的孪生平行结构，高一个粒度（管数据库而非表）。
- **贡献者要诀**：90% 的改动是「加一个链式方法 + 往 options 塞 key」或「改 request 字段」，几乎不碰执行逻辑——**组装而非重造**。

**下一篇可读 `09/10 Table 解剖`**：你在本篇用 Builder 造出的 `Table`，到了那里就是主角——看它如何握着 `Arc<dyn BaseTable>` 把请求转给本地/云端后厨。本篇的 `Connection`/`Database` 与那篇的 `Table`/`BaseTable` 是**同一套门面架构的上下游两半**，对照着读最透。
