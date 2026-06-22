# 17 · FFI 边界全解：一个方法如何穿过 napi-rs / pyo3 暴露给 Node 与 Python

> **本篇解剖的源码**：
> - Node 侧绑定：`nodejs/src/table.rs`、`nodejs/src/error.rs`、`nodejs/src/iterator.rs`、`nodejs/src/connection.rs`、`nodejs/src/lib.rs`、`nodejs/Cargo.toml`、`nodejs/lancedb/table.ts`、`nodejs/lancedb/query.ts`、`nodejs/package.json`
> - Python 侧绑定：`python/src/table.rs`、`python/src/error.rs`、`python/src/arrow.rs`、`python/src/connection.rs`、`python/src/lib.rs`、`python/Cargo.toml`、`python/python/lancedb/background_loop.py`、`python/python/lancedb/table.py`、`python/Makefile`
>
> **覆盖范围**：三层架构的最后一层缝合——用户语言（JS/Python）↔ 绑定层（napi-rs/pyo3 薄壳）↔ 核心（`lancedb::Table`）。重点是：**同一个 Rust 核心方法如何被两端各包一层暴露出去**，以及四个跨边界难题（解包 inner / 转参 Arrow / 驱动 async / 转错误）两端各自怎么解。
>
> **前置阅读**：`00 全景地图`（三层架构那张图）、`01 Rust 垫脚石`（`Arc` / async / trait 四节）、`02 三大基石`（Arrow 内存格式）、`09 Table 解剖·上`（`lancedb::Table` 是个"服务员"门面）。本篇所有论断都带真实行号，形如 `nodejs/src/table.rs:78`，可点击跳转。

---

## 0. 这一篇要回答的问题

你已经读完了核心引擎（`00`~`16`）。但用户从来不写 Rust——他们写的是：

```python
# Python
tbl.add(data)            # 同步
await async_tbl.add(data)  # 异步
```

```javascript
// Node.js
await tbl.add(data);
```

这一篇要回答：**`tbl.add(...)` 这一行 JS/Python 代码，是怎么最终走到 `lancedb::Table::add` 这个 Rust 方法上的？** 拆开来是四个具体问题：

1. JS 的 `class Table`、Python 的 `class Table`，跟 Rust 的 `lancedb::Table` 是什么关系？谁包着谁？
2. Rust 核心是 **async** 的，但 JS 用 `Promise`、Python 同步用户压根不知道 async——中间怎么桥接？
3. `data`（一个 pandas DataFrame / JS Arrow Table）这么大坨数据，怎么跨语言边界**不被拷贝一遍又一遍**？
4. Rust 抛的 `lancedb::Error`，怎么变成 Python 能 `except ValueError` 捕获、JS 能 `catch` 的异常？

### 比喻锚点：绑定层 = 同声传译

> 沿用 `09` 的比喻家族——后厨（`lancedb` 核心，Rust）只说一种语言：**Rust + Arrow**。顾客（Python/JS 用户）说的是另一种语言。**绑定层（napi-rs 壳 / pyo3 壳）就是坐在边界上的同声传译员**：把顾客的话译进后厨（解码入参）、把后厨端出的菜译给顾客（编码出参），但**传译员自己不做菜**——所有业务逻辑都在 `lancedb::Table` 里，绑定层一行业务代码都不写。

把三层架构画全（呼应 `00` 那张图，这次补上传译员这一层）：

```
   ┌─────────────────┐   ┌─────────────────┐
   │  Python 用户      │   │  Node.js 用户    │
   │  tbl.add(df)     │   │  tbl.add(data)   │
   └───────┬─────────┘   └────────┬────────┘
           │ pyarrow / asyncio    │ Arrow.js / Promise
   ┌───────▼─────────┐   ┌────────▼────────┐
   │ python/python/   │   │ nodejs/lancedb/  │   ← 宿主语言薄层(.py/.ts)
   │   lancedb/*.py   │   │   *.ts           │     做参数整理、同步/异步分流
   └───────┬─────────┘   └────────┬────────┘
   ╔═══════▼═════════════════════ ▼════════╗
   ║  绑定层 = 同声传译 (Rust, 但是"壳")      ║
   ║  python/src/table.rs   nodejs/src/    ║   ← #[pyclass]      #[napi]
   ║  pyo3 #[pymethods]     table.rs #[napi]║     解 inner / 转参 / 驱动 async / 转错误
   ╚═══════╤═════════════════════ ╤════════╝
           │  self.inner_ref()?.add(...)    │
   ┌───────▼─────────────────────▼────────┐
   │   lancedb::Table  (核心门面 = 服务员)    │   rust/lancedb/src/table.rs (见 09)
   │   inner: Arc<dyn BaseTable>            │
   └───────────────┬───────────────────────┘
                   ▼
            lance / Arrow / DataFusion (后厨)
```

注意有**两层薄壳**：靠近用户那层是宿主语言写的（`.py` / `.ts`），靠近核心那层是 Rust 写的（`src/*.rs`，但编译成 `.so` / `.node`）。本篇主角是中间那条粗框线——**Rust 写的绑定壳**。

---

## 1. 总览：四件事，两端对称

先把结论摊开。无论 Node 还是 Python，绑定壳里的每个方法都只做**四件事**，业务全在 `lancedb::Table`：

| 步骤 | 干什么 | Node 怎么做 | Python 怎么做 |
|---|---|---|---|
| ① 解包 inner | 从 `Option<Table>` 取出真表，已关闭就报错 | `inner_ref()?`（`nodejs/src/table.rs:29`） | `inner_ref()?`（`python/src/table.rs:89`） |
| ② 转参/转出参 | Arrow 数据跨边界 | Arrow **IPC buffer**（序列化+拷贝） | Arrow **C Data Interface**（零拷贝） |
| ③ 驱动 async | 把 Rust future 跑起来、桥回宿主语言 | napi `"async"` feature 自动变 `Promise` | `future_into_py` 手动包成 awaitable |
| ④ 转错误 | `lancedb::Error` → 宿主异常 | 拍平成一条字符串（`default_error`） | 按变体映射到不同异常类（`infer_error`） |

两端**结构高度对称**，差异集中在 ②③④ 的"手法"上。本篇 §2 看 Node、§3 看 Python 各自怎么搭壳，§4~§6 逐一深挖 async / Arrow / 错误这三处差异，§7 给一张三处对照表，§8 给贡献者"改了核心要同步改哪儿"的清单。

> **三层架构收口（呼应 `00`）**：`00` 那张全景图的最顶端写着"用户代码 → 通过 FFI 绑定 → 核心引擎"。本篇就是把"FFI 绑定"这个箭头放大 100 倍看清楚。读完你会明白：**绑定层没有魔法，它就是四件机械活的循环**。

---

## 2. Node 侧：`#[napi]` 把 struct 变成 JS 类

### 2.1 一个 `#[napi]` struct = 一个 JS class

Node 侧的表壳定义在 `nodejs/src/table.rs:20-26`：

```rust
#[napi]                              // ← 这一行：把下面的 struct 导出成一个 JS class
pub struct Table {
    // 留一份表名副本，关闭后报错还能带上表名
    pub name: String,
    pub(crate) inner: Option<LanceDbTable>,   // 真正干活的核心 Table，被 Option 包着
}
```

`LanceDbTable` 是个别名，指向核心的 `lancedb::table::Table`——`nodejs/src/table.rs:8-11` 的 `use` 里写着 `Table as LanceDbTable`。这就是"传译员手里握着后厨的电话"。

> **Rust 知识点 · `#[napi]` 过程宏**：`#[napi]` 不是普通注解，它是一个**过程宏（procedural macro）**——编译时会读取你的 `struct`/`impl`，**自动生成一大坨胶水代码**（N-API 的 C ABI 入口、JS 端的类型声明等），让这个 Rust 类型在 JS 世界里现身成一个 `class Table`。非 Rust 读者可以理解成"一个会改写源码的超级装饰器"。你写 Rust struct，它替你生成 JS class——你看不见生成的代码，但 `npm run build` 时它跑了。

为什么 `inner` 是 `Option<Table>` 而不是直接 `Table`？为了支持 `close()`。看 `nodejs/src/table.rs:53-61`：

```rust
#[napi]
pub fn is_open(&self) -> bool {
    self.inner.is_some()        // Some 就是开着, None 就是关了
}

#[napi]
pub fn close(&mut self) {
    self.inner.take();          // ★ take(): 把 Some 里的 Table 拿走, 原地留 None
}
```

> **Rust 陷阱 · `Option<T>` + `.take()`**：`close()` 只有一行 `self.inner.take()`。`take()` 把 `Option` 里的值**移走**、原地留下 `None`，并返回原值（这里没接住，直接丢弃 → `Table` 被 `drop` → 资源释放）。非 Rust 读者容易以为这只是"清空字段"，其实是**所有权转移**：核心 Table 被销毁了。之后任何方法走 `inner_ref()` 都会拿到 `None` → 报"Table is closed"。
>
> **比喻**：`Option<Table>` 是服务员还挂着的工牌。`close()` 用 `take()` 把工牌摘下（变 `None`），服务员（核心 Table）就下班了；之后顾客再点单，`inner_ref()` 只会回一句"该桌已结账（Table is closed）"。`name` 单独存一份，就是为了下班后还能喊出"这是 X 号桌"。

### 2.2 `inner_ref()`：每个方法的第一步

`nodejs/src/table.rs:28-34`：

```rust
impl Table {
    fn inner_ref(&self) -> napi::Result<&LanceDbTable> {
        self.inner
            .as_ref()                                          // Option<Table> -> Option<&Table>
            .ok_or_else(|| napi::Error::from_reason(           // None -> 一个 JS Error
                format!("Table {} is closed", self.name)))
    }
}
```

`as_ref()` 把 `Option<Table>` 变成 `Option<&Table>`（借用，不夺走所有权），`ok_or_else` 把 `None` 这一情况转成 `Err(...)`。于是 `inner_ref()?` 这一句就同时完成了"解包 + 已关闭检查"，是绑定壳每个方法的第一句。

### 2.3 同步方法 vs async 方法

`#[napi] impl Table`（`nodejs/src/table.rs:36-37`）里两类方法泾渭分明：

**同步小方法**直接返回值——`is_open`(:54)、`close`(:59)、`display`(:46)。

**真正干活的方法**几乎都是 `pub async fn`，并且带 `#[napi(catch_unwind)]`。以 `count_rows` 为例（`nodejs/src/table.rs:94-101`，逐字摘录）：

```rust
#[napi(catch_unwind)]
pub async fn count_rows(&self, filter: Option<String>) -> napi::Result<i64> {
    self.inner_ref()?                  // ① 解包 inner（已关闭则提前 return Err）
        .count_rows(filter)            // ② 调核心方法（这里入参简单, 无需 Arrow 转换）
        .await                         // ③ 等 async 完成
        .map(|val| val as i64)         // ④ Rust usize -> JS number(用 i64 表示)
        .default_error()               // ⑤ lancedb::Error -> napi::Error
}
```

五步对应 §1 的四件事（②④合并成了"转参/转出参"）。**整个方法体没有一行业务逻辑**——业务在 `.count_rows(filter)` 那一跳里，那是核心 `lancedb::Table` 的方法（见 `09`）。

> **Rust 陷阱 · `#[napi(catch_unwind)]`**：这个属性让 napi 在 FFI 边界**捕获 Rust 的 panic**，转成一个 JS 异常。为什么必须捕获？因为 Rust panic **穿过 FFI 边界（C ABI）是未定义行为**，会直接让 Node 进程崩溃。`catch_unwind` 在边界上架了张网，把 panic 兜成正常的 JS 异常。`nodejs/src/table.rs` 里几乎所有方法都带它（:64/:77/:94/:103/:108…）。

`add` 方法展示了 Arrow 入参的处理（`nodejs/src/table.rs:77-92`）：

```rust
#[napi(catch_unwind)]
pub async fn add(&self, buf: Buffer, mode: String) -> napi::Result<()> {
    let batches = ipc_file_to_batches(buf.to_vec())     // ① IPC buffer -> RecordBatch（有拷贝）
        .map_err(|e| napi::Error::from_reason(
            format!("Failed to read IPC file: {}", e)))?;
    let mut op = self.inner_ref()?.add(batches);        // ② 解包 + 调核心 add，拿到 builder
    op = if mode == "append" {
        op.mode(AddDataMode::Append)
    } else if mode == "overwrite" {
        op.mode(AddDataMode::Overwrite)
    } else {
        return Err(napi::Error::from_reason(format!("Invalid mode: {}", mode)));
    };
    op.execute().await.default_error()                  // ③ 执行 + 转错误
}
```

注意签名直接是 `pub async fn`，但 JS 端拿到的是一个返回 `Promise` 的函数——这个"魔法"在 §4 讲。注意 `buf: Buffer` 和 `buf.to_vec()`：**Node 侧 Arrow 数据是序列化成 IPC 字节流传进来的**，`.to_vec()` 是一次拷贝（§5 细讲）。

### 2.4 TS 层：用户真正接触的那层

用户不直接碰 `#[napi]` 生成的原生类。中间还隔着一层 TypeScript（`nodejs/lancedb/table.ts`）。`LocalTable` 持有原生表（`nodejs/lancedb/table.ts:455-456`）：

```typescript
export class LocalTable extends Table {
  private readonly inner: _NativeTable;   // ← _NativeTable 就是 #[napi] 生成的原生 Table 类
```

`add` 在 TS 层先把数据编码成 buffer，再调原生方法（`nodejs/lancedb/table.ts:492-498`）：

```typescript
async add(data: Data, options?: Partial<AddDataOptions>): Promise<void> {
    const mode = options?.mode ?? "append";
    const schema = await this.schema();
    const buffer = await fromDataToBuffer(data, undefined, schema);  // JS Arrow -> IPC buffer
    await this.inner.add(buffer, mode);                              // 调原生, await 它的 Promise
}
```

这就接上了 §2.3 那个 Rust `add(buf: Buffer, ...)`。**TS 层负责把 JS 的 Arrow 数据序列化成 buffer**，Rust 壳负责把 buffer 反序列化回 RecordBatch——一来一回正是同声传译的"译进/译出"。

---

## 3. Python 侧：`#[pyclass]` 把 struct 变成 Python 类

### 3.1 对称的壳，不同的注解

Python 侧表壳定义在 `python/src/table.rs:62-67`，**和 Node 侧几乎一模一样**：

```rust
#[pyclass]                              // ← 把下面的 struct 导出成一个 Python class
pub struct Table {
    // 留一份 name 副本, inner 被 drop 后还能用
    name: String,
    inner: Option<LanceDbTable>,        // 同样是 Option<核心 Table>
}
```

`inner_ref` 也同构（`python/src/table.rs:88-94`），只是错误类型从 `napi::Error` 换成了 `PyRuntimeError`：

```rust
fn inner_ref(&self) -> PyResult<&LanceDbTable> {
    self.inner
        .as_ref()
        .ok_or_else(|| PyRuntimeError::new_err(    // None -> Python RuntimeError
            format!("Table {} is closed", self.name)))
}
```

`close` 同样是 `self.inner.take()`（`python/src/table.rs:108-110`）。**两端的"工牌"机制完全一致**。

> **Rust 知识点 · `#[pyclass]` / `#[pymethods]`**：`#[pyclass]` 标在 struct 上，告诉 pyo3"把这个类型暴露成 Python 类"；`#[pymethods]` 标在 `impl` 块上（`python/src/table.rs:96-97`），里面的方法就成了 Python 方法。和 napi 一样，它们是过程宏，编译时生成胶水。**区别在 async 的处理上**，见下。

### 3.2 关键差异：Python 的 async 方法签名是同步 `fn`

Node 可以直接写 `pub async fn`，pyo3 **不行**——`async fn` 不能直接当 `#[pymethod]`。所以 Python 侧每个异步方法的签名都是**同步 `fn`，返回 `PyResult<Bound<PyAny>>`**（一个 Python awaitable 对象）。看 `count_rows`（`python/src/table.rs:169-178`，逐字摘录），和 §2.3 的 Node 版对照：

```rust
#[pyo3(signature = (filter=None))]
pub fn count_rows(                                  // ← 是同步 fn, 不是 async fn!
    self_: PyRef<'_, Self>,
    filter: Option<String>,
) -> PyResult<Bound<'_, PyAny>> {                   // ← 返回一个 Python awaitable
    let inner = self_.inner_ref()?.clone();         // ① 解包 + clone 出 Table（内部 Arc, 廉价）
    future_into_py(self_.py(), async move {         // ② 把 future 交给 tokio, 返回 Python awaitable
        inner.count_rows(filter).await.infer_error()  // ③ 调核心 + 转错误
    })
}
```

三处需要解释的非 Rust 读者陷阱：

> **Rust 陷阱 · `self_: PyRef<'_, Self>` 不是普通 `&self`**：pyo3 里，方法想拿到 Python 的 GIL token（用来构造 Python 对象、调 `future_into_py`）时，第一个参数写成 `self_: PyRef<'a, Self>`，再用 `self_.py()` 取出那个 token。它本质上仍是"对 self 的借用"，但额外携带了 GIL 信息。

> **Rust 陷阱 · 为什么先 `clone()` 再 `async move`**：第 ① 步 `let inner = self_.inner_ref()?.clone();`，第 ② 步 `async move { inner... }`。因为这个 future **可能在方法返回之后才真正执行**（交给 tokio 排队了），那时 `self_` 早已不在——所以**不能在 future 里借用 `self_`**，必须把需要的东西（`inner`）`clone` 出来 `move` 进 future。好在核心 `Table` 内部是 `Arc<dyn BaseTable>`（见 `09`），`clone` 只是引用计数 +1，几乎免费。

> **Rust 知识点 · `Bound<'_, PyAny>`**：pyo3 0.23 用 `Bound<'_, PyAny>` 表示"一个带 GIL 生命周期的 Python 对象引用"，约等于"一个借来的、此刻活在 GIL 下的 Python 对象"。`future_into_py` 返回的就是包成 `Bound<PyAny>` 的那个 awaitable。

### 3.3 `add` 的 Arrow 入参——零拷贝

Python 的 `add`（`python/src/table.rs:120-139`）和 Node 版最大的不同在第一行：

```rust
pub fn add<'a>(
    self_: PyRef<'a, Self>,
    data: Bound<'_, PyAny>,       // ← 直接收一个 Python 对象(如 pyarrow Table/RecordBatchReader)
    mode: String,
) -> PyResult<Bound<'a, PyAny>> {
    let batches = ArrowArrayStreamReader::from_pyarrow_bound(&data)?;  // ★ C Data Interface, 零拷贝
    let mut op = self_.inner_ref()?.add(batches);
    if mode == "append" {
        op = op.mode(AddDataMode::Append);
    } else if mode == "overwrite" {
        op = op.mode(AddDataMode::Overwrite);
    } else {
        return Err(PyValueError::new_err(format!("Invalid mode: {}", mode)));
    }
    future_into_py(self_.py(), async move {
        op.execute().await.infer_error()?;
        Ok(())
    })
}
```

对比 Node 的 `ipc_file_to_batches(buf.to_vec())`（要解析 IPC 字节流 + 拷贝），Python 这里 `from_pyarrow_bound(&data)` 是**直接共享 pyarrow 对象底层那块 Arrow 内存的指针**，没有序列化、没有拷贝。这是两端最重要的性能差异，§5 专门讲。

### 3.4 同步 + 异步两套 Python API

Python 把"异步内核"同时暴露成**同步**和**异步**两套用户 API（都在 `python/python/lancedb/table.py`）：

- **异步 `AsyncTable`**（`table.py:2785`）：持有 `self._inner`（`table.py:2860`，就是 `#[pyclass]` 那个原生 Table），方法直接 `await`，例如 `add` 在 `table.py:3092`：`await self._inner.add(data, mode or "append")`，`count_rows` 在 `table.py:2917`：`return await self._inner.count_rows(filter)`。**这条路径不经过任何后台线程**——原生方法本就返回 awaitable，直接 await 即可。

- **同步 `LanceTable`**（`table.py:1425`）：处处用 `LOOP.run(self._table.xxx())`，例如 `count_rows` 在 `table.py:1599`：`return LOOP.run(self._table.count_rows(filter))`，`schema` 在 `table.py:1505`：`return LOOP.run(self._table.schema())`。这个 `LOOP.run` 就是把异步内核"阻塞化"的桥，§4 讲。

> **比喻**：同一个后厨，开了两个窗口。异步窗口（`AsyncTable`）：顾客拿到取餐小票（awaitable）自己 `await` 等叫号。同步窗口（`LanceTable`）：顾客把订单递进后台那个永不打烊的传菜窗口，站着等菜出来——他根本不知道后厨是异步运转的。

---

## 4. async 跨 FFI：Rust 是 async 的，宿主语言怎么驱动？

这是 FFI 最烧脑的一环。核心 `lancedb::Table` 的方法都是 `async`，要靠一个 **tokio runtime** 来真正"跑"这些 future。问题是：JS 和 Python 各有自己的事件循环，怎么和 Rust 的 tokio 对接？两端哲学完全不同。

### 4.1 Node：napi 的 `"async"` feature 全自动

Node 侧的 async 能力来自一个 Cargo feature（`nodejs/Cargo.toml:22-25`）：

```toml
napi = { version = "2.16.8", default-features = false, features = [
    "napi9",
    "async"          # ★ 就是它: 开启 async fn -> JS Promise 的自动转换
] }
```

开了 `"async"` feature 后，**`pub async fn` 会被 napi 的过程宏自动改写**成一个返回 JS `Promise` 的函数。程序员"看不见" runtime——napi 内部自己管理一个 tokio runtime，把 future 丢进去跑，跑完了 resolve 那个 Promise。注意 `nodejs/Cargo.toml` **没有直接依赖 tokio**，runtime 完全由 napi 托管。

TS 端就直接 `await` 这个 native Promise（见 §2.4 的 `await this.inner.add(...)`）。**JS 天生有 Promise**，所以 Node 侧不需要任何额外桥接层。

### 4.2 Python：`future_into_py` 手动包，外加一个后台线程

Python 侧分两步走。

**第一步（async API 用）：`future_into_py` 把 Rust future 包成 Python awaitable。** 这个函数来自 `pyo3-async-runtimes`（`python/src/table.rs:20` 的 `use pyo3_async_runtimes::tokio::future_into_py;`），依赖在 `python/Cargo.toml:21-24`：

```toml
pyo3-async-runtimes = { version = "0.23", features = [
    "attributes",
    "tokio-runtime",     # ★ 用 tokio 作为驱动 Rust future 的 runtime
] }
```

它做的事：把 `async move { ... }` 这个 Rust future 交给 pyo3-async-runtimes 管理的 tokio runtime 去跑，**同时返回一个 Python awaitable 对象**，让 Python 的 `asyncio` 能 `await` 它。这就是为什么 §3.2 里每个方法都是"同步 `fn` + `future_into_py(py, async move {...})`"的结构。`AsyncTable` 直接 `await` 这个 awaitable 即可。

**第二步（sync API 用）：一个常驻后台线程跑 asyncio loop。** 同步用户不能 `await`，怎么办？`python/python/lancedb/background_loop.py:8-28`（逐字摘录）给出了答案：

```python
class BackgroundEventLoop:
    """
    A background event loop that can run futures.
    Used to bridge sync and async code, without messing with users event loops.
    """
    def __init__(self):
        self.loop = asyncio.new_event_loop()
        self.thread = threading.Thread(
            target=self.loop.run_forever,            # 后台线程里跑一个永不停的 asyncio loop
            name="LanceDBBackgroundEventLoop",
            daemon=True,                             # daemon: 主程序退出时它自动消失
        )
        self.thread.start()

    def run(self, future):
        # 把 coroutine 扔进后台 loop, 然后 .result() 阻塞等待结果
        return asyncio.run_coroutine_threadsafe(future, self.loop).result()

LOOP = BackgroundEventLoop()                         # 全局单例
```

`LanceTable` 的每个同步方法（§3.4）调的 `LOOP.run(...)`，就是把那个 awaitable 扔进这个后台 loop 跑、然后 `.result()` 阻塞等结果。注释写得很直白："bridge sync and async code, **without messing with users event loops**"——用独立线程跑独立 loop，就不会污染用户自己的事件循环（比如用户在 Jupyter / FastAPI 里已经有一个 loop 在跑了）。

### 4.3 两端对照

```
                Rust future (核心 async 方法)
                        │
        ┌───────────────┴────────────────┐
        ▼                                 ▼
   ── Node ──                        ── Python ──
   napi "async" feature             future_into_py(py, fut)
   自动改写 async fn                  手动包成 Python awaitable
        │                                 │
        ▼                          ┌──────┴───────┐
   返回 JS Promise              异步API           同步API
        │                       直接 await     LOOP.run(awaitable)
        ▼                          │           = run_coroutine_threadsafe
   TS: await this.inner.x()        │             (后台线程 asyncio loop)
                                   ▼              + .result() 阻塞
                              asyncio await
```

> **比喻**：`future_into_py` / napi 的 async 转换 = 把还在炒的锅交给后厨的定时器（tokio runtime）看着，先递给顾客一张取餐小票（`Promise`/awaitable）；炒好了自动叫号，顾客不用站灶台前干等。`background_loop` 的常驻线程 = 餐厅角落那个永不打烊的传菜窗口，同步顾客把订单递进去（`run_coroutine_threadsafe`）、站窗口前等菜（`.result()` 阻塞），既不用自己进后厨，也不打扰别桌的流水线。

> **关键洞察**：Node 没有这层 `background_loop`——因为 JS **天生异步**，连"同步 API"都不存在，TS 层一律 `await`。Python 因为有大量同步用户（脚本、notebook），才需要额外搭一个后台 loop 把异步内核"假装成"同步。这是两个语言生态的根本差异在绑定层的投影。

---

## 5. Arrow 作为跨语言"中立餐盘"：数据怎么近乎零拷贝跨边界

呼应 `02`：Arrow 是一套**与语言无关的列式内存格式**。Python 的 pyarrow、JS 的 Arrow.js、Rust 的 arrow crate，**底层内存布局完全一致**。这意味着理论上可以"同一块内存，三种语言都能读"，不用拷贝。但两端实际选择了不同策略。

### 5.1 Python：Arrow C Data Interface（真零拷贝）

Python 侧靠 arrow crate 的 `pyarrow` feature（`python/Cargo.toml:17`）：

```toml
arrow = { version = "54.1", features = ["pyarrow"] }
```

**入参**（pyarrow → Rust）：`add` 用 `ArrowArrayStreamReader::from_pyarrow_bound(&data)`（`python/src/table.rs:125`），`merge_insert` 同理（`python/src/table.rs:399`）。这走的是 **Arrow C Data Interface**——通过一组约定好的 C 结构体（`ArrowArray`/`ArrowSchema`）**直接传递底层数据缓冲区的指针**，Rust 拿到的是和 Python 同一块内存，没有复制。

**出参**（Rust → pyarrow）：`schema` 用 `schema.to_pyarrow(py)`（`python/src/table.rs:116`）。`DataType` 入参也走同一机制：`DataType::from_pyarrow_bound`（`python/src/table.rs:496`）。

### 5.2 Node：Arrow IPC buffer（序列化 + 拷贝）

Node 侧不走 C Data Interface，而是把 Arrow 数据**序列化成 IPC 文件格式的字节流**，以 napi 的 `Buffer` 传递。

**入参**：`add` 收 `buf: Buffer`，用 `ipc_file_to_batches(buf.to_vec())` 解码（`nodejs/src/table.rs:79`）。`.to_vec()` 是一次明确的拷贝。

**出参**：`schema` 把空 batch 用 `FileWriter::try_new` 写成 IPC，再 `Buffer::from(writer.into_inner())` 返回（`nodejs/src/table.rs:65-75`）。

为什么 Node 选更"重"的 IPC 路线？因为 napi 的 `Buffer` 模型（一块 JS 可见的字节缓冲）比直接对接 Arrow C 指针**更简单、更稳妥**——序列化虽有开销，但避免了手动管理跨语言裸指针的复杂度和风险。

### 5.3 一图对照

```
            同一份 RecordBatch
                  │
     ┌────────────┴─────────────┐
     ▼                          ▼
  ── Python ──               ── Node ──
  C Data Interface           Arrow IPC buffer
  传指针, 共享内存             序列化成字节流 + Buffer + to_vec()
  ✅ 零拷贝                    ⚠️ 至少一次拷贝
  from_pyarrow_bound          ipc_file_to_batches
  to_pyarrow                  batches_to_ipc_file
```

> **比喻**：Arrow C Data Interface = 直接把**同一个餐盘**端过去（Python，零拷贝）。Arrow IPC buffer = 把菜**拍照打印成菜单**再让对方照着重做一份（Node，序列化 + 拷贝）。两者都能上菜，但前者省一次搬运。这是 Python 侧大数据吞吐通常更快的底层原因之一。

---

## 6. 错误跨边界：`lancedb::Error` → JS Error / Python Exception

Rust 用 `Result<T, E>` 表达错误，宿主语言用异常。绑定层必须把 `lancedb::Error` 翻译过去。两端风格差异很大：Node **粗放**（拍平成字符串），Python **精细**（按变体映射到地道异常类）。

### 6.1 Node：拍平成一条多行字符串

`nodejs/src/error.rs:6-15` 定义了一个扩展 trait `NapiErrorExt`，给 `Result<T, lancedb::Error>` 加了 `.default_error()` 方法（§2.3 见过）。它内部调 `convert_error`（`nodejs/src/error.rs:17-32`）：

```rust
pub fn convert_error(err: &dyn std::error::Error) -> napi::Error {
    let mut message = err.to_string();
    let mut cause = err.source();          // 取下一层 cause
    let mut indent = 2;
    while let Some(err) = cause {           // 沿 cause 链逐层往下
        let cause_message = format!("Caused by: {}", err);
        message.push_str(&indent_string(&cause_message, indent));  // 缩进拼进同一条字符串
        cause = err.source();
        indent += 2;
    }
    napi::Error::from_reason(message)       // 整条链 -> 一个 JS Error, 只剩 message
}
```

结果：JS 只拿到**一个 `Error`，message 里是缩进排版的整条 cause 链**。能看，但程序上无法区分错误类型——`catch` 到的永远是同一种 `Error`。

> **Rust 知识点 · `err.source()` 与 cause 链**：Rust 的 `std::error::Error` trait 有个 `source()` 方法返回"导致本错误的上一层错误"。一连串 `source()` 就是 cause 链（类比 Java 的 `getCause()`）。`convert_error` 在这里把整条链拍扁成一段文本。

### 6.2 Python：按 `LanceError` 变体映射到不同异常类

Python 侧 `PythonErrorExt::infer_error`（`python/src/error.rs:24-108`）讲究得多——它 `match` `LanceError` 的枚举变体，**分流到不同的 Python 异常类型**（`python/src/error.rs:28-36` 节选）：

```rust
Err(err) => match err {
    LanceError::InvalidInput { .. }
    | LanceError::InvalidTableName { .. }
    | LanceError::TableNotFound { .. }
    | LanceError::Schema { .. }
    | LanceError::TableAlreadyExists { .. } => self.value_error(),     // -> PyValueError
    LanceError::CreateDir { .. } => self.os_error(),                   // -> OSError
    LanceError::ObjectStore { .. } => Err(PyIOError::new_err(err.to_string())),         // -> IOError
    LanceError::NotSupported { .. } => Err(PyNotImplementedError::new_err(err.to_string())), // -> NotImplementedError
    LanceError::Http { .. } => { /* 构造 lancedb.remote.errors.HttpError + 接 __cause__ 链 */ }
    LanceError::Retry { .. } => { /* 构造 RetryError + 接 __cause__ 链 */ }
    _ => self.runtime_error(),                                         // 兜底 -> RuntimeError
}
```

对于 `Http`/`Retry`，它甚至 `py.import("lancedb.remote.errors")` 拿到 Python 自定义的 `HttpError`/`RetryError` 类、`call1((...))` 构造实例、并用 `setattr("__cause__", ...)` **还原整条 cause 链**（`python/src/error.rs:39-104`、`123-142` 的 `http_from_rust_error`）。

这样 Python 用户能写出地道的精确捕获：

```python
try:
    tbl.add(bad_data)
except ValueError:        # InvalidInput / Schema 等会到这
    ...
except NotImplementedError:  # NotSupported 会到这
    ...
```

> **对照表 · 错误风格**：
>
> | | Node | Python |
> |---|---|---|
> | 入口 | `.default_error()` | `.infer_error()` |
> | 策略 | 拍平 cause 链成一条字符串 | 按变体映射到不同异常类 |
> | 用户能区分类型吗 | 否（永远是 `Error`） | 能（`ValueError`/`IOError`/…） |
> | cause 链 | 进 message 文本 | 重建为 `__cause__` 链 |
> | 代码量 | ~30 行 | ~120 行 |

> **为什么差这么多？** Python 生态强烈依赖"按异常类型分支处理"（`except SomeError`），所以值得花 120 行精细映射。JS 生态里 `catch (e)` 后看 `e.message` 是更常见的范式，napi 也没那么方便构造自定义 JS 错误子类，于是选了拍平方案。**这不是谁偷懒，是两个生态的错误处理文化不同。**

---

## 7. 同一方法三处对照：`count_rows` 与 `add`

把一个核心方法在三层的样子并排放，FFI 的"薄壳"本质一目了然。

### `count_rows`

| 层 | 文件:行 | 关键代码（精简） |
|---|---|---|
| **Rust 核心** | `rust/lancedb/src/table.rs`（见 `09`） | `pub async fn count_rows(&self, filter: Option<String>) -> Result<usize>` —— 转发给 `self.inner`（`BaseTable`） |
| **Node 绑定** | `nodejs/src/table.rs:94-101` | `pub async fn count_rows(&self, filter: Option<String>) -> napi::Result<i64>`：`inner_ref()?.count_rows(filter).await.map(\|v\| v as i64).default_error()` |
| **Python 绑定** | `python/src/table.rs:169-178` | `pub fn count_rows(self_: PyRef, filter) -> PyResult<Bound<PyAny>>`：`let inner = inner_ref()?.clone(); future_into_py(py, async move { inner.count_rows(filter).await.infer_error() })` |

差异点：Node 直接 `async fn` + `i64` 出参 + `default_error`；Python 同步 `fn` + `clone` + `future_into_py` + `infer_error`。**业务（真正数行）只在 Rust 核心那一行，两侧都只是转发。**

### `add`（含 Arrow 入参）

| 层 | 文件:行 | Arrow 入参方式 | async 方式 |
|---|---|---|---|
| **Node 绑定** | `nodejs/src/table.rs:77-92` | `ipc_file_to_batches(buf.to_vec())`（IPC，有拷贝） | `pub async fn`，napi 自动 Promise |
| **Python 绑定** | `python/src/table.rs:120-139` | `ArrowArrayStreamReader::from_pyarrow_bound(&data)`（C Data Interface，零拷贝） | 同步 `fn` + `future_into_py` |
| **TS 用户层** | `nodejs/lancedb/table.ts:492-498` | `fromDataToBuffer(data,...)` 先编码成 buffer | `await this.inner.add(...)` |
| **Py 同步用户层** | `python/python/lancedb/table.py:3092`（async）/ 经 `LOOP.run` 的同步包装 | pyarrow 对象直接传 | `await`（async）/ `LOOP.run`（sync） |

---

## 8. 给贡献者：改了核心方法，两侧绑定要同步改什么

假设你在 `rust/lancedb/src/table.rs` 给 `BaseTable` / `Table` **加了一个新方法**，或改了某方法签名。要让 Python/Node 用户也能用，得在多处同步改动。**这是一张清单**（详细 build/test 流程见 `18`）：

### 8.1 改 Node 侧

1. 在 `nodejs/src/table.rs` 的 `#[napi] impl Table`（:36-37）里加一个对应方法：解 `inner_ref()?` → 调核心 → `.default_error()`。带 `#[napi(catch_unwind)]`。
2. 若有 Arrow 数据进出 → 用 `ipc_file_to_batches` / `batches_to_ipc_file`（参考 `add`/`iterator.rs`）。
3. 若返回的是新类型 → 可能要新建一个 `#[napi]` struct（参考 `nodejs/src/iterator.rs` 的 `RecordBatchIterator`）。
4. 在 TS 层 `nodejs/lancedb/table.ts` 加用户友好的封装方法（编码入参 / 解码出参，参考 `add` :492、`query.ts:43` 的 `tableFromIPC`）。
5. 重新生成 TS 类型声明 + 编译（见 §8.3）。

### 8.2 改 Python 侧

1. 在 `python/src/table.rs` 的 `#[pymethods] impl Table`（:96-97）里加方法：**注意是同步 `fn` 返回 `PyResult<Bound<PyAny>>`**，结构是 `let inner = inner_ref()?.clone(); future_into_py(py, async move { inner.xxx().await.infer_error() })`。
2. Arrow 数据 → 用 `from_pyarrow_bound` / `to_pyarrow`。
3. 返回新类型 → 新建 `#[pyclass]` struct（参考 `python/src/arrow.rs`），**并且必须在 `python/src/lib.rs` 的 `#[pymodule] fn _lancedb` 里手动 `m.add_class::<新类>()` 注册**（`python/src/lib.rs:30-37`）——漏注册，Python 那边 import 不到，是最常见的低级错误。
4. 在 `python/python/lancedb/table.py` 给**两套** API 各加一个方法：`AsyncTable`（直接 `await self._inner.新方法()`）和 `LanceTable`（`LOOP.run(self._table.新方法())`）。

> **两端模块注册机制不同（容易踩）**：
> - **Node**：靠过程宏全自动——文件里写了 `#[napi]` 就自动导出，`#[napi::module_init]`（`nodejs/src/lib.rs:57`）只做日志初始化，**不用手动登记类**。
> - **Python**：**必须手动**在 `_lancedb` 里 `m.add_class::<T>()` 一个个登记（`python/src/lib.rs:24-42`）。漏一个 → `ImportError`。

### 8.3 重新构建与测试

| 端 | 构建命令 | 来源 |
|---|---|---|
| Node | `napi build --platform --no-const-enum --dts ../lancedb/native.d.ts --js ../lancedb/native.js lancedb`（即 `npm run build:debug`） | `nodejs/package.json:75` |
| Node（全量） | `npm run build`（= build:debug + tsc + 拷 .node） | `nodejs/package.json:77` |
| Python | `maturin develop --extras tests,dev,embeddings`（即 `make develop`，编译 Rust 扩展并装进当前 venv） | `python/Makefile:8` |
| Python 测试 | `pytest python/tests -vv --durations=10 -m "not slow and not s3_test"`（即 `make test`） | `python/Makefile:36` |

> 完整的 build 工具链、lint 规则、三端协同发版流程，全部留给收官的**下一篇 `18 贡献你的第一行代码`**。

---

## 附 · 流式结果跨边界（两端的迭代器协议）

查询结果是流式的（逐 batch，不一次性返回大数组）。两端各自实现了"边界上的迭代器"，原理值得单独看一眼。

**Node**：`RecordBatchIterator`（`nodejs/src/iterator.rs:11-14`）持有 `SendableRecordBatchStream`，`next()` 每次取一个 batch 编码成 IPC buffer（`nodejs/src/iterator.rs:22-35`）：

```rust
#[napi(catch_unwind)]
pub async unsafe fn next(&mut self) -> napi::Result<Option<Buffer>> {
    if let Some(rst) = self.inner.next().await {
        let batch = rst.map_err(...)?;
        batches_to_ipc_file(&[batch]).map(|buf| Some(Buffer::from(buf)))  // 序列化跨边界
    } else {
        Ok(None)    // None = 流结束信号
    }
}
```

TS 端消费（`nodejs/lancedb/query.ts:39-47`）：`const n = await this.inner.next();` 拿到 buffer，`n == null` 即结束，否则 `tableFromIPC(n)` 解码、断言只有一个 batch。

> **Rust 陷阱 · `pub async unsafe fn next(&mut self)`**：这里的 `unsafe` **不是因为代码危险**，而是 napi 对"`async` + `&mut self`"组合有安全约束——多个 JS 调用理论上可能并发持有 `&mut self`，napi 要求程序员用 `unsafe` 显式承诺"我不会同时调它"。

**Python**：`RecordBatchStream`（`python/src/arrow.rs:20-24`）持有 `Arc<tokio::sync::Mutex<SendableRecordBatchStream>>`，实现 Python 异步迭代协议 `__aiter__`/`__anext__`（`python/src/arrow.rs:43-58`）：

```rust
pub fn __anext__(self_: PyRef<'_, Self>) -> PyResult<Bound<'_, PyAny>> {
    let inner = self_.inner.clone();
    future_into_py(self_.py(), async move {
        let inner_next = inner.lock().await.next().await
            .ok_or_else(|| PyStopAsyncIteration::new_err(""))?;  // 流尽 -> Python StopAsyncIteration
        Python::with_gil(|py| inner_next.infer_error()?.to_pyarrow(py))  // batch 零拷贝交付
    })
}
```

同步路径（`python/python/lancedb/table.py:2344-2362`）再用一个 generator 反复 `LOOP.run(async_iter.__anext__())` 直到 `StopAsyncIteration`，包成 `pa.RecordBatchReader.from_batches(...)`。

> **比喻**：流尽时的"收摊信号"，Node 用返回 `None` 喊（`Option::None`），Python 用抛 `PyStopAsyncIteration` 喊——这是 Python 异步迭代协议约定的结束信号。**约定不同，意思一样：传菜车空了。** 同样地，Python 的 batch 用 `to_pyarrow` 零拷贝交付，Node 仍走 IPC 序列化——和 §5 一致。

---

## 9. 本篇出现的 Rust 语法点 · 速查

| 语法 / 概念 | 一句话 | 出现处 |
|---|---|---|
| `#[napi]` / `#[pyclass]` | 过程宏：把 Rust struct 导出成 JS class / Python class | `nodejs/src/table.rs:20`、`python/src/table.rs:62` |
| `#[napi(catch_unwind)]` | FFI 边界捕获 Rust panic，转成 JS 异常（防进程崩溃） | `nodejs/src/table.rs:77/94/103…` |
| `Option<T>` + `.take()` | `take()` 移走值留 `None`；`close()` 借此 drop 核心 Table | `nodejs/src/table.rs:60`、`python/src/table.rs:109` |
| `.as_ref()` + `ok_or_else` | `Option<T>`→`Option<&T>`，`None` 转成 `Err`（解包 + 已关闭检查） | `nodejs/src/table.rs:30`、`python/src/table.rs:90` |
| `pub async fn`（napi） | 开 `"async"` feature 后自动变返回 JS `Promise` 的函数 | `nodejs/src/table.rs:78`、`nodejs/Cargo.toml:24` |
| `future_into_py(py, async move{})` | pyo3：把 Rust future 包成 Python awaitable | `python/src/table.rs:175` |
| `self_: PyRef<'_, Self>` | pyo3 里带 GIL token 的"self"，`self_.py()` 取 token | `python/src/table.rs:170` |
| 先 `clone()` 再 `async move` | future 延迟执行，不能借 `self_`，把 `Arc` clone 进去 | `python/src/table.rs:174` |
| `Bound<'_, PyAny>` | pyo3 0.23：带 GIL 生命周期的 Python 对象引用 | `python/src/table.rs:124` |
| extension trait（`.default_error()`/`.infer_error()`） | 给外部类型 `Result<T,E>` 加方法的唯一合法途径，需 `use` 进作用域 | `nodejs/src/error.rs:6`、`python/src/error.rs:13` |
| `err.source()` | 取 cause 链下一环（类比 Java `getCause()`） | `nodejs/src/error.rs:21` |
| `match` 枚举变体 | 按 `LanceError` 变体分流到不同 Python 异常 | `python/src/error.rs:28` |
| `pub async unsafe fn next(&mut self)` | `unsafe` 是 napi 对 "async + &mut self" 的安全承诺，非代码危险 | `nodejs/src/iterator.rs:23` |
| `ArrowArrayStreamReader::from_pyarrow_bound` | Arrow C Data Interface，零拷贝读 pyarrow 对象 | `python/src/table.rs:125` |
| `#[napi::module_init]` / `#[pymodule]` | 模块入口：Node 自动注册，Python 需手动 `add_class` | `nodejs/src/lib.rs:57`、`python/src/lib.rs:24` |

---

## 10. 动手验证（建议亲手做一遍）

1. **找对称**：并排打开 `nodejs/src/table.rs:94-101` 和 `python/src/table.rs:169-178`（两个 `count_rows`）。逐行对照，标出"哪几行做的是同一件事、用了不同语法"。这能让你彻底吃透"薄壳四件事"。
2. **抓 close 的本质**：在两端各搜 `inner.take()`（`nodejs/src/table.rs:60`、`python/src/table.rs:109`）。问自己：为什么 struct 里 `name` 要单独存一份？（提示：`take()` 之后 `inner` 没了，报错信息还想带表名。）
3. **跟一次 Arrow 入参**：从 TS 的 `nodejs/lancedb/table.ts:496` 的 `fromDataToBuffer` 出发 → 追到 Rust 的 `nodejs/src/table.rs:79` 的 `ipc_file_to_batches`。再对照 Python 的 `python/src/table.rs:125` 的 `from_pyarrow_bound`。亲眼确认"Node 序列化、Python 零拷贝"。
4. **验 async 桥接**：读 `python/python/lancedb/background_loop.py` 全文（只有 28 行），再看 `python/python/lancedb/table.py:1599` 的 `LOOP.run(self._table.count_rows(filter))` 与 `table.py:2917` 的 `await self._inner.count_rows(filter)`。对比同步/异步两条路径。
5. **故意漏注册（破坏性实验，改完记得还原）**：把 `python/src/lib.rs:31` 的 `m.add_class::<Table>()?;` 注释掉，`make develop` 重新编译，然后 `python -c "import lancedb"` 试图用 Table——观察报什么错。这能让你牢记"Python 端必须手动注册类"。

---

## 11. 小结 & 全套文档回顾

本篇拆穿了三层架构的最后一道缝：

- **绑定层 = 同声传译**：napi-rs / pyo3 把 Rust `lancedb::Table` 包成 JS / Python 对象。**两端 struct 完全对称**，都持有 `Option<lancedb::Table>`，靠 `take()` 支持 `close()`。
- **每个方法只做四件事**：解 `inner_ref()` → 转 Arrow 参 → 驱动 async → 转错误。**业务一行不写**，全在核心。
- **三处关键差异**：① async——Node 靠 napi `"async"` feature 自动变 Promise，Python 靠 `future_into_py` + `background_loop` 后台线程同时支撑同步/异步两套 API；② Arrow——Python 用 C Data Interface 零拷贝，Node 用 IPC buffer 序列化；③ 错误——Node 拍平成字符串，Python 按变体精细映射到地道异常。
- **模块注册**：Node 全自动，Python 必须手动 `add_class`。
- 改核心方法 → 两侧绑定要按 §8 清单同步改，构建/测试细节见 `18`。

### 全套 18 篇全景索引

| # | 标题 | 一句话 |
|---|---|---|
| **Part 0 · 地基** | | |
| `00` | 全景地图 | 森林视角 + 阅读路线 + 三层架构 |
| `01` | Rust 垫脚石 | 所有权 / trait / `dyn` / async / `Arc` / builder |
| `02` | 三大基石 | Arrow · DataFusion · lance 各"组装"了什么 |
| **Part 1 · 请求的一生** | | |
| `03` | 连接建立链路 | `connect()` → `Database` trait → `ListingDatabase` |
| `04` | 写入链路 | `add()`：`IntoArrow`→`RecordBatch`→落盘 |
| `05` | 建索引链路 | `create_index()` → 向量/标量 → lance |
| `06` | 查询链路 | `query()` → 三层 query trait → DataFusion plan |
| **Part 2 · 模块深潜** | | |
| `07` | Connection 与 Builder 体系 | `connection.rs`，const generics builder |
| `08` | Database 与 Catalog 抽象层 | `database.rs`、`catalog.rs` |
| `09` | **Table 解剖·上** | `BaseTable` 契约 + `Table` 门面 + `NativeTable` 骨架（本篇基石） |
| `10` | Table 解剖·下 | 一致性锁 / merge / 时间旅行 / optimize |
| `11` | Query 体系全解 | 三层 trait + `Query`/`VectorQuery` |
| `12` | Index 体系 | 向量 5 种 + 标量 4 种 builder |
| `13` | Rerankers 与混合搜索 | `rrf.rs`、`query/hybrid.rs` |
| `14` | Embeddings | 注册表机制 + 三家实现 |
| `15` | Remote | `RemoteTable` 如何也实现 `BaseTable` |
| `16` | 横切关注点 | 错误模型 / 存储抽象 / 工具 |
| **Part 3 · FFI 与贡献** | | |
| `17` | **FFI 边界全解** | 一个方法如何穿过 napi-rs / pyo3（本篇） |
| `18` | 贡献你的第一行代码 | build/test、风格、改核心如何同步三端绑定 |

读到这里，你已经从"森林"（`00`~`02`）走过"一个请求的一生"（`03`~`06`），钻进了每个模块（`07`~`16`），最后看清了核心如何被绑定层包给 Python/Node（`17`）。**只差最后一步——`18` 教你把环境搭起来、把第一个 PR 提出去。**
