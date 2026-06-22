# 15 · Remote 远程表与云端：把"后厨"搬到天上

> **本篇解剖的源码**：`rust/lancedb/src/remote/` 整个目录 —— `remote.rs`（22 行总览）、`db.rs`（`RemoteDatabase`）、`table.rs`（`RemoteTable`）、`client.rs`（HTTP 客户端）、`util.rs`（IPC 序列化 / 版本解析）
> **覆盖区间**：`remote.rs:1–22`、`db.rs:32–435`、`table.rs:47–1055`、`client.rs:16–656`、`util.rs:1–48`，以及路由入口 `connection.rs:805–858`
> **前置阅读**：`01 Rust 垫脚石`（trait / `dyn` / `Arc` / async 四节）、`03 连接建立链路`（`ConnectBuilder` 怎么落到具体 `Database`）。强烈建议**先读完 `09 Table 解剖（上）`**——本篇大量与它的 `NativeTable` 同名方法对照。
> **与哪些篇互补**：`09 / 10` 讲本地后厨 `NativeTable`，本篇讲云端后厨 `RemoteTable`，**两者实现同一份合同 `BaseTable`**；`11 query 体系`讲查询对象怎么生成，本篇讲查询对象怎么**序列化成 HTTP**；`12 index 体系`讲索引类型，本篇讲索引类型怎么**映射成 REST 字符串**。

---

## 0. 本篇要回答的三个问题

如果你只带走三句话，就是这三句：

1. **云端表"伪装"成本地表**——`RemoteTable` 和 `NativeTable` 签的是**同一份用工合同** `BaseTable`（`table.rs:418` vs 09 篇的 `table.rs:1886`）。上层的 `Table` 服务员根本分不清自己端的菜是楼下现炒还是外地空运。
2. **远程后厨不亲自颠勺，只打电话**——`RemoteTable` 的每个方法都把请求**翻译成一通 HTTP**：`POST /v1/table/菜名/动作/`，body 是真空打包的 Arrow 数据（IPC 字节流）。
3. **整个 remote 模块是个"翻译层 + 可选插件"**——它不自写 HTTP（复用 reqwest）、不自定义序列化（复用 arrow_ipc）、不自做查询计划（把远端结果流包进 DataFusion）。而且它**只在 `feature = "remote"` 打开时才编进二进制**（`lib.rs:206`）。

画成一张图，就是 remote 模块的骨架：

```
         你的代码:  connect("db://my-db").open_table("t")
            │
            │  ① ConnectBuilder::execute  (connection.rs:846)
            │     uri.starts_with("db") ?  ── 真 ──▶ execute_remote (connection.rs:806)
            ▼
  ┌──────────────────────────┐
  │  RemoteDatabase (经理)     │  db.rs:176   ← impl Database
  │  client + table_cache     │
  └───────┬──────────────────┘
          │  open_table / create_table ⇒ HTTP
          ▼
  ┌──────────────────────────┐         ┌──────────────────────────┐
  │ RestfulLanceDbClient(电话机)│◀──────│  RemoteTable (遥控对讲机)   │  table.rs:50
  │ client.rs:160             │  借用   │  impl BaseTable           │  table.rs:418
  │ host / retry / sender:S   │         │  client / name / version  │
  └───────┬──────────────────┘         └──────────────────────────┘
          │  send() → 重试循环 → sender.send()
          ▼
  ┌──────────────────────────┐
  │  HttpSend trait (话筒)      │  client.rs:167
  │  Sender(真) / MockSender(假) │
  └───────┬──────────────────┘
          ▼
     reqwest → LanceDB Cloud（云端真后厨 / Phalanx 服务）
```

> **比喻总钥匙**（延续 09 篇家族）：
> - `BaseTable` = **用工合同**；`Table` = **服务员**；`NativeTable` = **本地后厨**。
> - 本篇新增三位：**`RemoteDatabase` = 远程分店的餐厅经理**（本人不在本地后厨，所有指令通过电话下达云端，手里攥着熟客名册 `table_cache` 避免每次都打电话确认餐桌）；**`RestfulLanceDbClient` = 经理桌上的专线电话机**（拨号、报工号、占线重拨、超时挂断）；**`RemoteTable` = 云端后厨的遥控对讲机**（拿同一份合同，但每道命令都翻译成一通电话发给云端真后厨）。
> - **`HttpSend` = 话筒**：真话筒 `Sender` 接通云端，测试时换成录音机 `MockSender` 放预设答复。

---

## 1. 总览：远程后端如何"伪装"成本地一样

### 1.1 `remote.rs`：22 行的模块门牌

整个子系统的入口只有 22 行（`remote.rs:1–22`）。它做三件事：

```rust
pub(crate) mod client;   // ① 电话机    remote.rs:9
pub(crate) mod db;       //   经理        remote.rs:10
pub(crate) mod table;    //   遥控对讲机   remote.rs:11
pub(crate) mod util;     //   打包/解析工具 remote.rs:12

const ARROW_STREAM_CONTENT_TYPE: &str = "application/vnd.apache.arrow.stream";  // remote.rs:14 ②

pub use client::{ClientConfig, RetryConfig, TimeoutConfig};                     // remote.rs:20 ③
pub use db::{RemoteDatabaseOptions, RemoteDatabaseOptionsBuilder};              // remote.rs:21
```

- **①** 四个子模块全是 `pub(crate)`——只在 crate 内部可见，外界够不着 `RemoteTable` 这种具体类型，只能透过 `Arc<dyn BaseTable>` 用它。
- **②** `ARROW_STREAM_CONTENT_TYPE` 是后续所有"上传数据"请求的 `Content-Type` 头（写入、建表、merge 都贴这个标签）。另有两个 `#[cfg(test)]` 的 content type（`ARROW_FILE` / `JSON`，`remote.rs:16,18`）只在测试里出现。
- **③** **只 re-export 配置类**（`ClientConfig` 等）和**选项类**给用户。后厨本体 `RemoteTable`、电话机 `RestfulLanceDbClient` 一律不导出。

> **Rust 知识点：`pub(crate)`**。可见性修饰符。`pub` = 谁都能用；`pub(crate)` = 只有本 crate 内的代码能用；不写 = 只有本模块能用。remote 模块刻意把实现细节锁在 `pub(crate)`，对外只露一个"实现了 `BaseTable` 的盒子"。这就是**封装**——你想改 `RemoteTable` 的字段，不必担心外部用户依赖了它。

### 1.2 一份合同两个后厨：与 09 篇的对接点

回忆 09 篇 §1.3 的那张图——`BaseTable`（合同）下面挂着 `NativeTable` 和 `RemoteTable` 两家后厨。本篇就是把右边那家后厨打开。`BaseTable` 定义在 `table.rs:408`（即 `lancedb/src/table.rs`，**注意不是 remote 目录**），而 `RemoteTable` 兑现这份合同的地方是 `remote/table.rs:417-418`：

```rust
#[async_trait]
impl<S: HttpSend> BaseTable for RemoteTable<S> {   // table.rs:418
    fn as_any(&self) -> &dyn std::any::Any { self } // table.rs:419
    fn name(&self) -> &str { &self.name }           // table.rs:422
    async fn version(&self) -> Result<u64> { ... }  // table.rs:425
    // …… 逐条兑现 09 篇那 29 个方法
}
```

**这是理解整个模块的总钥匙**：上层 `Table`（服务员）只持有 `Arc<dyn BaseTable>`，调 `table.add(...)` 时，动态分发既可能落到 `NativeTable::add`（本地写盘），也可能落到 `RemoteTable::add`（发 HTTP），**调用方一行 `if 本地 else 云端` 都不用写**。

> **学习路径建议**（来自模块作者的隐含顺序）：`remote.rs`（22 行门牌）→ `connection.rs:805-858`（路由）→ `db.rs`（经理）→ `client.rs`（电话机）→ `table.rs`（后厨）。本篇正按此顺序展开。

---

## 2. `feature = "remote"`：为什么远程是可选插件

整个 remote 子系统挂在一个**条件编译开关**后面（`lib.rs:206-207`）：

```rust
#[cfg(feature = "remote")]   // lib.rs:206
pub mod remote;              // lib.rs:207
```

只有在 `Cargo.toml` 里开启 `remote` feature，`mod remote` 才会被编进二进制；**不开启时，这几千行代码对编译器而言根本不存在**——不会编译、不进产物、不拉 reqwest 依赖。

路由入口也对称地准备了两个版本（`connection.rs:805-842`）：

```rust
#[cfg(feature = "remote")]            // connection.rs:805
fn execute_remote(self) -> Result<Connection> {
    // …真正建 RemoteDatabase…  (connection.rs:806-834)
}

#[cfg(not(feature = "remote"))]       // connection.rs:836  ← 兜底版
fn execute_remote(self) -> Result<Connection> {
    Err(Error::Runtime {              // connection.rs:838
        message: "cannot connect to LanceDb Cloud unless the 'remote' feature is enabled".to_string(),
    })
}
```

> **Rust 陷阱：`#[cfg(...)]` 条件编译**。`#[cfg(feature = "remote")]` 是"编译期 `if`"——满足条件这段代码才存在。注意它和 C 的 `#ifdef` 一样是**编译期**的，不是运行时分支。两个同名 `execute_remote` 看似冲突，但因为 `cfg` 与 `not(cfg)` 互斥，**任何一次编译只会有一个进入**，绝不会重复定义。这就是为什么没开 feature 时调 `connect("db://…")` 会拿到一个清晰的 `Error::Runtime`，而不是编译失败。

> **为什么要做成可选？** 纯本地用户（嵌入式向量库场景）不需要 HTTP/reqwest/重试这一整套，把它做成 feature 能让他们的二进制更小、依赖更少、编译更快。**"用得上才编进来"** 是 Rust 生态控制依赖膨胀的标准手法。

路由判据本身在 `connection.rs:846`：

```rust
if self.request.uri.starts_with("db") {   // connection.rs:846  ← 注意是 "db" 前缀，不是严格 "db://"
    self.execute_remote()
} else {
    // 本地 ListingDatabase  (connection.rs:849)
}
```

`execute_remote`（`connection.rs:806`）随后做三步：① `parse_from_map` 取出 `region` / `api_key` / `host_override`（缺 `region` 或 `api_key` 直接 `InvalidInput`，`connection.rs:811-816`）；② 把剩下的 storage_options 转成 `RemoteOptions`；③ 调 `RemoteDatabase::try_new`（`connection.rs:819`）。

---

## 3. `RemoteDatabase`：实现 `Database` 的"经理"

### 3.1 结构体：只有两个字段

`db.rs:175-179`：

```rust
#[derive(Debug)]
pub struct RemoteDatabase<S: HttpSend = Sender> {   // db.rs:176
    client: RestfulLanceDbClient<S>,                // ① 专线电话机
    table_cache: Cache<String, Arc<RemoteTable<S>>>, // ② 熟客名册（异步缓存）
}
```

经理本人**也薄得惊人**——只有"电话机"和"熟客名册"两样。

> **Rust 陷阱：泛型默认参数 `S: HttpSend = Sender`**。这等价于 TypeScript 的 `class RemoteDatabase<S = Sender>`。`S` 是"用哪个发请求实现"的占位符，默认填生产实现 `Sender`。所以日常代码里 `RemoteDatabase` 就是 `RemoteDatabase<Sender>`，而测试里写 `RemoteDatabase<MockSender>` 把发送动作换成假的。**这是编译期依赖注入——换 `S` 就换发请求实现，零运行时开销**（不像 Java 的接口注入要运行时虚调用）。

> **Rust 知识点：`moka::future::Cache`**（`db.rs:11`）。一个**异步并发缓存**，类比"带 TTL 的 `ConcurrentHashMap`"。它的 `get` / `insert` / `remove` 都是 `async`（要 `.await`），内部自带过期与并发安全。

### 3.2 `try_new`：建电话机 + 建名册

`db.rs:181-209`：

```rust
impl RemoteDatabase {                  // 注意：这个 impl 块只针对默认的 RemoteDatabase<Sender>
    pub fn try_new(uri, api_key, region, host_override, client_config, options) -> Result<Self> {
        let client = RestfulLanceDbClient::try_new(...)?;   // ① 造电话机（见 §5）  db.rs:190
        let table_cache = Cache::builder()
            .time_to_live(std::time::Duration::from_secs(300))  // ② 熟客名册存 300 秒  db.rs:200
            .max_capacity(10_000)                                //    最多记 10000 张桌  db.rs:201
            .build();
        Ok(Self { client, table_cache })
    }
}
```

熟客名册的语义：**记住"这张表存在且我已确认过"**，TTL 300 秒，超量自动淘汰。它的价值在 `open_table` 里立刻兑现。

### 3.3 `impl Database`：四类方法各自变成 HTTP

`RemoteDatabase` 兑现 `Database` 合同从 `db.rs:248` 开始。挑三个核心方法看"指令怎么变成请求"。

#### (a) `open_table`：先查名册，再打电话确认

`db.rs:357-379`：

```rust
async fn open_table(&self, request: OpenTableRequest) -> Result<Arc<dyn BaseTable>> {
    if let Some(table) = self.table_cache.get(&request.name).await {  // ① 名册命中 → 不打电话
        Ok(table.clone())                                            //    db.rs:359-360
    } else {
        let req = self.client.post(&format!("/v1/table/{}/describe/", request.name)); // ② POST describe  db.rs:362-364
        let (request_id, rsp) = self.client.send(req, true).await?;
        if rsp.status() == StatusCode::NOT_FOUND {                   // ③ 404 → TableNotFound  db.rs:366
            return Err(crate::Error::TableNotFound { name: request.name });
        }
        let rsp = self.client.check_response(&request_id, rsp).await?;
        let version = parse_server_version(&request_id, &rsp)?;      // ④ 读 phalanx-version 头  db.rs:370
        let table = Arc::new(RemoteTable::new(self.client.clone(), request.name.clone(), version));
        self.table_cache.insert(request.name, table.clone()).await;  // ⑤ 写进名册  db.rs:376
        Ok(table)
    }
}
```

> **缓存语义务必记牢**（改这些方法时同步维护 `table_cache`）：
> - `open_table` 命中缓存就**不发 describe 请求**（`db.rs:359`）——省一次往返。
> - `rename_table` 会**搬移缓存键**：`remove(旧名)` 再 `insert(新名)`（`db.rs:388-391`）。
> - `drop_table` 会**删缓存**（`db.rs:399`）。
>
> 如果你给 `RemoteDatabase` 加方法而忘了维护名册，就会出现"删了表还能从缓存拿到"的诡异 bug。

#### (b) `create_table`：三分支 + 后台线程序列化

`db.rs:277-355`：

```rust
async fn create_table(&self, request: CreateTableRequest) -> Result<Arc<dyn BaseTable>> {
    let data = match request.data {
        CreateTableData::Data(data) => data,                       // ① 有数据：直接用
        CreateTableData::StreamingData(_) => {                     // ② 流式数据：云端不支持！
            return Err(Error::NotSupported {                       //    db.rs:280-284
                message: "Creating a remote table from a streaming source".to_string(),
            })
        }
        CreateTableData::Empty(table_definition) => {              // ③ 空表：造一个空 RecordBatchIterator
            Box::new(RecordBatchIterator::new(vec![], table_definition.schema.clone()))  // db.rs:285-288
        }
    };

    let data_buffer = spawn_blocking(move || batches_to_ipc_bytes(data)).await.unwrap()?;  // ④ db.rs:294
    let req = self.client
        .post(&format!("/v1/table/{}/create/", request.name))
        .query(&[("mode", Into::<&str>::into(&request.mode))])     // ⑤ ?mode=create/overwrite/exist_ok  db.rs:301
        .body(data_buffer)
        .header(CONTENT_TYPE, ARROW_STREAM_CONTENT_TYPE);
    let (request_id, rsp) = self.client.send(req, false).await?;
    // …400 + "already exists" 按 mode 分支处理…  db.rs:307-342
}
```

- **④ `spawn_blocking`**：把 Arrow IPC 序列化（`batches_to_ipc_bytes`）丢到**专门的阻塞线程池**。这一步是 CPU 密集 + 可能慢（数据大），若直接在异步任务里跑会**霸占 tokio 的异步 worker**，拖垮整个 runtime 的并发。
- **⑤ `?mode=`**：`CreateTableMode` 通过 `From<&CreateTableMode> for &'static str`（`db.rs:238-246`）转成 `"create"` / `"overwrite"` / `"exist_ok"` 三个查询字符串。
- **400 分支**（`db.rs:307-342`）：服务端返回 400 且 body 含 `"already exists"` 时，按 `mode` 决定——`Create` 抛 `TableAlreadyExists`、`ExistOk` 跑回调后转 `open_table`、`Overwrite` 抛 `Http`（老服务端不认 mode 参数时的兜底）。

> **Rust 陷阱：`spawn_blocking(...).await.unwrap()?`**（`db.rs:294`）。两层"解包"叠在一起，容易看花眼：
> - `spawn_blocking` 返回 `JoinHandle`，`.await` 拿到 `Result<T, JoinError>`——**`.unwrap()` 处理的是"线程 panic 了吗"**（panic 就直接崩，这里认为不可能）。
> - 里层 `batches_to_ipc_bytes` 返回 `Result<Vec<u8>>`——**末尾的 `?` 处理的是"序列化出错了吗"**（出错就把错误向上传播）。

#### (c) `drop_all_tables`：直接拒绝

`db.rs:403-407`：

```rust
async fn drop_all_tables(&self) -> Result<()> {
    Err(crate::Error::NotSupported {
        message: "Dropping databases is not supported in the remote API".to_string(),
    })
}
```

> **`NotSupported` 占位方法 = 菜单上的灰色条目**。本地有、云端还没开的能力，直接礼貌回绝。整个 remote 模块里有好几处这种占位（下文 §4.4 还会见到），它们是"协议尚未覆盖"的标记。

#### 四个 `Database` 方法对照

| 方法 | 行号 | HTTP | 备注 |
|---|---|---|---|
| `table_names` | `db.rs:250` | `GET /v1/table/` | 支持 `limit` / `page_token` 分页（`db.rs:252-257`）；顺便把列出的表写进名册 |
| `create_table` | `db.rs:277` | `POST /v1/table/{name}/create/?mode=` | 见上 |
| `open_table` | `db.rs:357` | `POST /v1/table/{name}/describe/` | 先查缓存 |
| `rename_table` | `db.rs:381` | `POST /v1/table/{name}/rename/` | body `{new_table_name}`，搬缓存键 |
| `drop_table` | `db.rs:395` | `POST /v1/table/{name}/drop/` | 删缓存 |
| `drop_all_tables` | `db.rs:403` | —— | `NotSupported` |

### 3.4 配置类：`RemoteDatabaseOptions` 与 `ServerVersion`

`RemoteDatabaseOptions`（`db.rs:68`）装四样：`api_key` / `region` / `host_override`（企业版）/ `storage_options`。它 `impl DatabaseOptions`（`db.rs:110`），并有 `parse_from_map`（`db.rs:92`）从字符串 map 解析。配套 Builder（`db.rs:128`）提供三个 setter（`db.rs:144-167`）。

`ServerVersion`（`db.rs:34`）是理解"协议演进"的关键：

```rust
pub const DEFAULT_SERVER_VERSION: semver::Version = semver::Version::new(0, 1, 0);  // db.rs:32
pub struct ServerVersion(pub semver::Version);                                       // db.rs:34

impl ServerVersion {
    pub fn support_multivector(&self) -> bool { self.0 >= semver::Version::new(0, 2, 0) }      // db.rs:52
    pub fn support_structural_fts(&self) -> bool { self.0 >= semver::Version::new(0, 3, 0) }   // db.rs:56
}
```

> **`ServerVersion` 的 `support_xxx` = 看对方厨房的设备清单**。服务端能力随版本演进，客户端**先问清云端装没装新设备**（0.2.0 多向量灶 / 0.3.0 结构化全文检索灶），再决定用新菜谱还是老菜谱下单。版本号从响应头 `phalanx-version` 读取（`util.rs:29-48`），缺失就用默认 `0.1.0`。
>
> **给贡献者的模式**（`db.rs:30-31` 注释明说）：要加需要客户端配合的新特性，**先 bump 服务端版本号，再加一个 `support_xxx()` 判定方法**，然后在 `RemoteTable` 里用它分流新旧协议。这是本模块演进的标准姿势。

`RemoteOptions`（`db.rs:416`）是 `StorageOptions` 的子集，`From<StorageOptions>`（`db.rs:424-435`）**只保留 `account_name` 和 `azure_storage_account_name` 两个键**——其余存储选项在远程场景无意义，直接丢弃。

---

## 4. `RemoteTable`：实现 `BaseTable` 的"遥控对讲机"

### 4.1 结构体：四个字段，一把锁

`table.rs:49-57`：

```rust
#[derive(Debug)]
pub struct RemoteTable<S: HttpSend = Sender> {  // table.rs:50
    #[allow(dead_code)]
    client: RestfulLanceDbClient<S>,            // ① 借用经理的电话机
    name: String,                               // ② 表名（拼进 URL）
    server_version: ServerVersion,              // ③ 云端能力清单
    version: RwLock<Option<u64>>,               // ④ 当前钉住的版本号
}
```

字段 ④ 最值得说：

> **Rust 陷阱：`RwLock<Option<u64>>`**（`table.rs:56`）。一把读写锁包着"当前钉住的版本号"：
> - `None` = 没钉版本，**指向最新、可写**。
> - `Some(v)` = 钉在历史版本 `v`，**只读快照**。
>
> 读锁用 `self.version.read().await`（`table.rs:317`），写锁用 `self.version.write().await`（`table.rs:441`）。注意这是 **tokio 的异步 `RwLock`**：`.await` 等锁时**让出线程**而非阻塞，别的任务还能跑。

### 4.2 三个翻译工具方法

`RemoteTable` 把"Rust 对象 → HTTP"的翻译活儿封装在几个私有方法里，它们被 `impl BaseTable` 的各方法反复调用：

| 方法 | 行号 | 作用 | 关键点 |
|---|---|---|---|
| `reader_as_body` | `table.rs:107` | 把 `RecordBatchReader` 抽真空打包成流式 `reqwest::Body` | 用 `arrow_ipc::StreamWriter` + `std::iter::from_fn`；`Mutex` 仅为满足 `Sync`（`table.rs:111`） |
| `read_arrow_stream` | `table.rs:144` | 把响应解析回 RecordBatch 流 | **目前无法真流式**：先 `bytes().await` 全读，再 `FileReader` 解析（注释引 arrow-rs issue 6420，`table.rs:151-153`） |
| `check_table_response` | `table.rs:130` | 统一错误处理 | 先把 404 转 `TableNotFound`（`table.rs:135`），再委托 `client.check_response` |

> **Arrow IPC 字节流 = 真空打包的预制菜**。`reader_as_body`（上传）和 `read_arrow_stream`（下载）就是抽真空打包 / 拆箱两个动作。`RemoteTable` 不自己设计序列化格式，**直接复用 `arrow_ipc`**——又一处"组装而非重造"。

> **Rust 陷阱：`std::iter::from_fn` + `Mutex` 造流**（`table.rs:112-125`）。`from_fn` 接一个闭包，每次调用产出迭代器的下一项——用来把"逐个 batch 写 IPC"包装成一个流。这里的 `Mutex` **不是为了并发**，纯粹是因为 `reqwest::Body::wrap_stream` 要求内部状态 `Sync`，而裸 reader 不是 `Sync`，套个 `Mutex` 骗过编译器（注释 `table.rs:111` 明说"shouldn't have any contention"）。读到这种"为满足 trait 约束而存在的锁"别误以为有竞争。

### 4.3 写方法的统一骨架 + 可变性守卫

所有写方法（`add` / `update` / `delete` / `create_index` / `merge_insert` / `add_columns` …）开头第一行都是 `self.check_mutable().await?`。

`check_mutable`（`table.rs:316-327`）：

```rust
async fn check_mutable(&self) -> Result<()> {
    let read_guard = self.version.read().await;          // 拿读锁看版本
    match *read_guard {
        None => Ok(()),                                  // 没钉版本 → 放行
        Some(version) => Err(Error::NotSupported {       // 钉了版本 → 挡回
            message: format!(
                "Cannot mutate table reference fixed at version {}. Call checkout_latest() to get a mutable table reference.",
                version),
        })
    }
}
```

> **`check_mutable` 守卫 = "历史快照菜单不接受修改"的告示牌**。一旦 `checkout` 到某历史版本（`version = Some`），**所有写操作都被前台挡回**，提示你先 `checkout_latest()` 切回当前菜单。这与 09/10 篇本地表的时间旅行语义一致，只是本地靠 `DatasetConsistencyWrapper`，远程靠这把 `RwLock`。

以 `add` 为例看完整骨架（`table.rs:526-551`）：

```rust
async fn add(&self, add: AddDataBuilder<NoData>, data: Box<dyn RecordBatchReader + Send>) -> Result<()> {
    self.check_mutable().await?;                                    // ① 守卫
    let body = Self::reader_as_body(data)?;                         // ② 真空打包
    let mut request = self.client
        .post(&format!("/v1/table/{}/insert/", self.name))         // ③ 拼 endpoint
        .header(CONTENT_TYPE, ARROW_STREAM_CONTENT_TYPE)
        .body(body);
    match add.mode {                                                // ④ Overwrite 加 ?mode=overwrite
        AddDataMode::Append => {}
        AddDataMode::Overwrite => { request = request.query(&[("mode", "overwrite")]); }
    }
    let (request_id, response) = self.client.send(request, false).await?;  // ⑤ 发送
    self.check_table_response(&request_id, response).await?;       // ⑥ 检错
    Ok(())
}
```

**几乎所有写方法都是这六步**：守卫 → 构造 body → 拼 endpoint → 发送 → 检错 → 返回。读懂这个骨架，`update`（`table.rs:683`）、`delete`（`table.rs:706`）、`add_columns`（`table.rs:833`）、`alter_columns`（`table.rs:867`）、`drop_columns`（`table.rs:899`）你就全懂了。

> **一个"白色谎言"**：`update`（`table.rs:683`）末尾**写死返回 `Ok(0)`**（`table.rs:703`），因为 SaaS 暂不返回受影响行数（TODO 注释）。本地 `NativeTable` 这里返回真实行数。改 SaaS 协议时这是个待填的坑。

### 4.4 查询方法：把 `QueryRequest` 序列化成 JSON

查询是 remote 模块最复杂的翻译。`execute_query`（`table.rs:334`）是总入口：

```rust
async fn execute_query(&self, query: &AnyQuery, options: &QueryExecutionOptions)
    -> Result<Vec<Pin<Box<dyn RecordBatchStream + Send>>>> {
    let mut request = self.client.post(&format!("/v1/table/{}/query/", self.name));
    if let Some(timeout) = options.timeout {
        request = request.timeout(timeout);                              // ① 客户端超时  table.rs:343
        if let Ok(timeout_ms) = u64::try_from(timeout.as_millis()) {
            request = request.header(REQUEST_TIMEOUT_HEADER, timeout_ms); // ② 同时告知服务端 x-request-timeout-ms  table.rs:347
        }
    }
    let query_bodies = self.prepare_query_bodies(query).await?;          // ③ 生成一或多个 JSON body  table.rs:351
    let requests = query_bodies.into_iter()
        .map(|body| request.try_clone().unwrap().json(&body)).collect(); // ④ 每个 body 克隆一份请求
    let futures = requests.into_iter().map(|req| async move {
        let (request_id, response) = self.client.send(req, true).await?;
        self.read_arrow_stream(&request_id, response).await             // ⑤ 解析回流
    });
    let streams = futures::future::try_join_all(futures).await?;        // ⑥ 并发等所有请求
    Ok(streams)
}
```

`prepare_query_bodies`（`table.rs:365`）按 `AnyQuery` 分派：普通 `Query` 走 `apply_query_params`，向量 `VectorQuery` 走 `apply_vector_query_params`。

`apply_query_params`（`table.rs:160`）把查询参数逐个写进 JSON：`prefilter` / `offset` / `k`（limit，服务端必填，缺省 `usize::MAX`，`table.rs:171-172`）/ `filter` / `columns` / `fast_search` / `with_row_id` / `full_text_query`。其中两处**只支持子集**：

- **filter 只支持 SQL**：非 `QueryFilter::Sql` 直接 `NotSupported`（`table.rs:174-181`）。
- **FTS 按版本走新旧格式**：`support_structural_fts()` 为真走新格式 `{query}`，否则走老格式 `{columns, query}`（`table.rs:219-228`）；`wand_factor` 在 Cloud 未支持（`table.rs:213-216`）。

`apply_vector_query_params`（`table.rs:234`）先写通用参数（`distance_type` / `nprobes` / `ef` / `refine_factor` 等，`table.rs:242-247`），再按**查询向量个数**分三支（`table.rs:280`）：

| `query_vector.len()` | 处理 | 行号 |
|---|---|---|
| `0` | 写空数组（无向量搜索） | `table.rs:281-284` |
| `1` | 单向量直接写 | `table.rs:286-288` |
| 多个 | 看 `support_multivector()`：支持 → 一个请求里塞数组；不支持 → **拆成多个请求**各发一遍 | `table.rs:290-310` |

> **多向量拆请求 = 老厨房一次只收一份单**。当云端版本 < 0.2.0（不支持多向量），客户端把 N 个查询向量拆成 N 个 HTTP 请求并发发出，回来再合并。合并靠把每个远端流包成 DataFusion 的 `OneShotExec`，再用 `Table::multi_vector_plan` 拼成一个执行计划（`table.rs:563-567`、`table.rs:583-592`）。

> **组装而非重造（查询版）**：`RemoteTable` 拿到远端返回的字节流后，**不自己写查询计划框架**——它把流包进 DataFusion 的 `OneShotExec` / `RecordBatchStreamAdapter`（`table.rs:19,27,561`），让上层像处理本地查询一样处理远程结果。云端是"一次性数据源"，DataFusion 是"传菜流水线"，无缝对接。

### 4.5 `create_index`：索引类型 → REST 字符串

`create_index`（`table.rs:718-804`）的核心是一张**类型映射表**（`table.rs:741-789`），把 Rust 的 `Index` 枚举翻译成 `(类型字符串, 距离类型)`：

```rust
let (index_type, distance_type) = match index.index {
    Index::IvfFlat(index)  => ("IVF_FLAT",   Some(index.distance_type)),  // table.rs:744
    Index::IvfPq(index)    => ("IVF_PQ",     Some(index.distance_type)),
    Index::IvfHnswSq(index)=> ("IVF_HNSW_SQ",Some(index.distance_type)),
    Index::BTree(_)        => ("BTREE",      None),
    Index::Bitmap(_)       => ("BITMAP",     None),
    Index::LabelList(_)    => ("LABEL_LIST", None),
    Index::FTS(fts)        => { /* 把 tokenizer_configs 摊进 body */ ("FTS", None) },  // table.rs:750-762
    Index::Auto            => { /* 拉 schema 按字段类型自动选 */ },                      // table.rs:763-783
    _ => return Err(Error::NotSupported { message: "Index type not supported".into() }),
};
body["index_type"] = serde_json::Value::String(index_type.into());
if let Some(distance_type) = distance_type {
    body["metric_type"] = serde_json::Value::String(distance_type.to_string().to_lowercase());  // table.rs:793-794
}
```

- **`Index::Auto`**（`table.rs:763`）：拉一次 `schema`，按字段类型自动选——向量列 → `IVF_PQ`(L2)，可建 BTree 的列 → `BTREE`，否则 `NotSupported`。
- **`metric_type` 转小写**（`table.rs:793`）：Phalanx 服务端当前要求距离类型小写。
- 注释 `table.rs:742` 坦言：**SaaS 暂不接收实际索引参数**（如 num_partitions），只发类型。

`list_indices`（`table.rs:911-966`）是个有趣的"两段式"：先 `POST /index/list/` 拿到名字 + 列，再**对每个索引调一次 `index_stats`** 拿 `index_type`。注释 `table.rs:945-946` 自己承认"有点低效，但这是拿到索引类型的唯一办法"。

### 4.6 七处 `NotSupported` 占位 + 与 `NativeTable` 对照

`RemoteTable` 里有一批"本地有、云端还没开"的方法，直接返回 `NotSupported` 或占位字符串：

| 方法 | 行号 | 表现 |
|---|---|---|
| `optimize` | `table.rs:827` | `NotSupported`（云端自动维护，不需用户触发） |
| `prewarm_index` | `table.rs:1006` | `NotSupported` |
| `table_definition` | `table.rs:1012` | `NotSupported` |
| `dataset_uri` | `table.rs:1017` | 返回字符串 `"NOT_SUPPORTED"` |

> **给贡献者的"加新能力"配方**（最高频改动）：要给 Cloud 接通某个现在还 `NotSupported` 的能力，典型四步——① 在 `table.rs` 对应方法构造 JSON/IPC body；② 拼 `POST /v1/table/{name}/新endpoint/`；③ 发送后过 `check_table_response`；④ **若服务端还不支持就先返回 `Error::NotSupported`**（模块里这 7 处占位可直接照抄格式）。别忘了配套写一个 mock 单测（见 §5.4）。

---

## 5. HTTP 客户端 `client.rs`：认证、重试、错误处理

### 5.1 `RestfulLanceDbClient`：电话机本体

`client.rs:159-165`：

```rust
#[derive(Clone, Debug)]
pub struct RestfulLanceDbClient<S: HttpSend = Sender> {  // client.rs:160
    client: reqwest::Client,         // ① 复用 reqwest，不自写 HTTP
    host: String,                    // ② 拼接好的主机地址
    retry_config: ResolvedRetryConfig, // ③ 解析后的重试配置
    sender: S,                       // ④ 话筒（真/假可换）
}
```

`try_new`（`client.rs:205-283`）做四件事：① 解析 `db://` URL 取 `db_name`（host）和 `db_prefix`（path，`client.rs:222-230`）；② 从配置或环境变量取三个超时；③ 建 `reqwest::Client` 并注入默认头；④ 拼 host。

host 默认值（`client.rs:271-273`）：

```rust
let host = match host_override {
    Some(host_override) => host_override,                                  // 企业版优先用 override
    None => format!("https://{}.{}.api.lancedb.com", db_name, region),    // client.rs:273
};
```

### 5.2 认证与路由头：`default_headers`

`default_headers`（`client.rs:291-366`）按情况注入一组头：

| 头 | 注入条件 | 行号 |
|---|---|---|
| `x-api-key` | 总是 | `client.rs:302` |
| `Host: {db_name}.local.api.lancedb.com` | `region == "local"` | `client.rs:307-315` |
| `x-lancedb-database` | 有 `host_override`（企业版） | `client.rs:316-323` |
| `x-lancedb-database-prefix` | URL path 非空 | `client.rs:324-334` |
| `x-azure-storage-account-name` | 选项里有 `account_name` / `azure_storage_account_name` | `client.rs:336-351` |

> **Rust 知识点：`HeaderName::from_static`**（`client.rs:16,302`）。在**编译期**构造 header 名常量，比运行时 `HeaderName::from_str` 更快且不会失败（`from_static` 要求传入的字面量已是合法小写 header 名，编译器替你保证）。

### 5.3 发送主流程与重试三计数器

`send`（`client.rs:378-424`）是所有请求的总入口：① 拆出 request；② 注入或读取 `x-request-id`（`client.rs:384-391`，重试时复用同一 id 便于服务端追踪）；③ debug 日志（JSON body 特殊打印，`client.rs:393-408`）；④ `with_retry` 为真走重试循环，否则单发。

重试循环 `send_with_retry_impl`（`client.rs:426-481`）是 `client.rs` 的灵魂：

```rust
loop {
    let request = req.try_clone().ok_or_else(|| Error::Runtime {       // ① 流式 body 不可克隆 → 报错
        message: "Attempted to retry a request that cannot be cloned".to_string(),  // client.rs:438-440
    })?;
    let response = self.sender.send(&client, request).await.map(|r| (r.status(), r));
    match response {
        Ok((status, response)) if status.is_success() => return Ok(...),          // 成功
        Ok((status, response)) if self.retry_config.statuses.contains(&status)    // 状态码可重试
            => retry_counter.increment_request_failures(source)?,                 // client.rs:459
        Err(err) if err.is_connect() => retry_counter.increment_connect_failures(err)?, // 连接失败  client.rs:461
        Err(err) if err.is_timeout() || err.is_body() || err.is_decode()
            => retry_counter.increment_read_failures(err)?,                       // 读失败  client.rs:464
        Err(err) => return Err(Error::Http { ... }),                              // 其他 → 直接抛
        Ok((_, response)) => return Ok(...),
    }
    let sleep_time = retry_counter.next_sleep_time();                             // client.rs:478
    tokio::time::sleep(sleep_time).await;
}
```

> **重试三分类 = 三种"打不通"的登记本**（理解 `client.rs` 的钥匙）：
> - **request 失败**（菜被退回）：状态码在可重试集合（默认 429/500/502/503，`client.rs:148`）。
> - **connect 失败**（电话拨不通）：`err.is_connect()`。
> - **read 失败**（对方说话听不清）：`is_timeout() / is_body() / is_decode()`。
>
> 三类**各自独立计数、各自有上限**（默认各 3 次，`client.rs:141-143`）。`check_out_of_retries`（`client.rs:523-546`）只要**任一类超限**就整体放弃，抛 `Error::Retry`。

退避算法 `next_sleep_time`（`client.rs:570-573`）：

```rust
let backoff = self.config.backoff_factor * (2.0f32.powi(self.request_failures as i32));  // 指数退避  client.rs:571
let jitter  = rand::random::<f32>() * self.config.backoff_jitter;                        // 随机抖动  client.rs:572
let sleep_time = Duration::from_secs_f32(backoff + jitter);
```

即 `退避 = backoff_factor * 2^已失败次数 + 随机抖动`（默认 factor=0.25、jitter=0.25）。指数退避避免雪崩，随机抖动避免大量客户端同步重试撞车。

### 5.4 `HttpSend` 话筒：真假可换的测试基石

`client.rs:167-186`：

```rust
pub trait HttpSend: Clone + Send + Sync + std::fmt::Debug + 'static {  // client.rs:167
    fn send(&self, client: &reqwest::Client, request: reqwest::Request)
        -> impl Future<Output = reqwest::Result<Response>> + Send;     // client.rs:168-172
}

pub struct Sender;                          // client.rs:177  生产实现（零大小）
impl HttpSend for Sender {
    async fn send(&self, client, request) -> reqwest::Result<Response> {
        client.execute(request).await       // client.rs:184  真的发出去
    }
}
```

测试侧 `MockSender`（`client.rs:615`）持一个 `Arc<dyn Fn(Request) -> Response>` 闭包当假服务器，`client_with_handler`（`client.rs:636`）造测试客户端。

> **Rust 陷阱：trait 里返回 `impl Future`（RPITIT）**（`client.rs:168-172`）。这里没写 `async fn`，而是手写返回 `impl Future<...> + Send`。原因是要**显式加 `+ Send` 约束**（保证返回的 future 能跨线程移动），而 trait 里的 `async fn` 当前无法直接标注这个约束。非 Rust 读者可理解为"这函数返回一个将来产值、且能跨线程搬运的对象"。这是 Rust 较新的特性，叫 RPITIT（Return Position Impl Trait In Trait）。

> **mock 测试范式**（新增 endpoint 必配）：所有单测用 `Connection::new_with_handler`（`connection.rs:918`）/ `Table::new_with_handler`（`table.rs:510`）传入一个**闭包当假服务器**，断言收到的请求 `method` / `path` / `body`，返回构造好的 `http::Response`。例如 `db.rs:493-506` 的 `test_table_names` 就断言"`GET /v1/table/`、无 query"，返回 `{"tables": [...]}`。这就是为什么换 `S` 泛型如此重要——**整条 HTTP 链路可以零网络地单测**。

错误处理收尾：`check_response`（`client.rs:483-501`）把非 2xx 响应读 body 拼成 `Error::Http`；`RequestResultExt::err_to_http`（`client.rs:589-606`）把 reqwest 的错误转成带 `request_id` 和 `status_code` 的 `Error::Http`。

---

## 6. 本地 vs 远程对照表：同一方法两种实现

把 09/10 篇的 `NativeTable` 和本篇的 `RemoteTable` 并排看，最能体会 `BaseTable` 抽象的威力——**上层一行不改，底层天壤之别**：

| `BaseTable` 方法 | `NativeTable`（本地后厨） | `RemoteTable`（云端后厨） |
|---|---|---|
| `add` | 拿写锁 → 交 lance 写盘 | `check_mutable` → IPC 打包 → `POST .../insert/`（`table.rs:526`） |
| `query` | 构造 lance scanner，本地执行 | 序列化成 JSON → `POST .../query/` → 把响应流包进 DataFusion（`table.rs:571`） |
| `count_rows` | lance 本地计数 | `POST .../count_rows/`，解析 JSON（`table.rs:495`） |
| `create_index` | 构造 lance `VectorIndexParams` 本地建 | 枚举映射成类型字符串 → `POST .../create_index/`（`table.rs:718`） |
| `update` | 返回真实行数 | `POST .../update/`，**写死返回 0**（`table.rs:683,703`） |
| `version` | 读本地 `Dataset` 版本 | `describe()` 打电话问（`table.rs:425`） |
| `checkout` | 切 `DatasetConsistencyWrapper` 模式 | 改 `RwLock<Option<u64>>`（`table.rs:428`） |
| `optimize` | 本地跑 compact/prune/index | `NotSupported`（`table.rs:827`） |
| `dataset_uri` | 真实 URI | 字符串 `"NOT_SUPPORTED"`（`table.rs:1017`） |
| 可变性守卫 | `DatasetConsistencyWrapper` 内部锁 | `check_mutable`（`table.rs:316`） |

> **核心洞见**：两列方法**签名完全相同**（都来自 `BaseTable` 合同），实现却一个进 lance、一个发 HTTP。这就是 09 篇反复强调的"一份合同两家后厨"在代码层面的全貌。要给 LanceDB 加一种新后端（比如另一个云服务），你只需再写一个 `impl BaseTable`，门面层 `Table` 和所有用户代码**一行都不用动**。

---

## 7. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `#[cfg(feature = "remote")]` | 编译期开关，feature 没开这段代码就不存在 | `lib.rs:206`、`connection.rs:805/836` |
| 泛型默认参数 `S: HttpSend = Sender` | 编译期依赖注入，默认填生产实现，测试换 mock | `db.rs:176`、`table.rs:50`、`client.rs:160` |
| trait 里返回 `impl Future`（RPITIT） | 为显式加 `+ Send` 约束而手写 future 返回 | `client.rs:167-172` |
| `RwLock<Option<u64>>` | 异步读写锁；`None`=可写最新，`Some`=只读快照 | `table.rs:56,317,441` |
| `moka::future::Cache` | 带 TTL 的异步并发缓存，操作都 `.await` | `db.rs:11,178,199` |
| `spawn_blocking(...).await.unwrap()?` | 把同步阻塞活儿丢到专用线程池；两层解包各管一事 | `db.rs:294` |
| `std::iter::from_fn` + `Mutex` | 闭包造流；`Mutex` 只为满足 `Sync` 而非并发 | `table.rs:112-125` |
| `TryFrom` / `try_into()?` | 可失败的类型转换，类比带校验的构造 | `client.rs:136,276`、`table.rs:1032` |
| `let ... else { return }` | 解构失败就提前返回 | `table.rs:503` |
| `serde_json::json!` 宏 | 内联构造 JSON 值，像动态对象一样赋值 | `table.rs:83,737` |
| `err.is_connect()/is_timeout()/...` | reqwest 错误细分，决定归到哪类重试计数 | `client.rs:461-464` |
| `HeaderName::from_static` | 编译期构造 header 名常量，快且不失败 | `client.rs:16,302`、`table.rs:47` |
| `pub(crate)` | 仅本 crate 内可见，封装实现细节 | `remote.rs:9-12` |

---

## 8. 动手验证（建议亲手做一遍）

1. **数占位**：在 `remote/table.rs` 里搜 `Error::NotSupported`，对照 §4.6 确认那 7 处"云端未开"的方法。再思考：为什么 `optimize` 在云端是 `NotSupported`？（提示：云端服务自己后台维护，不需用户触发。）
2. **跟一次 open_table**：从 `connection.rs:846` 的 `starts_with("db")` 出发，追到 `db.rs:357` 的 `open_table`，确认"缓存命中就不发 describe"这条路径。再看 `db.rs:388-391` 的 rename 如何搬缓存键。
3. **读懂重试**：在 `client.rs:434` 的 `loop` 里，对照 §5.3 把"成功 / request 失败 / connect 失败 / read 失败 / 其他错误"五个分支逐个找到行号，理解为什么三类失败要**各自独立计数**。
4. **抓 feature 开关**：临时把 `Cargo.toml` 的 `remote` feature 去掉编译（或看 `connection.rs:836` 的 `not(feature)` 版），确认 `connect("db://…")` 会得到 `Error::Runtime` 而非编译失败。
5. **照抄一个 mock 单测**：读 `db.rs:493-506` 的 `test_table_names`，仿照它给某个 endpoint 写一个 `Connection::new_with_handler` 测试，断言 `method` / `path`，体会"零网络测 HTTP"。

---

## 9. 小结 & 下一篇

- **远程"伪装"成本地**：`RemoteTable` 与 `NativeTable` 实现同一份合同 `BaseTable`（`table.rs:418` vs `table.rs:1886`），上层 `Table` 服务员只持 `Arc<dyn BaseTable>`，对底层完全无感。
- **三位主角**：`RemoteDatabase`（经理，`db.rs:176`，持电话机 + 熟客名册）、`RestfulLanceDbClient`（电话机，`client.rs:160`，管认证/重试/超时）、`RemoteTable`（遥控对讲机，`table.rs:50`，把每个方法翻译成 `POST /v1/table/…`）。
- **整个模块是翻译层**：不自写 HTTP（reqwest）、不自定义序列化（arrow_ipc）、不自做查询计划（DataFusion `OneShotExec`）——**组装而非重造**。
- **两个 Rust 模式贯穿全篇**：`S: HttpSend` 泛型默认参数实现编译期依赖注入（生产 `Sender` / 测试 `MockSender`），让整条 HTTP 链路可零网络单测；`ServerVersion::support_xxx()` 实现协议随版本平滑演进。
- **贡献入口**：要给 Cloud 接通新能力，在 `table.rs` 对应方法构造 body → 拼 endpoint → `check_table_response`，服务端未支持就先 `NotSupported` 占位，并配一个 `new_with_handler` mock 单测。

**下一篇** 将进入 `embedding` 体系，看 LanceDB 如何把"把文本/图片算成向量"这件事，做成可插拔的 `EmbeddingFunction` 与 `EmbeddingRegistry`——它正是 09 篇里 `Table` 第二个字段 `embedding_registry` 背后的世界。
