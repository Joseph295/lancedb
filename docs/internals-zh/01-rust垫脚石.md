# 01 · Rust 垫脚石：读 lancedb 源码会绊倒你的 9 个语法点

> **本篇解剖的源码**：`rust/lancedb/src/error.rs`、`table.rs`、`table/dataset.rs`、`arrow.rs`、`database.rs`、`catalog.rs`、`connection.rs`（只挑能讲清概念的真实片段，不求全）
> **覆盖区间**：不按文件顺序，而按「概念」组织——所有权 / trait·dyn / Arc / async / Result·错误 / 泛型·impl Trait / 派生宏 / 内部可变性 / 可见性
> **留给哪篇**：每个概念的「业务含义」（BaseTable 为什么这样设计、DatasetConsistencyWrapper 怎么保一致性）留给 `09 Table 解剖·上` 及之后各篇；本篇只负责「让你看得懂语法」
> **读前建议**：你懂至少一门语言（Python / JS），但没写过 Rust。**不必背**——把这篇当字典，读后续源码卡住时回来查对应小节即可。

---

## 0. 本篇要回答的问题

**一句话**：本篇不教你写 Rust，只教你**读** lancedb 的 Rust——把那些"非 Rust 读者一定会卡住"的 9 个语法点，用库里的真实代码一次性讲透。

**比喻锚点**：你要去一个说着方言的城市旅游（贡献 lancedb 代码）。你不需要先精通这门方言，但得认得路牌上反复出现的那十几个字。本篇就是那张"路牌速查卡"。

**为什么 lancedb 是绝佳教材**：记住这个贯穿全系列的心智模型——

```
                    LanceDB（薄薄一层"组装"代码）
                   /          |            \
            站在三个巨人的肩膀上，组装而非重造
                 /            |              \
          lance            Arrow          DataFusion
      （磁盘列存·落盘）   （内存列存·Schema）  （执行引擎·查询计划）
```

正因为 lancedb 是"胶水层"，它的代码里几乎每一行都在做三件事之一：**包一个指针、转发一个调用、翻译一个错误**。这意味着——Rust 的核心语法点（所有权、trait、Arc、async、Result）在这里出现得**又密又典型**，是天然的活教材。

下面 9 节，每节固定四步走：**概念 → 类比你会的语言 → lancedb 真实例子（带行号）→ 为什么这样设计**。

---

## §1 所有权 / 借用 / 移动：`&self` / `mut self` / `self` 到底差在哪

### 概念

这是 Rust 最独特、也最容易绊倒新人的一点。每个值都有**唯一的所有者**；函数参数里的 `self` 有三种写法，代表三种"我要怎么对待这个对象"的态度：

| 写法 | 含义 | 调用后原对象还能用吗 |
|---|---|---|
| `&self` | **共享只读借用**：我只是看看，不改、也不拿走 | 能 |
| `&mut self` | **独占可变借用**：我要改它，期间别人不能碰 | 能 |
| `self`（含 `mut self`） | **拿走所有权**：这东西归我了，用完就销毁 | **不能**（被"移动"走了） |

### 类比你会的语言

Python / JS 里**没有"按值消耗"的概念**——对象都是引用，传来传去都还活着，垃圾回收器负责善后。Rust 把"谁负责销毁"显式写进了类型：

| Python / JS | Rust 对应 |
|---|---|
| `def look(self):`（只读方法） | `fn look(&self)` |
| `def mutate(self):`（改自己） | `fn mutate(&mut self)` |
| （无对应）"用完这个对象就不准再碰它" | `fn consume(self)` |

最后一行没有等价物——这是你最容易栽的地方。

### lancedb 真实例子

一条 `tbl.add(data).mode(...).execute()` 链，就把三态全演了一遍。

**第一态 `&self`**——`Table::add`（`table.rs:619`）只是"看一眼" Table 然后造个 builder，不消耗 Table：

```rust
pub fn add<T: IntoArrow>(&self, batches: T) -> AddDataBuilder<T> {   // ① &self：借用，table 之后还能用
    AddDataBuilder {
        parent: self.inner.clone(),                                  // ② clone 见 §3
        data: batches,
        mode: AddDataMode::Append,
        write_options: WriteOptions::default(),
        embedding_registry: Some(self.embedding_registry.clone()),
    }
}
```

**第二态 `mut self`**——builder 的链式方法 `mode`（`table.rs:299`）和 `write_options`（`table.rs:304`）**拿走 builder、改一个字段、再整个还回去**：

```rust
pub fn mode(mut self, mode: AddDataMode) -> Self {   // ① mut self：按值拿走 builder，且本地可改
    self.mode = mode;                                // ② 改字段
    self                                             // ③ 把改好的 builder 还回去
}
```

`UpdateBuilder` 也是同款，`only_if`（`table.rs:343`）、`column`（`table.rs:367`）都用 `mut self -> Self`。

**第三态 `self`（用完即焚）**——终结方法 `execute`（`table.rs:309`）拿走 builder 后**不再返回它**，builder 至此销毁：

```rust
pub async fn execute(self) -> Result<()> {       // ① self（非 &self）：吃掉 builder，调用后它就没了
    let parent = self.parent.clone();
    let data = self.data.into_arrow()?;
    let without_data = AddDataBuilder::<NoData> { /* ... */ };
    parent.add(without_data, data).await         // ② 真正委托后厨执行
}
```

`UpdateBuilder::execute`（`table.rs:378`）同理，返回 `Result<u64>`。

### 为什么这样设计

> **Rust 陷阱：`mut self` ≠ `&mut self`**
> - `mut self`：按值**拿走所有权**，且在方法内部允许修改。调用 `b.mode(x)` 后，原来的 `b` 变量就**被移动走、不能再用**了——这正是 builder "一次性点菜单"的实现方式。
> - `&mut self`：**独占借用**，改完对象还在原地。
>
> Python / JS 读者最大的误区：以为 `b.mode(x).execute()` 之后 `b` 还在。在 Rust 里它已经"焚毁"。

**为什么 builder 要用 `mut self -> Self` 这种消耗式写法？** 因为 Rust **没有可选参数、默认参数、函数重载**。要表达"加数据时可以配 mode、可以配 write_options、也可以都不配"，最地道的办法就是 builder：每个方法接过菜单、勾一项、传回去，`execute` 收单。消耗所有权保证了**同一张菜单不会被两个地方同时改**——编译期就杜绝了并发改配置的 bug。

> **比喻（点菜单家族）**：`mut self -> Self` 是"传纸条点菜"——每个方法把菜单拿过来勾一项再传回，`execute(self)` 把菜单交给后厨、菜单作废。详见 `09 Table 解剖·上 §3.1`。

---

## §2 trait = 接口，dyn = 运行时多态

### 概念

- **`trait`**：一组方法签名，规定"想当某种东西，必须会做哪些事"。它只定义"能力清单"，不含数据。
- **`dyn Trait`**：一个**运行时多态**的"接口指针"。`dyn BaseTable` 的意思是"某个实现了 BaseTable 的东西，但具体是谁，运行时才知道"。调用方法时通过虚表（vtable）动态派发。

### 类比你会的语言

| 概念 | Java | Go | Python | Rust |
|---|---|---|---|---|
| 接口 | `interface` | `interface` | 抽象基类 / `Protocol` | `trait` |
| 运行时多态变量 | `List list` | `var w io.Writer` | duck typing | `dyn Trait`（要放指针后面） |

关键差异：Java/Python 里"接口变量"天然就是运行时多态；Rust 必须**显式写 `dyn`**，并且 `dyn Trait` 没有固定大小，必须包在指针里用（`&dyn T`、`Box<dyn T>`、`Arc<dyn T>`——见 §3）。

### lancedb 真实例子

库里有三个核心 trait，都是"用工合同"，规定后端必须实现哪些方法。最重要的是 `BaseTable`（`table.rs:407-408`）：

```rust
#[async_trait]                                                          // ① 见 §4
pub trait BaseTable: std::fmt::Display + std::fmt::Debug + Send + Sync {  // ② 冒号后是 supertrait 约束
    fn as_any(&self) -> &dyn std::any::Any;                            // ③ 用于向下转型（详见 09 §2.4）
    async fn version(&self) -> Result<u64>;
    // ……共 29 个方法（详见 09 Table 解剖·上 §1.2）
}
```

冒号后的 `Display + Debug + Send + Sync` 叫 **supertrait 约束**：想实现 `BaseTable`，必须先满足这四个。其中 `Send + Sync` 是"能跨线程的通行证"（数据库要被并发使用），编译器强制。

另外两个同构的 trait：

| trait | 行号 | supertrait 约束 | 方法返回的多态类型 |
|---|---|---|---|
| `Database` | `database.rs:151` | `Send + Sync + Any + Debug + Display + 'static` | `Result<Arc<dyn BaseTable>>`（`database.rs:157`） |
| `Catalog` | `catalog.rs:68` | `Send + Sync + Debug + 'static` | `Result<Arc<dyn Database>>`（`catalog.rs:73`） |

注意它们都返回 `Arc<dyn XXX>`——这就是 §3 要讲的"接口指针"。

### 为什么这样设计

`BaseTable` 这份合同同时被**本地表 `NativeTable`** 和**云端表 `RemoteTable`** 实现。上层代码永远只跟 `dyn BaseTable` 打交道，**从不写 `if 本地 else 云端`**。要把本地换成云端，只需换掉躲在 `Arc<dyn BaseTable>` 后面的实现，门面层一行不改。

> **trait 的两种身份要分清**（贯穿全库）：
> - 作为**约束**：`fn add<T: IntoArrow>`（`table.rs:619`）——编译期单态化，零运行时成本（见 §6）。
> - 作为 **dyn 对象**：`Arc<dyn BaseTable>`（`table.rs:501`）——运行期虚表派发，有一次指针跳转开销，换来多态。
>
> lancedb 的惯例：**泛型用于入口 API（追求零成本），dyn 用于存储可替换的后端（追求多态）**。

> **比喻（用工合同家族）**：`trait` = 用工合同（岗位职责清单），`dyn BaseTable` = "一个签了这份合同的人，但具体哪位待定"。详见 `09 Table 解剖·上 §1`。

---

## §3 `Arc<dyn T>`：共享所有权 + 接口指针，`clone` 只 +1 计数

### 概念

`Arc<T>` = **A**tomically **R**eference **C**ounted，原子引用计数智能指针。它让**多个所有者共享同一份数据**：每 `clone` 一次，内部计数 +1；每个 Arc 销毁时计数 -1；归零时才真正释放数据。

`Arc<dyn T>` 把 §2 的"接口指针"和 §3 的"共享所有权"合二为一——既能多态，又能被多处持有。这是全库**最高频的多态手段**。

### 类比你会的语言

| 语言 | 行为 |
|---|---|
| Python / JS | 对象本来就是共享引用，`b = a` 后两者指向同一对象——天然就是 Arc 的效果，只是隐式 |
| Rust | 默认**不**共享（所有权唯一），要共享必须显式用 `Arc`，并显式 `clone` 来"多发一个所有权份额" |

最大的坑：JS 读者看到 `.clone()` 会以为是**深拷贝**。在 `Arc` 上，`.clone()` **只复制指针 + 计数 +1，绝不复制底层数据**。

### lancedb 真实例子

`Table` 结构体（`table.rs:499-503`）只有两个字段，**全是 `Arc<dyn T>`**：

```rust
#[derive(Clone)]                                  // ① 见 §7
pub struct Table {
    inner: Arc<dyn BaseTable>,                    // ② 指向后厨（本地表 or 云表）
    embedding_registry: Arc<dyn EmbeddingRegistry>,
}
```

正因为两个字段都是 Arc，`#[derive(Clone)]` 出来的 `Table::clone` **极其廉价**——克隆一张表 = 两个计数各 +1，不搬一行数据。

`add`（`table.rs:621`）里那句 `self.inner.clone()`：

```rust
parent: self.inner.clone(),   // 复制的是 Arc 指针（再印一张遥控器），不是底层 BaseTable
```

你会在源码里看到铺天盖地的 `self.inner.clone()` / `self.dataset.clone()`——读到时心里要清楚：**廉价指针复制，不是搬数据**。`Arc<dyn` 这个组合在 `/src` 里出现约 92 处。

### 为什么这样设计

数据库对象（表、连接）要被多个并发任务同时持有。如果用唯一所有权，就得到处传引用、处处标生命周期，极其难写。`Arc` 让"共享"变简单，`dyn` 让"可替换后端"变可能，二者合体的 `Arc<dyn T>` 因此成了 lancedb 的主梁。

> **Rust 陷阱：`.clone()` on Arc 不是深拷贝**
> 印多张遥控器控同一台电视，不是买多台电视。JS 读者尤其要改掉"clone = 深拷贝"的直觉。

> **比喻（遥控器家族）**：`Arc<dyn BaseTable>` = 一张遥控器，`clone` = 再印一张，背后是同一台电视（同一个后厨）。印 100 张，电视还是那一台。详见 `09 Table 解剖·上 §2.1`。

---

## §4 `async` / `await` 与 `#[async_trait]`：为什么数据库到处是 async

### 概念

- **`async fn`**：声明一个"异步函数"。它被调用时**不立即执行**，而是返回一个 `Future`（"未来会算出结果的承诺"）。
- **`.await`**：在 `Future` 上等待结果。等待期间，当前线程**不阻塞**，可以去干别的活——这是高并发的关键。
- **`#[async_trait]`**：一个属性宏。Rust 原生 trait 当年还**不能直接写 `async fn` 又保持对象安全**，这个宏在编译期把 `async fn foo(&self)` 改写成 `fn foo(&self) -> Pin<Box<dyn Future>>`，绕过限制。

### 类比你会的语言

`async` / `await` 你大概率见过：

| 语言 | 异步语法 |
|---|---|
| JS | `async function f() { await g(); }` ——几乎一模一样 |
| Python | `async def f(): await g()` ——也几乎一样 |
| Rust | `async fn f() { g().await; }` ——注意 `.await` 是**后缀**，不是前缀关键字 |

唯一陌生的是 `#[async_trait]`——JS/Python 的接口/协议里直接写 `async` 方法就行，没有这道宏。

### lancedb 真实例子

三大 trait 全都顶着 `#[async_trait]`：

| trait | 宏所在行 |
|---|---|
| `BaseTable` | `table.rs:407` |
| `Catalog` | `catalog.rs:67` |
| `Database` | `database.rs:150`（写法是 `#[async_trait::async_trait]`，全路径） |

`NativeTable` 兑现合同时（`table.rs:1885-1897`），一个方法体里就同时出现了 `async`、`.await`、`?`、`Deref`：

```rust
#[async_trait::async_trait]
impl BaseTable for NativeTable {
    async fn version(&self) -> Result<u64> {
        Ok(self.dataset.get().await?.version().version)
        //              ①get() 是 async ②.await 等它 ③? 见 §5 ④.version() 来自 Deref，见 §8
    }
}
```

### 为什么这样设计

数据库的本质就是 **IO 密集**：读磁盘、发网络请求、等锁。如果每次 IO 都阻塞线程，几千个并发查询就要几千个线程，内存和切换开销爆炸。`async` 让一个线程在等待 IO 时切去服务别的请求，**用少量线程扛住海量并发**——这就是"数据库到处是 async"的根本原因。

> **Rust 陷阱：`#[async_trait]` 不是装饰，是必需**
> 去掉它，trait 里的 `async fn` 直接编译失败。你照抄 trait 时**最容易漏掉这一行**。看到 trait 里有 `async fn`，先确认顶上有没有 `#[async_trait]`。

> **比喻（同声传译器）**：`#[async_trait]` 是同声传译——trait 里你写的是"人话" `async fn`，宏在编译期把它翻成 Rust 编译器听得懂的 `Pin<Box<dyn Future>>` 机器语。

---

## §5 `Result` / `?` / 错误传播 与 LanceDB 的 `Error` 枚举

### 概念

Rust **没有异常**。函数靠返回 `Result<T, E>` 表达"成功给 T，失败给 E"：

- `Ok(value)` —— 成功。
- `Err(error)` —— 失败。
- **`?` 运算符**：贴在可能出错的表达式后面。成功就解包出值继续；失败就**立刻 `return` 这个错误**，并隐式调用 `From` 把它转成本函数的错误类型。

### 类比你会的语言

| Python / JS | Rust |
|---|---|
| `try: x = f() except E: raise` | `let x = f()?;` |
| 异常自动向上冒泡 | `?` 显式地、逐层地把错误"冒泡"出去 |
| 一个 `Exception` 类 + message 字符串 | 一个**结构化的 `enum Error`**，每个变体是一种"故障形状" |

最大的认知差：异常是"控制流隐式跳走"，`?` 是"**显式写在代码里的提前 return**"。不写 `?`，编译器会因为类型不匹配（`Result` ≠ `T`）直接报错。

### lancedb 真实例子

**第一步·错误枚举**（`error.rs:9-85`）。用 `snafu` 库定义结构化错误，每个变体配一句"人话"模板：

```rust
#[derive(Debug, Snafu)]                                          // ① snafu 帮你生成 Display/Error
#[snafu(visibility(pub(crate)))]
pub enum Error {
    #[snafu(display("Table '{name}' was not found"))]            // ② 人话模板
    TableNotFound { name: String },                              // error.rs:16-17
    // ……
    #[snafu(display("lance error: {source}"))]
    Lance { source: lance::Error },                              // ③ 包装第三方错误 error.rs:42-43
}
```

**第二步·项目级 `Result` 别名**（`error.rs:87`）：

```rust
pub type Result<T> = std::result::Result<T, Error>;   // 全库写 Result<u64> 默认就带本地 Error
```

这就是为什么全库函数签名只写 `Result<u64>`、`Result<()>` 就够——`E` 默认是本地 `Error`。

**第三步·`From` 转换链**（`error.rs:89-123`），这是 `?` 背后的隐式机制：

```rust
impl From<ArrowError> for Error { /* ... */ }          // error.rs:89  Arrow 错误 → 本地 Error
impl From<lance::Error> for Error {                    // error.rs:95
    fn from(source: lance::Error) -> Self {
        Self::Lance { source }
    }
}
impl From<object_store::Error> for Error { /* ... */ } // error.rs:103
impl<T> From<PoisonError<T>> for Error {               // error.rs:117 泛型 From：锁中毒 → Runtime
    fn from(e: PoisonError<T>) -> Self {
        Self::Runtime { message: e.to_string() }
    }
}
```

**三步合一**，看 `NativeTable::version`（`table.rs:1896`）那句 `self.dataset.get().await?`：`get()` 返回的是 `Result`，`?` 在出错时提前 return；若错误来自 lance，就自动走 `error.rs:95` 的 `From` 转成 `Error::Lance`。**`Result`（声明）→ `?`（传播）→ `From`（转换）三者是一条流水线，缺一不可。**

当没有现成 `From` 时，就手动用闭包 `.map_err(...)` 转换（见 §10）。例如富 schema 反序列化（`table.rs:131-133`）：

```rust
serde_json::from_str(column_definitions).map_err(|e| Error::Runtime {   // 把 serde 错误改写成本地 Error
    message: format!("Failed to deserialize column definitions: {}", e),
})?
```

### 为什么这样设计

结构化枚举比"一个 Exception + 字符串"强在哪？**调用方能 `match` 不同变体分别处理**——遇到 `TableNotFound` 可以创建，遇到 `Lance` 则原样上报。贡献者新增一种错误，就是往枚举里**加一个变体**，编译器会强制所有 `match` 处理它（或显式忽略），不会漏。

> **Rust 陷阱：`?` 不是异常**
> 它是语法糖，等价于"`match`：`Ok` 取值、`Err` 提前 return 并 `.into()`"。不写它编译器报类型错；它也不会"跳过几层栈"——每一层都得自己写 `?` 才会继续冒泡。

> **比喻（故障代码手册 + 保险丝）**：`enum Error` 是故障代码手册，每个变体一种故障码，`#[snafu(display)]` 是故障的人话说明，`source` 字段是根因追溯链。`?` 是保险丝——电流（数据）正常就通过，一旦短路（Err）立刻跳闸，把错误顺着 `From` 链送出去。

---

## §6 泛型 与 `impl Trait`：`add<T: IntoArrow>` 和 `impl Into<String>`

### 概念

- **泛型 `<T: SomeTrait>`**：写一份代码，对所有满足约束的类型都管用。编译期为每个具体类型生成专门版本（**单态化**），零运行时开销。
- **`impl Trait` 作参数**：`fn f(x: impl Into<String>)` 是泛型的语法糖——"接受任何能转成 String 的类型"。
- **blanket impl**：`impl<T: A> B for T`——给**所有满足 A 的类型**一次性实现 B。Rust 特有的"批量赋能"。

### 类比你会的语言

| 概念 | Java/TS 泛型 | Python | Rust |
|---|---|---|---|
| 泛型 + 约束 | `<T extends Comparable>` | duck typing（运行期） | `<T: Trait>`（编译期检查） |
| blanket impl | 无直接对应 | 最接近的是 mixin / monkey-patch，但那是运行期 | `impl<T: ...> Trait for T`（编译期、类型安全） |

`impl Into<String>` 对 Python/JS 读者最反直觉：你写 `start_after("foo")` 传个字面量也能编译，会困惑"类型怎么对上的"——答案是泛型 + 自动 `.into()`。

### lancedb 真实例子

**泛型方法 + trait 约束**——`Table::add`（`table.rs:619`）：

```rust
pub fn add<T: IntoArrow>(&self, batches: T) -> AddDataBuilder<T> { /* ... */ }
//         └ T 必须实现 IntoArrow，调用方传 RecordBatch / 自定义类型都行
```

`AddDataBuilder` 本身也是泛型结构体（`table.rs:280`）：`pub struct AddDataBuilder<T: IntoArrow>`。

**blanket impl**——`IntoArrow` trait（`arrow.rs:119-130`）：

```rust
pub trait IntoArrow {
    fn into_arrow(self) -> Result<Box<dyn arrow_array::RecordBatchReader + Send>>;
}

impl<T: arrow_array::RecordBatchReader + Send + 'static> IntoArrow for T {  // ★一行赋能所有
    fn into_arrow(self) -> Result<Box<dyn arrow_array::RecordBatchReader + Send>> {
        Ok(Box::new(self))
    }
}
```

这一个 blanket impl 让**所有** `RecordBatchReader` 自动获得 `into_arrow()`，无需逐个实现。

**`impl Trait` 作参数**——connection builder 里随处可见：

| 方法 | 行号 | 参数 |
|---|---|---|
| `start_after` | `connection.rs:58` | `impl Into<String>` |
| `storage_option` | `connection.rs:235` | `key: impl Into<String>, value: impl Into<String>` |
| `storage_options` | `connection.rs:257` | `impl IntoIterator<Item = (impl Into<String>, impl Into<String>)>` |

所以 `conn.start_after("foo")`（传 `&str`）和 `conn.start_after(some_string)`（传 `String`）都能编译——函数内部统一 `.into()`。`UpdateBuilder::column`（`table.rs:367`）也用了两个 `impl Into<String>`。

### 为什么这样设计

泛型让入口 API **既灵活又零成本**：调用方爱传啥传啥，编译器为每种类型生成专版，运行时没有派发开销。blanket impl 则避免了"为 100 个类型抄 100 遍 impl"。这正是 lancedb 入口层"对用户友好、对机器高效"的秘诀。

> **Rust 陷阱：blanket impl 会"占满"类型空间**
> `impl<T: ...> IntoArrow for T`（`arrow.rs:126`）几乎覆盖所有类型，再想为某个具体类型单独 impl 就会撞"conflicting implementation"（孤儿/重叠规则）。所以同文件里 `IntoArrowStream`（`arrow.rs:148`、`arrow.rs:154`）改成对具体类型逐个 impl。你自己加 impl 时若报这个错，回头看是不是撞上了 blanket impl。

> **比喻（批量发驾照）**：blanket impl = 凡是会开车（实现了 `RecordBatchReader`）的类型，自动发一张 `into_arrow` 驾照，不用一个个去考。

---

## §7 派生宏：`#[derive(...)]` / `#[default]` / `#[allow(dead_code)]`

### 概念

`#[xxx]` 是**属性宏**，贴在类型/字段/方法上，让编译器在编译期帮你生成或调整代码：

- `#[derive(Clone, Debug, ...)]`：**自动生成** trait 实现，省去手写样板。
- `#[default]`：标在 enum 的某个变体上，配合 `#[derive(Default)]` 指定"默认是哪个变体"。
- `#[allow(dead_code)]`：**压制**"这段代码没被用到"的警告。

### 类比你会的语言

| 概念 | 类比 |
|---|---|
| `#[derive(Debug)]` | Python 的 `@dataclass` 自动生成 `__repr__`；Java Lombok 的 `@Data` |
| 属性宏 | 装饰器（Python `@decorator`）/ 注解（Java `@Annotation`），但在**编译期**展开 |

### lancedb 真实例子

**`#[derive(...)]`** 到处都是：`Table`（`table.rs:499`）`#[derive(Clone)]`；`DatasetConsistencyWrapper`（`dataset.rs:18`）`#[derive(Debug, Clone)]`；请求结构体如 `TableNamesRequest`（`database.rs:38`）`#[derive(Clone, Debug, Default)]`。

**`#[default]`** + 派生 `Default`——`BadVectorHandling`（`table.rs:240-253`）：

```rust
#[derive(Clone, Debug, Default)]
enum BadVectorHandling {
    #[default]                                                   // ① 默认变体就是 Error
    Error,
    #[allow(dead_code)] // https://github.com/lancedb/lancedb/issues/992
    Drop,                                                        // ② 占位、暂未启用
    #[allow(dead_code)] // https://github.com/lancedb/lancedb/issues/992
    Fill(f32),
    #[allow(dead_code)] // https://github.com/lancedb/lancedb/issues/992
    None,
}
```

**手动 `Default`**（当 `#[derive(Default)]` 不够灵活时，比如 enum 变体带数据）——`OptimizeAction`（`table.rs:223-227`）：

```rust
impl Default for OptimizeAction {
    fn default() -> Self {
        Self::All
    }
}
```

`CreateTableMode`（`database.rs:81-85`）、`CreateDatabaseMode`（`catalog.rs:51-55`）同样手写 `Default`。

### 为什么这样设计

`#[derive]` 把"每个结构体都要手写一遍 `Clone`/`Debug`"的样板消灭掉——库里几百个类型，少打几千行。`#[allow(dead_code)]` + 指向 issue 的注释，则是 Rust 项目里"**功能挖了坑但还没填**"的典型信号。

> **给贡献者的信号**：看到 `#[allow(dead_code)] // https://.../issues/992` 这种组合，八成是个**尚未启用的占位功能**——常常就是 good-first-issue 的切入点。动手前先去对应 issue 看上下文（这里的 `BadVectorHandling` 详见 `09 Table 解剖·上 §3.4`）。

> **比喻（贴标签家族延伸）**：`#[derive]` = 给类型自动盖一排"它会哪些能力"的合规章，不用你一个个手写。

---

## §8 内部可变性：为什么 `&self` 也能改数据

### 概念

回看 §1：`&self` 是"只读借用"。但你会发现 lancedb 里很多"明明在改数据"的方法签名却是 `&self`——这怎么可能？

答案是**内部可变性（interior mutability）**：把"可变性"藏进一个特殊容器（`RwLock` / `Mutex` / `RefCell`）里。对外暴露 `&self`（共享只读），对内通过容器在**运行时**拿到可变访问。编译期的"独占可变"检查被放行，改由运行时的锁来保证安全。

`RwLock`（读写锁）：允许**多个读者同时进**，但**写者必须独占**。

### 类比你会的语言

Python / JS **没有借用检查**，对象随便改，并发安全全靠你自己加锁（或 GIL 兜底）。Rust 反过来：默认禁止 `&self` 改数据（编译期拦），你想改就得显式用 `RwLock` 之类，等于告诉编译器"我用锁保证安全了，放行吧"。

| 语言 | 共享 + 可变 |
|---|---|
| Python/JS | 默认就能，安全自负 |
| Rust | 默认不能；用 `Arc<RwLock<T>>` 显式开启，编译期放行、运行期上锁保证安全 |

### lancedb 真实例子

核心就是 `DatasetConsistencyWrapper`（`dataset.rs:18-19`）：

```rust
#[derive(Debug, Clone)]
pub struct DatasetConsistencyWrapper(Arc<RwLock<DatasetRef>>);
//                                   └ Arc 让多人共享，RwLock 控制读写
```

注意用的是 **`tokio::sync::RwLock`**（`dataset.rs:11`），不是 `std::sync` 版——异步锁，加锁要 `.await`。

读写访问都建立在 `&self` 上（`dataset.rs:137-153`）：

```rust
pub async fn get(&self) -> Result<DatasetReadGuard<'_>> {   // ① 外层 &self（只读借用）
    self.ensure_up_to_date().await?;
    Ok(DatasetReadGuard {
        guard: self.0.read().await,                         // ② 内部却拿到读锁
    })
}

pub async fn get_mut(&self) -> Result<DatasetWriteGuard<'_>> {  // ① 外层仍是 &self！
    self.ensure_mutable().await?;
    self.ensure_up_to_date().await?;
    Ok(DatasetWriteGuard {
        guard: self.0.write().await,                        // ② 内部拿到写锁——能改数据
    })
}
```

**关键反直觉点**：`get_mut(&self)` 外层是不可变借用 `&self`，内部却能拿写锁改数据。这就是内部可变性。`self.0` 是元组结构体的字段访问——`DatasetConsistencyWrapper(Arc<...>)` 只有一个字段，`.0` 取它。

返回的 guard 通过 **`Deref`**（`dataset.rs:259-267`）"假装"成 `Dataset`：

```rust
impl Deref for DatasetReadGuard<'_> {
    type Target = Dataset;
    fn deref(&self) -> &Self::Target {        // 解引用时返回里面的 Dataset
        match &*self.guard {
            DatasetRef::Latest { dataset, .. } => dataset,
            DatasetRef::TimeTravel { dataset, .. } => dataset,
        }
    }
}
```

这就是为什么 §4 那句 `self.dataset.get().await?.version()` 里的 `.version()` 能直接调用——它其实是 `Deref` 出来的 `Dataset` 的方法。`DatasetWriteGuard` 还额外实现了 `DerefMut`（`dataset.rs:285`），可拿可变引用。

**生命周期标注**——guard 借自锁，不能比锁活得久（`dataset.rs:255`）：

```rust
pub struct DatasetReadGuard<'a> {             // <'a>：这个 guard 借了某个东西，活不过它
    guard: RwLockReadGuard<'a, DatasetRef>,
}
```

`get` 的返回类型写 `DatasetReadGuard<'_>`（`dataset.rs:137`）——`'_` 是"生命周期省略"，让编译器自己推断。

**并发模式·锁升级双重检查**（`dataset.rs:190-203`）——先读锁判断，需要时再升写锁、并二次确认（防止两个线程同时重载）：

```rust
pub async fn reload(&self) -> Result<()> {
    if !self.0.read().await.need_reload().await? {   // ① 先拿读锁快速判断
        return Ok(());
    }
    let mut write_guard = self.0.write().await;      // ② 升级到写锁
    // on lock escalation -- check if someone else has already reloaded
    if !write_guard.need_reload().await? {           // ③ 二次检查：别人可能已重载
        return Ok(());
    }
    write_guard.reload().await                       // ④ 确实需要才重载
}
```

### 为什么这样设计

`NativeTable` 的方法对外暴露 `&self`（这样它能被 `Arc` 共享、被多个并发任务调用），但底层 `Dataset` 又确实需要被改（写入会产生新版本）。内部可变性是唯一出路：**对外 `&self` 保证可共享，对内 `RwLock` 保证改的时候不打架**。

> **Rust 陷阱：tokio 锁 ≠ std 锁**
> `dataset.rs:19` 用的是 `tokio::sync::RwLock`，加锁是 `.read().await` / `.write().await`（带 `await`）。`std::sync::RwLock` 的加锁不带 `await`、会阻塞线程——在 async 代码里误用 std 锁可能**卡死整个 runtime**。看到 `.read().await` 就知道是异步锁。

> **比喻（金库保安家族）**：`Arc<RwLock<DatasetRef>>` = 金库。多人可同时进去看账本（read），但要改账本必须独占（write），保安（锁）负责排队。`Deref` = 万能转接头，让 guard 插上就当 `Dataset` 用。详见 `09 Table 解剖·上 §4.1`。

---

## §9 可见性：`pub` / `pub(crate)`

### 概念

Rust **默认一切私有**。你得显式标 `pub` 才能让外部看到：

| 标注 | 可见范围 |
|---|---|
| （无） | 仅当前模块（及子模块） |
| `pub(crate)` | 仅本 crate（本库）内可见，**对库的使用者隐藏** |
| `pub` | 完全公开，库的用户能用 |

### 类比你会的语言

| 概念 | Java | Python | Rust |
|---|---|---|---|
| 公开 | `public` | （约定）无下划线 | `pub` |
| 包内可见 | package-private（默认） | （约定）单下划线 `_x` | `pub(crate)` |
| 私有 | `private` | （约定）双下划线 | 默认（不标） |

差异：Python 的"私有"只是命名约定（`_x` 你照样能访问）；Rust 的可见性是**编译器强制**的，越界访问直接编译失败。

### lancedb 真实例子

`pub(crate)` 大量用于"库内部要传递、但不想暴露给用户"的字段。`AddDataBuilder`（`table.rs:280-285`）：

```rust
pub struct AddDataBuilder<T: IntoArrow> {
    parent: Arc<dyn BaseTable>,                       // ① 无标注 = 私有，连本 crate 其他模块都看不到
    pub(crate) data: T,                               // ② pub(crate)：本库内部可读写
    pub(crate) mode: AddDataMode,
    pub(crate) write_options: WriteOptions,
    embedding_registry: Option<Arc<dyn EmbeddingRegistry>>,  // ③ 又是私有
}
```

`NativeTable` 的核心字段 `dataset`（`09 §4` 提到，`table.rs` 内）也是 `pub(crate)`——库内的 builder/操作模块要访问它，但 lancedb 的用户不该碰。

错误枚举顶部还有个模块级控制（`error.rs:10`）：

```rust
#[snafu(visibility(pub(crate)))]   // snafu 生成的错误构造器只在本 crate 内可见
```

### 为什么这样设计

**可见性即 API 边界**。`pub` 的东西是对用户的承诺——一旦公开，改它就是破坏性变更。`pub(crate)` 让库作者能在内部模块间自由共享数据，**同时不把这些内部结构泄露成公开 API**，保留未来重构的自由。读源码时，`pub(crate)` 是个明确信号：**这是内部管线，不是给你（用户）调的**。

> **给贡献者的提示**：想知道某个字段/函数"算不算公开 API"，看它的可见性标注。改 `pub` 的签名要慎重（影响所有用户）；改 `pub(crate)` 或私有的，影响只在库内。

---

## 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 类比你会的语言 | 出现处 |
|---|---|---|---|
| `&self` | 共享只读借用，对象之后还能用 | 普通只读方法 | `add` `table.rs:619` |
| `&mut self` | 独占可变借用 | 改自身的方法 | `DerefMut` `dataset.rs:286` |
| `mut self -> Self` | 消费自身的链式 builder，调用后原变量失效 | （无对应） | `mode` `table.rs:299` |
| `self`（终结） | 拿走所有权用完即焚 | （无对应） | `execute` `table.rs:309` |
| `trait T: A + B` | 接口 + supertrait 约束 | `interface` / 抽象基类 | `BaseTable` `table.rs:408` |
| `Arc<dyn T>` | 共享所有权 + 接口指针；`clone` 只 +1 计数 | JS 共享引用（但显式） | `Table.inner` `table.rs:501` |
| `async fn` / `.await` | 异步函数 / 等待结果（后缀） | JS/Python `async`/`await` | `version` `table.rs:1895` |
| `#[async_trait]` | 让 trait 能写 `async fn` 的宏，必需 | （无对应） | `table.rs:407` |
| `Result<T>` / `?` | 无异常的错误传播；`?` = 出错就提前 return + From 转换 | `try/except` + `raise` | `error.rs:87`、`table.rs:1896` |
| `enum Error` + `snafu` | 结构化错误枚举，每变体一种故障 | 一个 Exception + message | `error.rs:9-85` |
| `impl From<X> for Error` | `?` 背后的自动错误转换 | 异常包装 | `error.rs:95` |
| `<T: Trait>` 泛型 | 编译期单态化，零成本 | TS/Java 泛型 | `add<T: IntoArrow>` `table.rs:619` |
| `impl Into<String>` | 接受任何可转 String 的类型 | （无对应，duck typing 近似） | `start_after` `connection.rs:58` |
| blanket impl `impl<T> Tr for T` | 给所有满足约束的类型批量实现 | mixin（但编译期） | `arrow.rs:126` |
| `#[derive(...)]` | 自动生成 trait 实现 | `@dataclass` / Lombok | `table.rs:499` |
| `#[default]` | 指定 enum 默认变体 | — | `table.rs:243` |
| `#[allow(dead_code)]` | 压制"未使用"警告（常标占位功能） | — | `table.rs:245` |
| 内部可变性 `RwLock` | `&self` 也能改，可变性藏锁里 | （无；Rust 特有约束） | `dataset.rs:19` |
| `Deref` | 让包装类型"假装"成内部类型 | — | `dataset.rs:259` |
| `<'a>` / `<'_>` | 生命周期：引用不能逃逸出借用源 | （无对应） | `dataset.rs:255` |
| `pub(crate)` | 仅本库内可见，对用户隐藏 | package-private / `_x` | `table.rs:282` |

---

## 动手验证（建议亲手做一遍）

1. **三态一条链**：打开 `table.rs`，把 `add`（:619，`&self`）→ `mode`（:299，`mut self`）→ `execute`（:309，`self`）三个签名抄到一起对比。然后试着在脑子里跑 `tbl.add(d).mode(x).execute()`，问自己：`execute` 之后那个 builder 变量还能再 `.execute()` 一次吗？（答案：不能，已被移动。）

2. **数 Arc**：在 `/rust/lancedb/src` 下跑 `grep -rn "Arc<dyn" *.rs | wc -l`，看看"接口指针"出现了多少处（约 92）。再在 `table.rs` 搜 `.clone()`，挑三处确认它复制的是 Arc 而非数据。

3. **追一条错误流水线**：从 `table.rs:1896` 的 `self.dataset.get().await?` 出发——`get()` 返回什么类型？`?` 在出错时会把 lance 的错误转成哪个 `Error` 变体？去 `error.rs:95` 验证那条 `From<lance::Error>`。

4. **找内部可变性**：在 `dataset.rs` 里确认 `get_mut`（:147）的签名是 `&self`，但内部 `self.0.write().await`（:151）拿到了写锁。再找 `Deref`（:259），理解为什么 `get().await?.version()` 能直接调 `Dataset` 的方法。

5. **认占位功能**：跳到 `table.rs:240` 的 `BadVectorHandling`，数一数有几个变体标了 `#[allow(dead_code)]` 并指向同一个 issue。点开那个 issue，体会"挖坑未填"的功能长什么样——这可能是你的第一个贡献落点。

---

## 小结 & 下一篇

把 9 个路牌收进口袋：

- **§1 所有权**：`&self`（借看）/ `mut self`（拿走改了还回，builder）/ `self`（用完即焚，execute）。
- **§2-3 trait + dyn + Arc**：trait 是合同，`Arc<dyn T>` 是"可共享的接口指针"，`clone` 只 +1 计数。
- **§4 async**：数据库是 IO 密集，靠 async 用少量线程扛海量并发；`#[async_trait]` 是必需的宏。
- **§5 Result/?/Error**：无异常，`Result` 声明 → `?` 传播 → `From` 转换三件套；错误是结构化枚举。
- **§6 泛型/impl Trait**：入口 API 用泛型（零成本灵活），blanket impl 批量赋能。
- **§7 派生宏**：`#[derive]` 消灭样板，`#[allow(dead_code)]` 常标占位功能。
- **§8 内部可变性**：`&self` 也能改，可变性藏在 `RwLock` 里，`Deref` 让 guard 假装成 `Dataset`。
- **§9 可见性**：`pub(crate)` = 库内部管线，不是给用户的 API。

**核心心智模型**（请刻进脑子）：LanceDB 站在 **lance（磁盘列存）/ Arrow（内存列存）/ DataFusion（执行引擎）** 三个巨人肩上，**组装而非重造**。它的代码里几乎每一行都在"包指针、转调用、翻译错误"——这就是为什么这 9 个语法点在这里如此密集、如此典型。

**下一篇推荐**：带着这张速查卡去读 `09 Table 解剖·上`——你会看到 `BaseTable` 合同（§2）、`Table` 门面持双 `Arc`（§3）、Builder 的 `mut self`（§1）、`DatasetConsistencyWrapper` 的锁（§8）全部在真实业务场景里联动。本篇是字典，09 是第一篇正文。
