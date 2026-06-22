# 10 · Table 解剖（下）：金库保安、时间旅行与后厨实现

> **本篇解剖的源码**：
> - `rust/lancedb/src/table/dataset.rs`（337 行，全文）—— 并发与一致性核心
> - `rust/lancedb/src/table.rs:1886-2433` —— `impl BaseTable for NativeTable`（兑现合同的本地后厨）
> - `rust/lancedb/src/table/merge.rs`（93 行，全文）—— upsert 点菜单
> - `rust/lancedb/src/table/datafusion.rs:25-205` —— 把表桥接成 DataFusion `TableProvider`
>
> **覆盖区间**：上篇（09）刻意留白的全部内容——锁机制、双模式状态机、时间旅行五方法、`merge_insert`、`optimize` 三件套、schema 演进、DataFusion 桥接。
>
> **前置阅读**：必须先读 `09 Table 解剖（上）`（本篇直接承接它的比喻与结构），并建议复习 `01 Rust 垫脚石` 的 `Arc`/`RwLock`/`Deref`/async 四节、`04 写入链路`、`06 查询链路`。
>
> **与哪些篇互补**：链路篇（04/05/06）讲"请求一跳跳到下一跳"；本篇讲"`NativeTable` 这个模块内部到底怎么搭、并发安全怎么保证"。读完上篇你知道了"合同长什么样"，读完本篇你知道"本地后厨怎么逐条履约"。

---

## 0. 本篇要回答的问题

上篇结尾留了三个坑，本篇逐一填上：

1. **`DatasetConsistencyWrapper`（那位"金库保安"）内部到底怎么搭的？** 它怎么做到"对外是 `&self` 也能改数据"，又怎么决定"什么时候去远端刷新一次"？
2. **时间旅行五方法（`version`/`checkout`/`checkout_latest`/`restore`/`list_versions`）怎么兑现？** 为什么"钉在历史版本"时不能写，`restore` 又凭什么是唯一的例外？
3. **`merge_insert`/`optimize`/schema 演进这些"重活儿"，`NativeTable` 自己写了多少？** 答案依然是：**几乎没写，全在编排 lance**。

一句话锚点：**`DatasetConsistencyWrapper` 是金库门口的保安——它不数钱（不碰数据），只管两件事：金库该不该开门盘点了（一致性刷新）、你有没有权限进去改东西（Latest 才放行写，TimeTravel 一律拒绝）。** 真正的增删改查全在金库里那台 lance `Dataset`。

把这一篇的主角画成一张图：

```
              NativeTable  (table.rs:1178, 上篇 §4)
                   │  pub(crate) dataset
                   ▼
   ┌──────────────────────────────────────────┐
   │  DatasetConsistencyWrapper (dataset.rs:19) │  ← 金库保安（元组结构体，只有 .0 一个字段）
   │      .0 : Arc<RwLock<DatasetRef>>          │
   └───────────────────┬────────────────────────┘
                       │  锁里包着一个双变体枚举
                       ▼
       ┌────────────────────────────────┐
       │  DatasetRef (dataset.rs:26)     │  ← 金库的两种营业模式
       │  ┌──────────┐   ┌─────────────┐ │
       │  │  Latest  │   │ TimeTravel  │ │
       │  │ 实时营业  │   │  历史档案室  │ │
       │  │(追最新版) │   │(钉死某版只读)│ │
       │  └────┬─────┘   └──────┬──────┘ │
       └───────┼────────────────┼────────┘
               ▼                ▼
          get() → DatasetReadGuard   (只读卡，Deref → &Dataset)
          get_mut() → DatasetWriteGuard (写卡，DerefMut → &mut Dataset)
               │
               ▼
        lance::Dataset  (真正的金库：存储/索引/版本)
```

> **比喻总钥匙（延续上篇）**：`NativeTable`=银行柜台，`DatasetConsistencyWrapper`=金库保安，`DatasetRef` 双变体=金库的两种营业模式，读写守卫=刷进门的读卡器/写卡器，lance `Dataset`=金库本体。本篇就按"保安怎么站岗（§1-4）→ 后厨怎么履约（§5-8）"展开。

---

## 1. `DatasetConsistencyWrapper`：金库保安的内部构造

`dataset.rs:18-19`：

```rust
#[derive(Debug, Clone)]
pub struct DatasetConsistencyWrapper(Arc<RwLock<DatasetRef>>);
```

整个保安**只有一个字段**——一把"读写锁里包着双模式枚举"。注释（`dataset.rs:15-17`）写得很直白：可以廉价克隆、支持"并发读或独占写"。

### 1.1 拆解这一行的每个零件（写给不懂 Rust 的你）

| 零件 | 含义 | 类比 |
|---|---|---|
| `struct X(Y)` | **元组结构体**：没有字段名，用 `self.0` 访问那个唯一字段 | Python 里一个单元素的具名包装类，但更轻 |
| `Arc<...>` | 原子引用计数指针；`clone` 只让计数 +1，不复制内容 | "万能遥控器壳"（上篇 §2.1） |
| `RwLock<...>` | 读写锁：允许多个读者**同时**进，但写者必须**独占** | 阅览室——很多人可同时看书，但要改书必须清场 |
| `DatasetRef` | 锁保护的"真身"：一个有两个变体的枚举（见 §2） | 金库的当前营业状态 |
| `#[derive(Clone)]` | 自动实现克隆 | clone 只是 `Arc` 计数 +1，故注释说"cloned cheaply" |

> **Rust 陷阱：元组结构体 `self.0`**。`DatasetConsistencyWrapper` 没有 `.dataset`/`.lock` 这样的字段名，访问唯一字段写作 `self.0`。源码里 `self.0.read().await`、`self.0.write().await` 满地都是——读到 `.0` 别懵，那就是"那把锁"。

> **Rust 陷阱：`tokio::sync::RwLock` 不是普通锁**（导入在 `dataset.rs:11`）。它是**异步锁**：`.read()` / `.write()` 都返回一个 `Future`，必须 `.await`。为什么用异步版而非 `std::sync::RwLock`？因为持锁期间可能跨越 `await`（比如持读锁时还要 `.await` 一次远端查询），异步锁在等待时**让出线程**而不是死等阻塞。非 Rust 读者把它理解成"会礼让的锁"即可。

### 1.2 保安的方法清单

`impl DatasetConsistencyWrapper`（`dataset.rs:126-253`）一共这些方法，按职责分组：

| 方法 | 行号 | 公开性 | 职责 |
|---|---|---|---|
| `new_latest` | `:128` | pub | 构造一个 Latest 模式的保安（上篇构造函数调它） |
| `get` | `:137` | pub | 拿**读锁** → `DatasetReadGuard`（先做一致性检查） |
| `get_mut` | `:147` | pub | 拿**写锁** → `DatasetWriteGuard`（先验"可写" + 一致性） |
| `get_mut_unchecked` | `:157` | pub | 拿写锁但**跳过"可写"校验**——专供 `restore` 用 |
| `as_latest` | `:165` | pub | 切到 Latest 模式（双重检查） |
| `as_time_travel` | `:178` | pub | 切到 TimeTravel 模式（钉死某版本） |
| `set_latest` | `:186` | pub | 写操作完成后，把"新版本 Dataset"塞回去 |
| `reload` | `:190` | pub | 刷新到最新版（双重检查锁升级） |
| `time_travel_version` | `:206` | pub | 当前若钉在某版本，返回该版本号，否则 `None` |
| `ensure_mutable` | `:210` | pub | 守门：TimeTravel 模式直接报错 |
| `is_up_to_date` | `:221` | 私有 | 判定当前是否还在一致性窗口内 |
| `ensure_up_to_date` | `:247` | 私有 | 不新鲜就触发 `reload` |

> **教学锚点：保安职责单一**。通读这张表你会发现——**没有一个方法在做增删改查**。它们全部围绕三个问题打转：①现在该不该 reload？②谁有权写？③写完的新版本是谁？这就是"组装而非重造"最纯粹的体现：一致性与并发由 lancedb 管，数据操作全部下沉到 lance。

---

## 2. 两种营业模式：`DatasetRef` 双变体状态机

保安锁里包着的 `DatasetRef`（`dataset.rs:26-35`）是整个时间旅行的骨架：

```rust
#[derive(Debug, Clone)]
enum DatasetRef {
    /// In this mode, the dataset is always the latest version.
    Latest {
        dataset: Dataset,                               // ① lance 数据集本体
        read_consistency_interval: Option<Duration>,   // ② 多久去远端盘一次账（None=永不主动盘）
        last_consistency_check: Option<time::Instant>, // ③ 上次盘账的时刻
    },
    /// In this mode, the dataset is a specific version. It cannot be mutated.
    TimeTravel { dataset: Dataset, version: u64 },      // 钉死在 version 这个历史快照，语义上不可变
}
```

> **比喻：金库的两种营业模式**。`Latest` 是"实时营业"——每隔 `read_consistency_interval` 就去后台盘一次最新账目（防止别的进程偷偷改了数据你还蒙在鼓里）。`TimeTravel` 是"历史档案室"——钉死某个历史快照，只许看不许改。

> **Rust 知识点：枚举的"带数据变体"**。Rust 的 `enum` 不只是 C 里的整数常量，每个变体能**携带自己的字段**（这叫 tagged union / 代数数据类型）。`Latest` 带三个字段、`TimeTravel` 带两个，编译器强制你用 `match` 把两种情况都处理掉——这正是下面所有方法满是 `match self { ... }` 的原因。非 Rust 读者可类比：一个"要么是 A 形态、要么是 B 形态"的对象，且取值时必须显式分支。

### 2.1 状态转换全集中在两个方法

整个状态机只有两条迁移边，都定义在 `impl DatasetRef`（`dataset.rs:37-124`）里：

```
        as_time_travel(v)  (dataset.rs:86)
   Latest ───────────────────────────────▶ TimeTravel{version:v}
   实时营业 ◀───────────────────────────────  历史档案室
        as_latest()  (dataset.rs:69)
```

- **`as_time_travel`（`:86`）**：无论当前在哪个模式，都 `dataset.checkout_version(target)` 切到目标版本，把状态机改写成 `TimeTravel`。若本就钉在同一版本则什么都不做（`:95` 的 `if *version != target_version`）。
- **`as_latest`（`:69`）**：若已是 `Latest` 直接返回；否则先 `checkout_version(latest_version_id)` 拉到最新，再把状态机改写回 `Latest`（`:76-80`）。

`checkout` 就是 `Latest → TimeTravel`，`checkout_latest`/`restore` 就是 `TimeTravel → Latest`——**状态转换的全部逻辑就这两个方法**，§5 会看到 `NativeTable` 怎么调它们。

### 2.2 `get` 与 `get_mut`：按模式分流的读写两条路

**读路径 `get`（`dataset.rs:137-142`）**：

```rust
pub async fn get(&self) -> Result<DatasetReadGuard<'_>> {
    self.ensure_up_to_date().await?;        // ① 先确保数据满足一致性窗口（可能触发 reload）
    Ok(DatasetReadGuard {
        guard: self.0.read().await,         // ② 再拿读锁，包成只读守卫返回
    })
}
```

**写路径 `get_mut`（`dataset.rs:147-153`）** 比读多了一道关卡：

```rust
pub async fn get_mut(&self) -> Result<DatasetWriteGuard<'_>> {
    self.ensure_mutable().await?;           // ① 守门：若 TimeTravel 模式直接报错
    self.ensure_up_to_date().await?;        // ② 一致性检查（同 get）
    Ok(DatasetWriteGuard {
        guard: self.0.write().await,        // ③ 拿写锁（独占），包成可写守卫
    })
}
```

那道守门关卡 `ensure_mutable`（`dataset.rs:210-219`）：

```rust
pub async fn ensure_mutable(&self) -> Result<()> {
    let dataset_ref = self.0.read().await;
    match &*dataset_ref {
        DatasetRef::Latest { .. } => Ok(()),                       // 实时营业 → 放行
        DatasetRef::TimeTravel { .. } => Err(crate::Error::InvalidInput {
            message: "table cannot be modified when a specific version is checked out".to_string(),
        }),                                                        // 历史档案室 → 拒绝写
    }
}
```

> **这就是"钉在历史版本不能写"的物理实现**。错误消息精确为 `table cannot be modified when a specific version is checked out`（`dataset.rs:215`）——你在 Python/JS 侧 `checkout(3)` 后试图 `add`，最终就撞在这堵墙上。

`get_mut_unchecked`（`dataset.rs:157`）则**故意省掉 `ensure_mutable`**：它是给 `restore` 开的后门（§5.3 详述）——restore 必须在 TimeTravel 模式下写，普通 `get_mut` 会被守门员挡死。

### 2.3 `is_up_to_date`：一致性窗口的判定矩阵

什么叫"数据还新鲜"？答案在 `is_up_to_date`（`dataset.rs:221-243`），它用一个 `match` 元组枚举所有组合：

```rust
match (read_consistency_interval, last_consistency_check) {
    (None, _)            => Ok(true),    // ① 没设刷新间隔 → 永远算新鲜（强缓存，绝不主动盘账）
    (Some(_), None)      => Ok(false),   // ② 设了间隔但从没盘过 → 不新鲜，得盘
    (Some(interval), Some(last)) =>      // ③ 都有 → 比"距上次盘账过了多久" < 间隔
        Ok(&last.elapsed() < interval),
}
// TimeTravel 模式（:239）：比 dataset.version().version == *version
```

| 配置 | 行为 | 适用场景 |
|---|---|---|
| `read_consistency_interval = None` | 永远不主动 reload（除非显式 checkout_latest） | 单写者、追求性能，默认 |
| `= Some(0)`（`Duration::ZERO`） | 每次读都判定为过期 → 每次都查最新版 | 强一致，多进程并发写 |
| `= Some(5s)` | 5 秒内的读复用缓存，超过才去远端盘账 | 容忍轻微滞后，省 IO |

> **Rust 陷阱：`match (Option, Option)` 元组**。把两个可空值打包成元组再 `match`，穷举 `(None,_)/(Some,None)/(Some,Some)` 三种组合，是 Rust 处理"多个可空值的组合逻辑"的标准手法——比嵌套 `if let` 清晰得多，而且编译器会检查你有没有漏掉某种组合。`(None, _)` 里的 `_` 是通配符，意思是"第二个值是什么我不关心"。

> **验证彩蛋**：文件末尾的测试 `test_iops_open_strong_consistency`（`dataset.rs:303-336`）证明了上表第二行——`read_consistency_interval = Duration::ZERO` 时，调一次 `schema()` 只产生 **1 个读 IOP**（`:335` 的 `assert_eq!(stats.read_iops, 1)`）。也就是说"强一致"并不等于"疯狂查询"，它只是"查一次最新版指针"。

---

## 3. 读写守卫与 `Deref` 魔法：为什么能像用 `Dataset` 一样用守卫

`get` / `get_mut` 返回的不是裸 `Dataset`，而是两个"守卫"包装类型。它们存在的意义，是 Rust 的**智能指针透明转发**。

### 3.1 两个守卫的定义

```rust
// 只读守卫（dataset.rs:255-257）
pub struct DatasetReadGuard<'a> {
    guard: RwLockReadGuard<'a, DatasetRef>,     // 持有读锁
}

// 可写守卫（dataset.rs:270-272）
pub struct DatasetWriteGuard<'a> {
    guard: RwLockWriteGuard<'a, DatasetRef>,    // 持有写锁
}
```

它们各自包着一个 tokio 锁守卫。**守卫变量活着的期间，锁就一直被持有；变量被 Drop（离开作用域）的瞬间，锁自动释放**——这就是 Rust 的 RAII。

### 3.2 `Deref` / `DerefMut`：解包 enum 双变体的脏活被藏起来了

只读守卫实现了 `Deref`（`dataset.rs:259-268`）：

```rust
impl Deref for DatasetReadGuard<'_> {
    type Target = Dataset;                         // ① "解引用后是个 Dataset"
    fn deref(&self) -> &Self::Target {
        match &*self.guard {                       // ② 不管是哪个变体，都解出里面的 &Dataset
            DatasetRef::Latest { dataset, .. } => dataset,
            DatasetRef::TimeTravel { dataset, .. } => dataset,
        }
    }
}
```

可写守卫**既实现 `Deref`（`:274`）又实现 `DerefMut`（`:285`）**，后者解出 `&mut Dataset`。

正因为有这两个 impl，上层代码才能写出这种"仿佛守卫就是 Dataset"的调用——回看上篇 §4.3 和本篇 §5：

```rust
self.dataset.get().await?.version()        // 实际是 (DatasetReadGuard).deref().version()
self.dataset.get_mut().await?.delete(p)    // 实际是 (DatasetWriteGuard).deref_mut().delete(p)
```

> **Rust 知识点：`Deref` = 编译期的"自动拆一层"**。实现了 `Deref<Target=Dataset>` 之后，当你对守卫调用一个它自己没有、但 `Dataset` 有的方法时，编译器会**自动插入** `.deref()` 把它当成 `&Dataset`。类似 C++ 的 `operator->`，但发生在编译期、零运行时开销。`&*self.guard` 里的 `*` 先解一层锁守卫拿到 `&DatasetRef`，`&` 再借引用——这是把"锁守卫"和"enum 解包"两层一起脱掉。

> **设计点：只读守卫故意不实现 `DerefMut`**。`DatasetReadGuard` 只有 `Deref` 没有 `DerefMut`，意味着**从类型层面就禁止**了"拿读锁却想改数据"——编译器直接拒绝编译，根本到不了运行时。可写权限被锁死在"必须先 `get_mut` 拿写锁"这一条路上。这是 Rust 用类型系统替你防错的典型范例。

> **比喻：读卡器与写卡器**。`get()` 给你一张**读卡**，刷进金库只能看餐盘上的菜（`Deref &Dataset`）；`get_mut()` 给你**写卡**，才能动手改（`DerefMut &mut Dataset`）。卡一离手（守卫 Drop），门自动锁上——你不用记得"还锁"，作用域结束自动还。

---

## 4. `reload`：锁升级与双重检查

最精彩的一段并发技巧在 `reload`（`dataset.rs:190-203`）。问题是：read_consistency_interval 到期后要去远端刷新到最新版，但 reload 是个**写动作**（要改锁里的 `DatasetRef`），而触发它的往往是**读路径**。如果每次读都先抢写锁来"问一句要不要刷新"，读并发就被毁了。解法是**双重检查锁**：

```rust
pub async fn reload(&self) -> Result<()> {
    if !self.0.read().await.need_reload().await? {   // ① 先用便宜的【读锁】问一句：要刷新吗？
        return Ok(());                               //    不要 → 早退，全程没碰写锁
    }

    let mut write_guard = self.0.write().await;      // ② 真要刷 → 升级到【写锁】（独占）
    // on lock escalation -- check if someone else has already reloaded
    if !write_guard.need_reload().await? {           // ③ 拿到写锁后【再问一次】！
        return Ok(());                               //    因为 ①②之间别人可能已经刷过了
    }

    write_guard.reload().await                       // ④ 确实还得刷 → 真正 reload
}
```

`need_reload`（`dataset.rs:60-67`）的判定：Latest 模式比 `latest_version_id()`（远端最新版号）≠ `version().version`（本地当前版号）；TimeTravel 模式比本地版号 ≠ 钉死的版号。

为什么 ③ 那次"重新检查"不能省？因为在 ① 释放读锁、② 拿到写锁之间，**别的协程可能已经抢先 reload 完了**。如果不重查，你会白白再 reload 一次（多一次远端 IO，甚至可能覆盖更新的状态）。

> **Rust 陷阱 / 并发知识点：double-checked locking（DCL）**。"先用便宜的检查探一次，确认需要才升级到昂贵的独占锁，进了独占区**再检查一次**"——这是经典 DCL 模式。非 Rust 读者类比 Java 里 `synchronized` 单例的 `if (instance == null)` 两次判断。它在**读多写少**场景下是性能与正确性的最佳平衡：绝大多数读根本不碰写锁，只有真到期那一刻才付独占代价。

`as_latest`（`dataset.rs:165-176`）用的是**同一套** DCL：先 `read().await.is_latest()` 探一次，是 Latest 就早退（`:166-168`）；否则升级写锁、再 `write_guard.is_latest()` 探一次（`:171-173`）才真正切换。

> **Rust 陷阱：`set_latest` 的 `unreachable!`**。`DatasetRef::set_latest`（`dataset.rs:113-123`）在遇到非 `Latest` 变体时会 `unreachable!("Dataset should be in latest mode at this point")`（`:121`）——直接 panic。这是一种**契约断言**：调用方（写操作收尾时）必须保证此刻一定是 Latest 模式。非 Rust 读者类比 `assert False` / "这里逻辑上不可能到达，到了就是有 bug"。

---

## 5. `impl BaseTable for NativeTable`：时间旅行五方法

从 `table.rs:1885` 的 `#[async_trait::async_trait] impl BaseTable for NativeTable` 起，`NativeTable` 逐条兑现上篇那份"用工合同"。本节聚焦招牌能力——时间旅行。**注意它们几乎都只有一两行，因为活儿全在 §1-4 的保安 + lance**。

| 合同方法 | 实现行号 | 一句话 |
|---|---|---|
| `version` | `:1895` | `self.dataset.get().await?.version().version` —— 拿读锁问 lance 当前版本 |
| `checkout` | `:1899` | `self.dataset.as_time_travel(version)` —— 切历史档案室 |
| `checkout_latest` | `:1903` | `as_latest(...)` + `reload()` —— 切回实时营业并刷到最新 |
| `restore` | `:1914` | 把当前钉住的历史版本"扶正"成最新版（见 §5.3） |
| `list_versions` | `:1910` | `self.dataset.get().await?.versions().await?` —— 委托 lance 列版本 |

### 5.1 `version` / `list_versions`：纯转发

```rust
async fn version(&self) -> Result<u64> {
    Ok(self.dataset.get().await?.version().version)     // get() 读锁 → Deref → lance Dataset::version()
}
async fn list_versions(&self) -> Result<Vec<Version>> {
    Ok(self.dataset.get().await?.versions().await?)     // 同上，委托 lance 的 versions()
}
```

两行都是"`get()` 拿读卡 → 经 `Deref` 当成 `Dataset` 用 → 问 lance"。`version()` 里第二个 `.version` 是 lance `Version` 结构体的字段。

### 5.2 `checkout` / `checkout_latest`：双向切换

```rust
async fn checkout(&self, version: u64) -> Result<()> {
    self.dataset.as_time_travel(version).await          // Latest → TimeTravel（§2.1）
}

async fn checkout_latest(&self) -> Result<()> {
    self.dataset
        .as_latest(self.read_consistency_interval)      // ① TimeTravel → Latest，并把刷新间隔传下去
        .await?;
    self.dataset.reload().await                         // ② 再 reload 一次，确保拉到真正的最新版
}
```

> **回收伏笔**：上篇 §4 提到 `NativeTable` 存了个 `read_consistency_interval` 字段，注释说"留着 checkout_latest 重建 dataset 时传下去"。这里 `:1905` 就是兑现点——切回 Latest 模式时要重新设定"多久盘一次账"，而那个值只有 `NativeTable` 自己记着。

### 5.3 `restore`：唯一能在时间旅行模式下写的操作

`restore` 把"当前 checkout 的历史版本"扶正为最新版，是时间旅行里最微妙的一个（`table.rs:1914-1934`）：

```rust
async fn restore(&self) -> Result<()> {
    let version = self.dataset
        .time_travel_version().await
        .ok_or_else(|| Error::InvalidInput {                          // ① 必须先 checkout，否则报错
            message: "you must run checkout before running restore".to_string(),
        })?;
    {
        // Use get_mut_unchecked as restore is the only "write" operation that is allowed
        // when the table is in time travel mode.
        // Also, drop the guard after .restore because as_latest will need it
        let mut dataset = self.dataset.get_mut_unchecked().await?;    // ② 走后门拿写锁（绕过 ensure_mutable）
        debug_assert_eq!(dataset.version().version, version);         // ③ 断言：拿到的确实是钉住那版
        dataset.restore().await?;                                    // ④ 委托 lance 真正 restore
    }                                                                // ← 块结束，写锁在此 Drop 释放
    self.dataset
        .as_latest(self.read_consistency_interval).await?;           // ⑤ 出块后切回 Latest 模式
    Ok(())
}
```

三个关键点：
1. **为什么用 `get_mut_unchecked` 而不是 `get_mut`？** 因为 restore 发生时表正处于 TimeTravel 模式，普通 `get_mut` 会被 `ensure_mutable`（§2.2）当场拦死。`get_mut_unchecked`（`dataset.rs:157`）就是为它开的唯一后门。
2. **为什么 ②③④ 包在一对花括号 `{ }` 里？** 这是 Rust 用**块作用域控制锁释放**：写锁守卫 `dataset` 在 `}` 处 Drop、释放写锁。注释（`:1925`）说得很清楚——"drop the guard after .restore because as_latest will need it"。⑤ 的 `as_latest` 内部又要拿锁，如果写锁没在 `}` 处释放，⑤ 就会**死锁**（自己等自己）。
3. **`debug_assert_eq!`（③）** 只在 debug 构建里检查，release 里会被优化掉——一种零成本的自我校验。

> **Rust 陷阱：块作用域 = 锁的生命周期**。`{ let g = lock(); ... }` 这个花括号不是装饰——它精确控制了守卫 `g` 的存活范围，从而控制锁何时释放。**忘了加这对括号，后面再去拿同一把锁就死锁**。这是 Rust RAII 锁管理最容易绊倒新手的地方，上篇 §4 的 `add`（`table.rs:1976`）、本篇 §5.3 的 restore 都靠它。

---

## 6. `merge_insert`：upsert 的点菜单与翻译层

`merge_insert` 实现 upsert（存在则更新、不存在则插入）和其他"按 key 合并"语义。它分两半：用户侧的**点菜单** `MergeInsertBuilder`（`merge.rs`），和后厨侧的**翻译层** `NativeTable::merge_insert`（`table.rs:2236`）。

### 6.1 点菜单 `MergeInsertBuilder`

`merge.rs:15-24`：

```rust
#[derive(Debug, Clone)]
pub struct MergeInsertBuilder {
    table: Arc<dyn BaseTable>,                                       // 目标表（后厨电话）
    pub(crate) on: Vec<String>,                                     // 按哪些列匹配（join key）
    pub(crate) when_matched_update_all: bool,                      // 命中：是否全字段更新
    pub(crate) when_matched_update_all_filt: Option<String>,       //   ↳ 可选的更新条件
    pub(crate) when_not_matched_insert_all: bool,                  // 源有目标无：是否插入
    pub(crate) when_not_matched_by_source_delete: bool,            // 目标有源无：是否删除
    pub(crate) when_not_matched_by_source_delete_filt: Option<String>, // ↳ 可选的删除条件
}
```

`new`（`merge.rs:27`）把所有布尔/Option 字段初始化为 `false`/`None`——**默认什么都不做**，你得显式勾选。三个链式 setter 各管一种语义：

| setter | 行号 | 勾上后的含义 | 比喻 |
|---|---|---|---|
| `when_matched_update_all(cond)` | `:59` | 源和目标都有的行 → 用源行覆盖目标行（可带 SQL 条件） | 菜里已有的，换成新做的 |
| `when_not_matched_insert_all()` | `:67` | 只在源里有的行 → 插入目标 | 菜单上没有的，加上 |
| `when_not_matched_by_source_delete(filt)` | `:81` | 只在目标里有的行 → 删除（可带条件） | 点过又取消的，撤掉 |

经典 upsert = `when_matched_update_all(None)` + `when_not_matched_insert_all()`。

> **Rust 陷阱：setter 返回 `&mut Self`，`execute` 消费 `self`**。三个 setter（`:59/:67/:81`）签名是 `(&mut self) -> &mut Self`——改完字段返回自身的可变借用，所以能 `b.when_matched_update_all(None).when_not_matched_insert_all()` 链式调。但 `execute`（`merge.rs:90`）签名是 `(self, ...)`——它**拿走所有权**，调用后 builder 整个被 move 走、不能再用。这正是"点菜单一次性交单"：

```rust
pub async fn execute(self, new_data: Box<dyn RecordBatchReader + Send>) -> Result<()> {
    self.table.clone().merge_insert(self, new_data).await   // builder 自己就是参数，交给后厨
}
```

注意 `merge_insert(self, ...)`——**整张点菜单（builder）被当作参数塞给后厨**，自己用完作废。这是 builder 一次性消费的惯用法。

### 6.2 翻译层 `NativeTable::merge_insert`

后厨侧（`table.rs:2236-2270`）干的纯粹是"把一堆 bool+Option 翻译成 lance 听得懂的枚举"：

```rust
async fn merge_insert(&self, params: MergeInsertBuilder, new_data: ...) -> Result<()> {
    let dataset = Arc::new(self.dataset.get().await?.clone());                    // ① 拿读锁 clone 出 Dataset
    let mut builder = LanceMergeInsertBuilder::try_new(dataset.clone(), params.on)?; // ② 造 lance 的 builder

    // ③ 翻译 when_matched（命中）→ lance 的 WhenMatched 三态
    match (params.when_matched_update_all, params.when_matched_update_all_filt) {
        (false, _)        => builder.when_matched(WhenMatched::DoNothing),                       // 没勾 → 不动
        (true, None)      => builder.when_matched(WhenMatched::UpdateAll),                       // 勾了无条件 → 全更新
        (true, Some(filt))=> builder.when_matched(WhenMatched::update_if(&dataset, &filt)?),     // 勾了带条件 → 条件更新
    };

    // ④ 翻译 when_not_matched（源有目标无）→ InsertAll / DoNothing
    if params.when_not_matched_insert_all {
        builder.when_not_matched(WhenNotMatched::InsertAll);
    } else {
        builder.when_not_matched(WhenNotMatched::DoNothing);
    }

    // ⑤ 翻译 when_not_matched_by_source（目标有源无）→ Delete / delete_if / Keep
    if params.when_not_matched_by_source_delete {
        let behavior = if let Some(filter) = params.when_not_matched_by_source_delete_filt {
            WhenNotMatchedBySource::delete_if(dataset.as_ref(), &filter)?
        } else {
            WhenNotMatchedBySource::Delete
        };
        builder.when_not_matched_by_source(behavior);
    } else {
        builder.when_not_matched_by_source(WhenNotMatchedBySource::Keep);
    }

    let job = builder.try_build()?;                                              // ⑥ 编译成 lance 的 job
    let (new_dataset, _stats) = job.execute_reader(new_data).await?;            // ⑦ 跑！返回全新 Dataset
    self.dataset.set_latest(new_dataset.as_ref().clone()).await;               // ⑧ 把新版本塞回保安
    Ok(())
}
```

> **比喻：merge_insert 实现就是翻译官**。它把顾客的口头需求（一堆 bool+filter）翻译成后厨听得懂的标准术语（lance 的 `WhenMatched`/`WhenNotMatched`/`WhenNotMatchedBySource` 三个枚举），**自己一道菜都不下厨**——语义全在 lance。

> **关键模式：copy-on-write + set_latest**。注意 ⑦ 返回的是 `new_dataset`（**全新的** Dataset），不是原地改。lance 是只读追加式存储，写操作产出新版本而非修改旧版本，所以 ⑧ 必须 `set_latest` 把保安手里的"当前版本指针"更新过去。**`NativeTable` 所有写方法都遵循这个模式**：`get`/`get_mut` → clone 出 Dataset → 交 lance builder → 拿回 new_dataset → `set_latest` 塞回（对比上篇 §4 的 `add`、本节、§5.3）。

---

## 7. `optimize` 三件套：委托 lance 的 compact / cleanup / index

`optimize`（`table.rs:2278-2328`）实现上篇 §3.3 介绍的 `OptimizeAction` 枚举。它本身只是个**分发器**，三种动作各委托一个 lance 能力：

```rust
async fn optimize(&self, action: OptimizeAction) -> Result<OptimizeStats> {
    let mut stats = OptimizeStats { compaction: None, prune: None };
    match action {
        OptimizeAction::All => {                                    // ① 默认：递归调自己跑三遍
            stats.compaction = self.optimize(OptimizeAction::Compact { .. }).await?.compaction;
            stats.prune      = self.optimize(OptimizeAction::Prune   { .. }).await?.prune;
            self.optimize(OptimizeAction::Index(OptimizeOptions::default())).await?;
        }
        OptimizeAction::Compact { options, remap_options } =>      // ② 合并小文件
            stats.compaction = Some(self.compact_files(options, remap_options).await?),
        OptimizeAction::Prune { older_than, .. } =>               // ③ 删旧版本
            stats.prune = Some(self.cleanup_old_versions(
                older_than.unwrap_or(Duration::try_days(7).expect("valid delta")), .. ).await?),
        OptimizeAction::Index(options) =>                         // ④ 增量并索引
            self.optimize_indices(&options).await?,
    }
    Ok(stats)
}
```

三个 helper 都是"拿写锁 + 委托 lance"的薄壳：

| helper | 行号 | 委托的 lance 能力 |
|---|---|---|
| `compact_files` | `:1408` | `lance::dataset::optimize::compact_files(&mut ds, ...)`（`:1414`） |
| `cleanup_old_versions` | `:1388` | `ds.cleanup_old_versions(older_than, ...)`（`:1398`） |
| `optimize_indices` | `:1352` | `ds.optimize_indices(options)`（`:1357`） |

> **`All` 用"递归调自己"实现**（`:2284-2302`）：与其复制三段逻辑，`All` 分支直接 `self.optimize(Compact)` → `self.optimize(Prune)` → `self.optimize(Index)` 顺序跑一遍，收集各自的 stats。这是用递归避免重复代码的小技巧。

> **Prune 默认保留 7 天**（`:2316`）：`older_than.unwrap_or(Duration::try_days(7)...)`——不显式指定就删 7 天前的旧版本。这个保守默认是为了不误删"可能还在进行中的事务"的文件（参见 `cleanup_old_versions` 的文档注释 `:1381-1384`）。`Duration::try_days(7)` 返回 `Option`（极端情况下构造可能失败），`.expect("valid delta")` 在这里 unwrap——7 天显然永远是合法时长，所以敢 expect。

---

## 8. Schema 演进：add / alter / drop columns

三个 schema 演进方法（`table.rs:2330-2355`）是全篇**最纯粹的"一行转发"**，把"拿写锁 + 委托 lance"这个模式压到了极致：

```rust
async fn add_columns(&self, transforms: NewColumnTransform, read_columns: Option<Vec<String>>) -> Result<()> {
    self.dataset.get_mut().await?
        .add_columns(transforms, read_columns, None).await?;        // 直接转交 lance Dataset::add_columns
    Ok(())
}
async fn alter_columns(&self, alterations: &[ColumnAlteration]) -> Result<()> {
    self.dataset.get_mut().await?.alter_columns(alterations).await?;
    Ok(())
}
async fn drop_columns(&self, columns: &[&str]) -> Result<()> {
    self.dataset.get_mut().await?.drop_columns(columns).await?;
    Ok(())
}
```

三者都是 `get_mut()` 拿写锁 → 经 `DerefMut` 当 `&mut Dataset` 用 → 调 lance 同名方法。**注意它们没有 `set_latest`**——因为这里直接拿到的是写锁守卫（可变借用 lance Dataset 本体），lance 的 `add_columns`/`alter_columns`/`drop_columns` 是**原地改 `&mut self`** 而非返回新 Dataset，所以无需替换指针。这和 §6 的 `merge_insert`（clone + 返回 new_dataset + set_latest）形成对照——**写法取决于 lance 那一侧是"原地改"还是"返新值"**，贡献时改哪种要照着 lance 的签名走。

> **给贡献者**：要加一种 schema 演进能力（比如重命名列的新模式），路径是固定的——①在 `BaseTable` trait（`table.rs:408`）加方法签名；②在这里加一行 `get_mut` 转发；③在 `RemoteTable` 加 HTTP 实现。绝大多数情况下"多接一个 lance 能力"就够了，不需要重写逻辑。

---

## 9. 番外：`BaseTableAdapter` —— 把表反向接进 DataFusion

上篇讲的是"表向外提供能力"。`table/datafusion.rs` 做的是**反向桥接**：把 LanceDB 表伪装成 DataFusion 的 `TableProvider`，这样别的 SQL 引擎也能来查我们的表。`BaseTableAdapter`（`datafusion.rs:118-122`）：

```rust
#[derive(Debug)]
pub struct BaseTableAdapter {
    table: Arc<dyn BaseTable>,      // 被包装的表（任意 BaseTable）
    schema: Arc<ArrowSchema>,       // 预先剥掉 metadata 的 schema
}
```

`try_new`（`:125`）在构造时就 `with_metadata(HashMap::default())` 把 schema 的 metadata 抹空。核心是 `scan`（`datafusion.rs:152-192`）——它把 DataFusion 的查询参数翻译回 LanceDB 的 `QueryRequest`，**再调回 `create_plan` 复用同一套 lance Scanner**：

```rust
async fn scan(&self, state, projection, filters, limit) -> DataFusionResult<Arc<dyn ExecutionPlan>> {
    let mut query = QueryRequest::default();
    if let Some(projection) = projection { ... query.select = Select::Columns(列名) }  // ① 列下标→列名
    if !filters.is_empty() {
        let first = filters.first().unwrap().clone();
        let filter = filters[1..].iter().fold(first, |acc, e| acc.and(e.clone()));    // ② 多个 filter 用 AND 折成一个
        query.filter = Some(QueryFilter::Datafusion(filter));
    }
    // ③（简化示意）透传 limit；源码是 if let Some(l) = limit { query.limit = Some(l) } else { query.limit = None }
    query.limit = limit;             //   无 limit 时显式置 None，覆盖默认的 10（datafusion.rs:174-179，注释见 :177）
    let plan = self.table.create_plan(&AnyQuery::Query(query), options).await?;       // ④ 调回 create_plan（§ 见 06）
    Ok(Arc::new(MetadataEraserExec::new(plan)))                                       // ⑤ 外套抹 metadata 算子
}
```

两个值得记的细节：
- **`supports_filters_pushdown`（`:194-199`）全返回 `Exact`**：声明"过滤我能精确下推到底层，DataFusion 你自己别再过滤一遍"。这就是为什么 §6 的 filter 能一路压到 lance Scanner。
- **`MetadataEraserExec`（`:29`）的存在理由**：注释（`:25-27`）说 DataFusion 会维护 batch metadata 而这会触发 DF 的 bug，所以末尾（`:191`）套一个算子把 metadata 撕掉。`statistics`（`:201`）目前返回 `None` 带 `TODO`——这是个现成的贡献切入点。

> **比喻：传菜流水线的入口闸机**。`BaseTableAdapter` 把后厨（LanceDB 表）伪装成 DataFusion 流水线认得的标准料理台（`TableProvider`），别的菜系（SQL 引擎）也能来这台子取菜，取菜时还顺手把餐盘上多余的标签撕掉（`MetadataEraserExec`）。

> **Rust 陷阱：`fold` 合并表达式**（`datafusion.rs:169-171`）。`filters[1..].iter().fold(first, |acc, e| acc.and(e.clone()))` 是函数式的"累加"：以第一个 filter 为初值，把后续每个用 `.and()` 串进去，最终得到一个大 AND 表达式。`fold` 就是别的语言里的 `reduce`/`accumulate`。这里 `e.clone()` 是因为 `Expr` 不是 `Copy` 类型，借来的引用不能直接搬走，得克隆一份。

---

## 10. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `struct X(Y)`（元组结构体） | 无字段名，用 `self.0` 访问 | `DatasetConsistencyWrapper` `dataset.rs:19` |
| `tokio::sync::RwLock` | 异步读写锁，`.read()/.write()` 要 `.await` | `dataset.rs:11` |
| `enum` 带数据变体 | 每个变体携带自己的字段，`match` 强制穷举 | `DatasetRef` `dataset.rs:26` |
| `match (Option, Option)` | 元组打包多个可空值再穷举组合 | `is_up_to_date` `dataset.rs:228` |
| `Deref` / `DerefMut` | 智能指针透明转发，编译期自动"拆一层" | 守卫 `dataset.rs:259/285` |
| double-checked locking | 读锁探→升写锁→再探，读多写少最优 | `reload` `dataset.rs:190`、`as_latest` `:165` |
| `unreachable!` | "逻辑上不可达"的契约断言，到了就 panic | `set_latest` `dataset.rs:121` |
| 块作用域 `{ }` 控制锁释放 | 守卫在 `}` 处 Drop，释放锁；忘了会死锁 | `restore` `table.rs:1922`、`add` `:1976` |
| `&mut Self` setter / `self` consumer | 链式 setter 借用、`execute` 消费所有权 | `merge.rs:59/90` |
| `unwrap_or(...)` / `.expect(...)` | 给 `Option` 兜底默认 / 断言一定有值 | `optimize` `table.rs:2316` |
| `fold` 折叠 | 函数式 reduce/accumulate | `scan` `datafusion.rs:169` |
| `debug_assert_eq!` | 只在 debug 构建检查的零成本断言 | `restore` `table.rs:1927` |
| `pub(crate)` / `pub(super)` | 收敛可见性，外部用户看不到、改它们不破坏公共 API | `merge.rs:18`、`new` `:27` |

---

## 11. 动手验证（建议亲手做一遍）

1. **撞墙实验**：在任意语言绑定里对一张表 `checkout(某历史版本)` 后立刻 `add` 一批数据，观察报错消息。回到源码定位它——你会落在 `ensure_mutable`（`dataset.rs:214`），消息正是 `table cannot be modified when a specific version is checked out`。理解"TimeTravel 不可写"是物理强制的，不是约定。
2. **数 reload 的 IO**：读懂 `test_iops_open_strong_consistency`（`dataset.rs:303`）。把断言 `read_iops == 1`（`:335`）改成 `== 2` 跑测试，看它怎么 fail——体会"强一致也只查一次最新版指针"。
3. **跟一次 upsert**：从 `MergeInsertBuilder::execute`（`merge.rs:90`）手动追到 `NativeTable::merge_insert`（`table.rs:2236`），把三组 bool/filter 在 `:2243-2265` 的 `match`/`if` 里逐一对到 lance 的 `WhenMatched`/`WhenNotMatched`/`WhenNotMatchedBySource`。问自己：经典 upsert 该勾哪两个 setter？
4. **找 set_latest 的规律**：在 `table.rs` 里搜 `set_latest`，看看哪些写方法用了它（`add`/`update`/`merge_insert`）、哪些没用（`add_columns`/`drop_columns`/`delete`）。对照 lance 那侧的签名，总结"何时需要 set_latest"的规则（提示：lance 返新 Dataset 就要，原地改 `&mut self` 就不要）。
5. **找贡献切入点**：跳到 `BaseTableAdapter::statistics`（`datafusion.rs:201`）那个 `// TODO` + `None`。想想给 DataFusion 提供真实统计信息（行数、列的 min/max）能怎么实现——这是个真实的 good-first-issue 切入点。

---

## 12. 小结 & 下一篇

把上、下两篇拼起来，`NativeTable` 这个模块的全貌就完整了：

- **金库保安 `DatasetConsistencyWrapper`（§1-4）**：一个 `Arc<RwLock<DatasetRef>>` 的元组结构体，只管"何时 reload、谁能写、新版本是谁"三件事。双模式状态机（Latest/TimeTravel）是时间旅行的骨架，读写守卫靠 `Deref` 把"解 enum 双变体"的脏活藏起来，`reload`/`as_latest` 用双重检查锁在读多写少下兼顾性能与正确。
- **后厨实现 `impl BaseTable for NativeTable`（§5-8）**：时间旅行五方法、merge_insert、optimize 三件套、schema 演进——**几乎每一行都是"拿锁 → clone/借用 Dataset → 委托 lance → (按需) set_latest"**。`NativeTable` 是翻译官和编排者，不是实现者。
- **反向桥接 `BaseTableAdapter`（§9）**：把表伪装成 DataFusion `TableProvider`，又调回 `create_plan` 复用 lance Scanner——双向都在编排，不在重造。

**贡献导航**（改这个模块时去哪）：
- 加并发/一致性行为 → `table/dataset.rs`
- 加表级操作 → `BaseTable` trait（`table.rs:408`）+ `NativeTable` impl（`:1886`）+ `RemoteTable`
- 加 upsert 选项 → `table/merge.rs` 加字段+setter，`table.rs:2236` 加翻译分支（两处配套改）
- 让表参与 SQL → `table/datafusion.rs`

改动几乎总是"**多接一个 lance 能力**"，而非重写逻辑——这就是 LanceDB 全篇反复强调的"组装而非重造"。

**下一篇**将离开 `table.rs`，跟随一次真正的查询执行，看 `create_plan`（本篇 §5 只组装、不执行）产出的 `ExecutionPlan` 如何在 DataFusion 流水线里跑起来，与 `06 查询链路` 互补印证。
