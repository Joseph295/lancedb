# 09 · Table 解剖（上）：契约、门面与本地实现

> **本篇解剖的源码**：`rust/lancedb/src/table.rs`（约 3750 行，全库最重的文件）
> **覆盖区间**：第 1–1519 行 —— 类型定义、`BaseTable` 契约、`Table` 门面、`NativeTable` 骨架与构造
> **留给下一篇（10）**：`DatasetConsistencyWrapper` 的锁与一致性机制、`merge`、时间旅行、`optimize` 的完整实现
>
> 读这篇前，建议先读完 `01 Rust 垫脚石`（尤其是 trait / `dyn` / `Arc` / async 四节）。本篇所有论断都带真实行号，形如 `table.rs:408`，可点击跳转。

---

## 0. 这一篇要回答的三个问题

如果你只带走三句话，就是这三句：

1. **用户手里的 `Table` 几乎不干活**——它是个"服务员"，把请求转交给后厨。
2. **后厨是谁，上层根本不关心**——本地表（`NativeTable`）和云端表（`RemoteTable`）签了同一份"用工合同"`BaseTable`，所以能无缝替换。
3. **很多方法不立刻执行**——它们先发给你一张"点菜单"（Builder），你勾完选项喊一声 `.execute()` 才下单。

把这三句话画成一张图，就是整个 Table 体系的骨架：

```
   你的代码
      │  table.add(batch).execute()
      ▼
 ┌─────────────────────┐
 │   Table  (门面/服务员) │  table.rs:499   ← 只持有一个指针，自己不存数据
 │   inner: Arc<dyn BaseTable>            │
 └─────────┬───────────┘
           │  委托 self.inner.xxx()
           ▼
 ┌─────────────────────┐
 │  BaseTable (trait/合同) │  table.rs:408   ← 29 个方法的接口，谁实现谁就是"表"
 └───┬─────────────┬───┘
     │             │
     ▼             ▼
┌──────────┐  ┌───────────┐
│NativeTable│  │RemoteTable│   ← 两种后厨，实现同一份合同
│(本地·lance)│  │(云端·HTTP) │
└─────┬────┘  └───────────┘
      │  self.dataset.get()/get_mut()
      ▼
   lance 引擎 (真正的存储/索引)
```

> **比喻总钥匙**：`BaseTable` 是一份**用工合同**（岗位职责清单），`Table` 是只负责传话的**服务员**，`NativeTable`/`RemoteTable` 是两家签了同一份合同的**后厨**。本篇就按"先读合同 → 再看服务员 → 最后进本地后厨"的顺序展开。

---

## 1. `BaseTable`：整个架构的"用工合同"

`table.rs:407-408`：

```rust
#[async_trait]
pub trait BaseTable: std::fmt::Display + std::fmt::Debug + Send + Sync {
    fn as_any(&self) -> &dyn std::any::Any;
    async fn schema(&self) -> Result<SchemaRef>;
    async fn add(&self, add: AddDataBuilder<NoData>, data: Box<dyn RecordBatchReader + Send>) -> Result<...>;
    // ……共 29 个方法
}
```

这是全库最重要的一段代码。**理解了它，你就理解了 LanceDB 为什么能"本地/云端一套代码"。**

### 1.1 先拆 `trait` 这一行的每个零件（写给不懂 Rust 的你）

| 零件 | 含义 | 类比你熟悉的语言 |
|---|---|---|
| `trait BaseTable` | 定义一个**接口**：一组方法签名，规定"想当表必须会做哪些事" | Java 的 `interface`、Go 的 `interface`、Python 的抽象基类 |
| `: ... + ... + ...`（冒号后） | **supertrait 约束**：实现 `BaseTable` 之前，必须先满足这几个 trait | "继承前提" / "你还得先实现这些" |
| `std::fmt::Display` | 能被 `{}` 格式化打印（给人看） | `toString()` |
| `std::fmt::Debug` | 能被 `{:?}` 打印（给程序员调试看） | 调试用的 `repr()` |
| `Send + Sync` | **能安全地跨线程传递/共享**——这是并发的"通行证" | "线程安全标记" |
| `#[async_trait]` | 一个宏，让 trait 里可以写 `async fn`（Rust 原生 trait 当年对异步方法支持有限，靠它补齐） | 无直接对应；理解成"让接口方法能 async 的胶水" |

> **为什么 `Send + Sync` 是关键？** 数据库要被多个并发请求同时使用。`Send + Sync` 是编译器层面的"通行证"：只有带证的类型才允许被塞进 `Arc` 跨线程共享。`BaseTable` 把它写进合同，意味着**任何表实现都必须是线程安全的**——这不是建议，是编译器强制。

### 1.2 完整的 29 个方法（经逐行核对，table.rs:408–494）

按职责分组列出。**行号是该方法在 `table.rs` 中的定义位置**：

**① 读 / 元数据**
| 行号 | 方法 | 作用 |
|---|---|---|
| `:409` | `as_any(&self) -> &dyn Any` | 取 `Any` 引用，给"向下转型"用（见 §2.4） |
| `:411` | `name(&self) -> &str` | 表名 |
| `:413` | `schema(&self) -> Result<SchemaRef>` | Arrow schema |
| `:415` | `count_rows(&self, Option<Filter>)` | 统计行数（可带过滤） |
| `:491` | `table_definition(&self)` | 表定义（列定义 + schema）。**注意：门面层 `Table` 没有同名转发方法，最容易被漏讲** |
| `:493` | `dataset_uri(&self) -> &str` | 底层数据集 URI（内部 API） |

**② 查询 / 执行计划**
| 行号 | 方法 | 作用 |
|---|---|---|
| `:418` | `create_plan(query, options)` | 把查询编译成 DataFusion `ExecutionPlan` |
| `:424` | `query(query, options)` | 执行查询，返回结果流 `DatasetRecordBatchStream` |
| `:430` | `explain_plan(query, verbose)` | **唯一带默认实现的方法**：内部调 `self.create_plan(...)` 再用 `DisplayableExecutionPlan::indent` 渲染成文本 |
| `:436` | `analyze_plan(query, options)` | 执行并附带运行时统计（源码此处无文档注释） |

**③ 写 / 改**
| 行号 | 方法 | 作用 |
|---|---|---|
| `:443` | `add(add, data)` | 写入数据 |
| `:449` | `delete(predicate)` | 按 SQL 谓词删除 |
| `:451` | `update(update)` | 更新，返回受影响行数 |
| `:463` | `merge_insert(params, new_data)` | upsert / insert-if-not-exists |

**④ 索引**
| 行号 | 方法 | 作用 |
|---|---|---|
| `:453` | `create_index(IndexBuilder)` | 建索引 |
| `:455` | `list_indices()` | 列出索引 |
| `:457` | `drop_index(name)` | 删索引 |
| `:459` | `prewarm_index(name)` | 预热（把索引提前载入内存） |
| `:461` | `index_stats(name)` | 索引统计 |

**⑤ Schema 演进**
| 行号 | 方法 | 作用 |
|---|---|---|
| `:469` | `add_columns(...)` | 加列 |
| `:477` | `alter_columns(...)` | 改列 |
| `:479` | `drop_columns(...)` | 删列 |

**⑥ 版本 / 时间旅行**（LanceDB 的招牌能力）
| 行号 | 方法 | 作用 |
|---|---|---|
| `:481` | `version()` | 当前版本号 |
| `:483` | `checkout(version)` | 切到某个历史版本（只读回溯） |
| `:485` | `checkout_latest()` | 切回最新版 |
| `:487` | `restore()` | 把当前 checkout 的版本"扶正"成最新版 |
| `:489` | `list_versions()` | 列出所有版本 |

**⑦ 维护**
| 行号 | 方法 | 作用 |
|---|---|---|
| `:469` | `optimize(action)` | 压缩 / 清理 / 增量建索引（见 §3.3） |

> **划重点**：29 个方法里，**只有 `explain_plan`（:430）有方法体（默认实现）**，其余 28 个都是抽象方法——"合同只写职责，具体怎么做由后厨决定"。这正是 trait 作为"接口"的本质。

### 1.3 一份合同，两家后厨

`table.rs:408` 上方的文档注释点明：`BaseTable` 同时覆盖**本地原生表**（对应一个 lance `Dataset`）和**远程表**（对应 LanceDB Cloud）。

```
        BaseTable (合同)
        ╱            ╲
 NativeTable      RemoteTable
 (table.rs:1178)  (remote/table.rs)
 调本地 lance      发 HTTP 请求
```

上层代码永远只跟"实现了 `BaseTable` 的某个东西"打交道，从不写 `if 本地 else 云端`。**要把本地表换成云表，只需换掉那个躲在 `Arc<dyn BaseTable>` 后面的实现**，门面层一行都不用改。这就是 trait + `dyn` 多态在数据库工程里的威力。

---

## 2. `Table`：你手里的"服务员"

`table.rs:499-503`：

```rust
#[derive(Clone)]
pub struct Table {
    inner: Arc<dyn BaseTable>,
    embedding_registry: Arc<dyn EmbeddingRegistry>,
}
```

整个结构体**只有两个字段，且都是 `Arc<dyn ...>` 指针**。`Table` 自己不存任何数据——它只是个握着后厨电话的服务员。

### 2.1 为什么 `clone` 一张表几乎不要钱

`#[derive(Clone)]` 让 `Table` 可被克隆。但因为两个字段都是 `Arc`（原子引用计数智能指针），**克隆只是把引用计数 +1，并不复制底层表数据**。

> **比喻**：`Arc<dyn BaseTable>` 像一把"万能遥控器壳"。`clone` 是再印一张一模一样的遥控器，但它们背后控制的是**同一台电视**。印 100 张遥控器，电视还是那一台。

你会在源码里看到铺天盖地的 `self.inner.clone()`——读到时心里要清楚：这是廉价的指针复制，不是搬数据。

### 2.2 两类方法：立即委托 vs 惰性 Builder

`Table` 的方法泾渭分明地分成两类，**分清它们是读懂这个文件的关键**：

#### (a) 立即委托型 —— `async`，调用即转发

例如 `count_rows`（`table.rs:609`）：

```rust
pub async fn count_rows(&self, filter: Option<String>) -> Result<usize> {
    self.inner.count_rows(filter.map(Filter::Sql)).await
}
```

方法体就一行：把参数适配一下（`filter.map(Filter::Sql)` 把 `Option<String>` 包成 `Option<Filter>`），然后**原样转交给 `self.inner`** 并 `.await`。

属于这一类的还有：`schema`(:600)、`delete`(:696)、`optimize`(:980)、`version`(:1009)、`checkout`、`list_indices`(:1059)、`dataset_uri`(:1066)、`index_stats`(:1072)…… 它们读起来都"空空如也"，因为活儿全在后厨。

> **核对修正**：`list_indices`(:1059)、`dataset_uri`(:1066)、`index_stats`(:1072) 在源码里是**交错排列**的，`dataset_uri` 夹在中间，并非"索引四件套连续一块"。读源码时别被顺序误导。

#### (b) Builder 工厂型 —— 不是 `async`，只发"点菜单"

例如 `add`（`table.rs:619`）：

```rust
pub fn add<T: IntoArrow>(&self, batches: T) -> AddDataBuilder<T> {
    AddDataBuilder {
        parent: self.inner.clone(),                 // 把后厨电话塞进菜单
        data: batches,
        mode: AddDataMode::Append,                  // 默认追加
        write_options: WriteOptions::default(),
        embedding_registry: Some(self.embedding_registry.clone()),
    }
}
```

注意：**它不是 `async`，也不碰任何 IO**。它只是 `new` 出一个 `AddDataBuilder` 返回给你。真正写入要等你在这个 builder 上链式配置完、调用 `.execute()`（见 §3.1）。

属于这一类的：`add`(:619)、`update`(:643)、`create_index`(:761)、`merge_insert`(:847)、`query`(:941)。

> **为什么要搞 Builder？** Rust 没有"可选参数 / 默认参数 / 函数重载"。要表达"加数据时可以选追加还是覆盖、可以配写选项、也可以都不配"，最地道的方式就是 Builder：先返回一个可继续填的对象，填完再执行。
>
> **比喻**：`Table::add` 给你一张**点菜单**（`AddDataBuilder`），你可以在上面继续勾"要追加还是覆盖"，最后喊一声 `execute()` 才把单子送进厨房。

### 2.3 `Display` 也是"甩锅"给后厨

`table.rs:551-555`：

```rust
impl std::fmt::Display for Table {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.inner)   // 打印请求直接转交 inner
    }
}
```

`write!(f, "{}", self.inner)` 里的 `{}` 会触发 `inner` 的 `Display`。**这行能编译通过，正是因为 §1.1 里 `BaseTable` 被约束实现了 `std::fmt::Display`**——supertrait 约束在这里兑现了价值。（顺带一提：`Table` 只 `derive(Clone)`，**没有 `Debug`**。）

### 2.4 `as_native` 与"拆快递"——`dyn` 想变回具体类型

有时上层确实需要拿到具体的 `NativeTable`（比如调用一些只有本地表才有的方法）。但手里是 `Arc<dyn BaseTable>`，怎么"拆开看里面到底是不是 NativeTable"？

答案是 Rust 的运行时**向下转型**，机制藏在一个扩展 trait 里（`table.rs:1165-1174`）：

```rust
pub trait NativeTableExt {
    fn as_native(&self) -> Option<&NativeTable>;
}

impl NativeTableExt for Arc<dyn BaseTable> {
    fn as_native(&self) -> Option<&NativeTable> {
        self.as_any().downcast_ref::<NativeTable>()
        // ① as_any(): 经 BaseTable 合同的 :409 方法，拿到 &dyn Any
        // ② downcast_ref::<NativeTable>(): 试着把它转回 NativeTable，是则 Some，不是则 None
    }
}
```

`Table::as_native`（`table.rs:583`）就是转手委托给它。

> **两个 Rust 知识点**：
> - **扩展 trait（extension trait）**：给一个你"不拥有"的类型（这里是标准库的 `Arc<dyn BaseTable>`）后加方法的惯用法——定义一个新 trait 再为那个类型 `impl`。
> - **`as_any` + `downcast_ref`**：trait 对象（`dyn`）想变回具体类型的**唯一安全途径**。`downcast_ref` 返回 `Option`，转型失败给你 `None` 而不是崩溃。
>
> **比喻**：手里是个只写着"内含一张表"的通用快递箱（`dyn BaseTable`），`as_native` 就是拆开验货——是 `NativeTable` 这件具体商品就给你，不是就告诉你 `None`。
>
> 源码注释已标注 `as_native` 是**临时 API，将来会移除**——读到带这种注释的代码，贡献时要留意别在上面盖新楼。

---

## 3. 支撑类型：Builder 与配置项

`Table` 的 Builder 工厂方法返回的那些"点菜单"，定义在文件前段（`table.rs:88-401`）。挑最能说明设计的讲。

### 3.1 `AddDataBuilder` —— "消费自身"的链式 Builder

`table.rs:280-286`：

```rust
pub struct AddDataBuilder<T: IntoArrow> {
    parent: Arc<dyn BaseTable>,                          // 目标表（后厨电话）
    pub(crate) data: T,                                 // 待写入数据，泛型
    pub(crate) mode: AddDataMode,                       // 追加 / 覆盖
    pub(crate) write_options: WriteOptions,
    embedding_registry: Option<Arc<dyn EmbeddingRegistry>>,
}
```

它的链式方法长这样（`table.rs:299`）：

```rust
pub fn mode(mut self, mode: AddDataMode) -> Self {   // 注意：mut self，不是 &self
    self.mode = mode;
    self
}
```

> **Rust 陷阱：`mut self`（消费自身）**。这个方法接收的是 `self` 的**所有权**（而非借用 `&self`），改完字段后把 `self` 整个**还回去**（返回 `Self`）。这就实现了 `a.mode(x).write_options(y).execute()` 的链式调用。代价是：调用一次后，**原来的变量就被"移动"走了，不能再用**。这是 Rust builder 的标准形态。

最后 `execute`（`table.rs:309`）才真正干活：

```rust
pub async fn execute(self) -> Result<()> {
    let parent = self.parent.clone();                       // ① 克隆一份后厨电话
    let data = self.data.into_arrow()?;                     // ② 把泛型数据转成 Arrow（IntoArrow trait），出错就 ? 传播
    let without_data = AddDataBuilder::<NoData> { ... };    // ③ 造一个"去掉数据"的 builder
    parent.add(without_data, data).await                   // ④ 委托给 BaseTable::add（:443）真正写入
}
```

> **核对要点**：第 ① 步 `clone` 出的 `parent` 和第 ③ 步移进 `without_data` 的原 `self.parent`，是**同一个 `Arc` 的两个引用计数**，不是两张表。别被两个变量名误导。
>
> 这里也回答了"泛型数据怎么落地"：`execute` 先 `into_arrow()` 把任意 `T` 具象成 Arrow 数据，再把"无数据的配置 + Arrow 数据"一起交给合同方法 `add`。

### 3.2 `TableDefinition` —— 给 schema "贴标签"

`table.rs:102-106`：

```rust
#[derive(Debug, Clone)]
pub struct TableDefinition {
    pub column_definitions: Vec<ColumnDefinition>,   // 每列的来源（物理列 or embedding 列）
    pub schema: SchemaRef,                           // Arrow schema
}
```

`ColumnDefinition` 目前只记一个 `kind: ColumnKind`（`table.rs:87-93`），而 `ColumnKind` 区分两种列：

- `Physical`：用户直接写入的普通列（最常见）
- `Embedding(EmbeddingDefinition)`：由 embedding 函数算出来的列，携带"怎么算"的定义

最妙的是它和 Arrow schema 的**双向序列化**（`table.rs:127` / `:147`）：

```
列定义 ──into_rich_schema()──▶ 塞进 schema.metadata["lancedb::column_definitions"]（serde_json 序列化）
                              ◀──try_from_rich_schema()── 从 metadata 读回来反序列化
```

> **比喻**：富 schema 序列化像给行李箱**贴标签**。把"哪几列是 embedding 列、怎么生成"这类额外说明，写进 Arrow schema 的 metadata。这样表数据搬到哪（持久化到磁盘、跨进程传递），它的"使用说明书"都跟着走，不会丢。
>
> 细节：`into_rich_schema`（:150）用 `serde_json::to_string(...).unwrap()` ——这里敢 `unwrap`（出错即 panic）是因为代码注释判断"序列化列定义不可能失败，除非有 bug"。`try_from_rich_schema` 反过来用 `Result`，因为读外部数据可能格式不对。

### 3.3 `OptimizeAction` —— 一个枚举统一三类维护

`table.rs:165-221`：

```rust
pub enum OptimizeAction {
    All,                                          // 默认：全跑一遍
    Compact { options, remap_options },           // 合并小文件
    Prune { older_than, delete_unverified, ... }, // 删除旧版本，腾磁盘
    Index(OptimizeOptions),                       // 把新数据增量并入已有索引
}
```

> **为什么数据库需要"优化"这件事？** LanceDB 用的是**只读、追加式存储**：每次写入都产生新文件，每次"删除/更新"都产生新版本（旧版本仍在磁盘上）。好处是天然支持时间旅行、并发读不加锁；代价是日积月累，小文件多、旧版本占空间。于是需要这三类"大扫除"：
> - `Compact` = 把一堆零散小盒子并成大箱子（加快读）
> - `Prune` = 扔掉过期旧物腾空间（类比 PostgreSQL 的 `VACUUM`）
> - `Index` = 把新进的东西归类进现有收纳系统，而不重新整理整个房间
>
> 默认 `All` 三件一起做。`OptimizeAction` 本身**没有 derive 任何 trait**（注意它和别的类型不同），但有手写的 `Default`（:223）返回 `All`。

### 3.4 一个值得注意的"半成品"：`BadVectorHandling`

`table.rs:240-254` 有个非 `pub` 的内部枚举 `BadVectorHandling`，描述"向量含 NaN 或长度不对时怎么办"（报错 / 丢弃 / 填充 / 置 NULL）。但**除 `Error` 外的所有变体都标了 `#[allow(dead_code)]` 并指向 issue #992**，`WriteOptions` 里对应的 `on_bad_vectors` 字段也被注释掉了。

> **给贡献者的信号**：这是一处**尚未启用的占位功能**。`#[allow(dead_code)]`（压制"未使用代码"警告）+ 指向 issue 的注释，是 Rust 项目里"功能挖了坑但还没填"的典型标志。如果你想找 good-first-issue，这种地方常常就是切入点——但动手前先去 issue #992 看上下文。

---

## 4. `NativeTable`：真正的本地后厨

`table.rs:1177-1185`：

```rust
#[derive(Debug, Clone)]
pub struct NativeTable {
    name: String,                                       // 表名
    uri: String,                                        // 表所在 uri
    pub(crate) dataset: dataset::DatasetConsistencyWrapper,  // ★核心：包着 lance Dataset
    read_consistency_interval: Option<std::time::Duration>,  // 读一致性刷新间隔
}
```

`NativeTable` 也**薄得惊人**——只有 4 个字段。所有真正的存储能力都在 `dataset` 这一个字段里，`NativeTable` 同样只是一层门面，把 lance `Dataset` 包了一下。

### 4.1 `dataset` 字段：不是裸 `Dataset`，是带锁的包装器

注意 `dataset` 的类型不是 lance 的 `Dataset`，而是 `DatasetConsistencyWrapper`（定义在 `table/dataset.rs:19`，**详细机制留到下一篇 10**）。它内部是 `Arc<RwLock<DatasetRef>>`——一把读写锁。

铁律是：
- **任何读操作**走 `self.dataset.get().await?`（拿读锁 → `DatasetReadGuard`）
- **任何写操作**走 `self.dataset.get_mut().await?`（拿写锁 → `DatasetWriteGuard`）

> **Rust 知识点：内部可变性**。你会看到 `create_index`、`merge` 这些"会改数据"的方法，有的签名却是 `&self`（不可变借用）。怎么还能改？因为真正的可变性被藏在 `DatasetConsistencyWrapper` 内部的 `RwLock` 里——这叫**内部可变性（interior mutability）**：对外是 `&self`，对内通过锁拿到可变访问。这是 Rust 并发设计的核心套路之一。
>
> **比喻**：`NativeTable` 像银行**柜台**，自己不存钱；`dataset`（lance `Dataset`）才是**金库**；而 `DatasetConsistencyWrapper` 是金库门口那位**保安**，拿着读锁/写锁的钥匙，控制谁能进、什么时候进。

### 4.2 四个构造函数，同一套路

`NativeTable` 的固有方法块（`impl NativeTable`，`table.rs:1206` 起）开头是构造函数，全都遵循同一个模式：

```
用 lance 加载/创建底层 Dataset  ──▶  DatasetConsistencyWrapper::new_latest(...) 包起来  ──▶  组装成 NativeTable
```

| 构造函数 | 行号 | 干什么 |
|---|---|---|
| `open(uri)` | `:1217` | 最简入口：从 uri 推断表名，转调 `open_with_params(uri, &name, None, None, None)` |
| `open_with_params(...)` | `:1233` | 完整打开逻辑（下面详解） |
| `create(...)` | `:1295` | 用数据流创建新表 |
| `create_empty(...)` | `:1332` | 只建 schema、无数据行——内部造一个空 `RecordBatchIterator` 再转调 `create` |

以 `open_with_params`（`table.rs:1233`）为例，看清"委托给 lance + 翻译错误"这两个套路：

```rust
let params = params.unwrap_or_default();                            // ① 可选参数取默认（Rust 表达"默认参数"的惯用法）
if let Some(wrapper) = write_store_wrapper {
    params = params.patch_with_store_wrapper(wrapper)?;             // ② 可选地给存储打补丁
}
let dataset = DatasetBuilder::from_uri(uri)                         // ③ 委托给 lance 加载底层 Dataset
    .with_read_params(params)
    .load()
    .await
    .map_err(|e| match e {                                         // ④ 把 lance 错误翻译成 LanceDB 错误
        lance::Error::DatasetNotFound { .. } => Error::TableNotFound { .. },
        e => Error::Lance { source: e },
    })?;
let dataset = DatasetConsistencyWrapper::new_latest(dataset, read_consistency_interval);  // ⑤ 包上保安
Ok(Self { name, uri, dataset, read_consistency_interval })          // ⑥ 组装
```

> **套路一：错误翻译**。底层 lance 说的是 `Dataset` 的行话（`DatasetNotFound`、`DatasetAlreadyExists`），LanceDB 在边界上把它翻译成用户能懂的 `TableNotFound`、`TableAlreadyExists`，其余统一兜底成 `Error::Lance`。**比喻**：前台把后厨的行话翻译成顾客能听懂的话。`create`（:1295）里也是同样的翻译（`DatasetAlreadyExists → TableAlreadyExists`）。
>
> **套路二：一切交给 lance**。`open` 委托 `DatasetBuilder`，`create` 委托 `InsertBuilder::new(uri).with_params(&params).execute_stream(batches)`（:1313）。**LanceDB 自己不写存储代码，它编排 lance。** 这就是 README 里那句"组装而非重造"的字面证据。

### 4.3 操作方法也只是"拿锁 + 转交 lance"

构造函数之后是一堆操作方法，形态高度一致。看两个典型：

```rust
// 只读统计：拿读锁，转交 lance（table.rs:1419）
pub async fn count_fragments(&self) -> Result<usize> {
    Ok(self.dataset.get().await?.count_fragments())
}

// 建索引：拿写锁，转交 lance（table.rs:1446 create_ivf_flat_index 节选）
async fn create_ivf_flat_index(&self, index: IvfFlatIndexBuilder, field: &Field, replace: bool) -> Result<()> {
    if !supported_vector_data_type(field.data_type()) {            // ① 先校验列类型
        return Err(Error::InvalidInput { .. });
    }
    let num_partitions = index.num_partitions.unwrap_or_else(|| {  // ② 缺省值用启发式推断
        suggested_num_partitions(self.count_rows(None).await?)
    });
    let mut dataset = self.dataset.get_mut().await?;              // ③ 拿写锁
    let params = VectorIndexParams::ivf_flat(num_partitions, distance_type);  // ④ 构造 lance 索引参数
    dataset.create_index(&[field.name()], IndexType::Vector, None, &params, replace).await  // ⑤ 转交 lance
}
```

读懂这个模式，你就读懂了 `NativeTable` 后半部分几乎所有方法：**校验 → 拿锁 → 构造 lance 参数 → 委托 lance**。

> **注意**：真正的 `impl BaseTable for NativeTable`（即兑现 §1 那份合同的地方）从 `table.rs:1886` 开始，**不在本篇区间**——本篇看的是 `NativeTable` 的固有方法（构造 + 辅助），下一篇再看它如何逐条兑现 `BaseTable` 合同。

---

## 5. 串起来：一次 `table.add(batch).execute()` 的完整调用栈

把本篇所有零件拼成一条链路（本地表场景）：

```
你的代码:  table.add(batch).execute().await
   │
   │  ① Table::add  (table.rs:619)  —— 不是 async，只造菜单
   ▼
AddDataBuilder { parent: self.inner.clone(), data: batch, mode: Append, ... }
   │
   │  （你可链式 .mode(Overwrite) / .write_options(...)）
   │  ② AddDataBuilder::execute  (table.rs:309)  —— async，真正下单
   ▼
   ├─ self.data.into_arrow()?           把泛型数据 → Arrow RecordBatch
   └─ parent.add(without_data, data)    委托给 BaseTable 合同的 add (:443)
        │
        │  ③ 动态分发：parent 实际指向 NativeTable
        ▼
   impl BaseTable for NativeTable::add  (table.rs:1886+，下一篇详解)
        │
        │  ④ self.dataset.get_mut().await?   经"保安"拿写锁
        ▼
   lance Dataset 写入  → 落盘，产生新版本
```

四步里，**前两步在 `Table`/Builder 层（本篇 §2、§3），第三步靠 `BaseTable` 合同 + `dyn` 动态分发选中后厨（§1），第四步进 `NativeTable` 拿锁后交给 lance（§4）**。一条链路把本篇四个主角全串上了。

---

## 6. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `trait T: A + B` | 接口 + supertrait 约束（实现前必须先满足 A、B） | `BaseTable` :408 |
| `#[async_trait]` | 让 trait 能写 `async fn` 的宏 | :407 |
| `Send + Sync` | 跨线程安全的"通行证"，编译器强制 | :408 |
| `Arc<dyn T>` | 引用计数 + 运行时多态；`clone` 只 +1 计数，不搬数据 | 全篇 |
| `#[derive(...)]` | 自动生成 trait 实现（`Clone`/`Debug`/`Serialize`…） | :499 等 |
| `mut self -> Self` | 消费自身的链式 builder；调用后原变量被移动 | `AddDataBuilder::mode` :299 |
| `?` 运算符 | 出错就提前 `return` 这个错误 | `into_arrow()?` 等 |
| `Option<T>` + `unwrap_or_default()` | Rust 表达"可选/默认参数"的惯用法 | `open_with_params` :1240 |
| `.map_err(|e| match ...)` | 把一种错误翻译成另一种 | :1251 |
| `as_any()` + `downcast_ref` | `dyn` 变回具体类型的唯一安全途径 | `NativeTableExt` :1172 |
| 内部可变性（`RwLock`） | `&self` 也能改数据，可变性藏在锁里 | `dataset` 字段 |
| `pub(crate)` | 仅本 crate 内可见 | `dataset`、`multi_vector_plan` 等 |

---

## 7. 动手验证（建议亲手做一遍）

1. **数一数合同**：打开 `table.rs`，跳到 408 行，对照 §1.2 的表格，确认 `BaseTable` 确实是 29 个方法、且只有 `explain_plan` 有方法体。问自己：为什么 `table_definition`（:491）在 `Table` 门面层找不到同名方法？（提示：它是给内部/绑定层用的。）
2. **抓"委托"**：在 `table.rs` 里搜索 `self.inner.`，统计有多少个方法只是一行转发。再搜 `self.inner.clone()`，体会 Builder 工厂方法的共性。
3. **跟一次写入**：从 `Table::add`（:619）开始,手动追到 `AddDataBuilder::execute`（:309），再到 `NativeTable` 的 `add` 实现（搜 `impl BaseTable for NativeTable`，约 :1886）。把 §5 那张调用栈在源码里逐跳验证一遍。
4. **找半成品**：跳到 `BadVectorHandling`（:240），点开 issue #992，理解"挖坑未填"的功能长什么样——这是你将来贡献的潜在落点。

---

## 8. 小结 & 下一篇

- `Table` 是**门面/服务员**：只持有 `Arc<dyn BaseTable>`，方法非"立即委托"即"返回 Builder"。
- `BaseTable` 是**用工合同**：29 个方法的 trait，让本地表与云表对上层透明；`Send + Sync` 强制线程安全。
- `NativeTable` 是**本地后厨**：4 个字段的薄壳，核心是被"保安"`DatasetConsistencyWrapper` 包着的 lance `Dataset`；构造与操作都是"拿锁 + 委托 lance + 翻译错误"。

**下一篇 `10 Table 解剖（下）`** 将钻进本篇刻意留白的部分：
- `DatasetConsistencyWrapper` 的读写锁与"最新版 / 时间旅行"双模式到底怎么实现一致性
- `impl BaseTable for NativeTable`（:1886+）如何逐条兑现合同
- `merge_insert` 的 upsert 语义、`checkout`/`restore` 的时间旅行机制、`optimize` 三件套的完整实现
