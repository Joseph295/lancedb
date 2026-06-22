# LanceDB 核心引擎源码深度培训（中文 · 内部贡献者向）

> 目标读者：**懂至少一门编程语言（如 Python/JS）、但不熟悉 Rust，希望给 `rust/lancedb` 核心引擎贡献代码**的开发者。
>
> 这套材料的野心是让你**吃透整个核心引擎、不漏关键细节**——既见森林（端到端执行链路），也见树木（逐模块、深到 trait 契约与 struct 字段、控制流的源码解剖）。
>
> 所有论断都回到源码真实行号（形如 `table.rs:408`），可点击跳转。

## 怎么读这套材料

1. **先建心智模型**：按顺序读 Part 0（`00`→`01`→`02`）。如果你完全不懂 Rust，`01 Rust 垫脚石`是不能跳的。
2. **再走一遍链路**：读 Part 1（`03`→`06`），跟着一个请求从 API 入口走到磁盘，建立"森林"视角。
3. **最后逐棵钻**：Part 2 的模块深潜（`07`→`16`）可按需跳读；想贡献某模块就精读那一篇。
4. **准备动手**：Part 3（`17`/`18`）讲 FFI 边界与贡献流程。

每篇都配有：trait 契约 / struct 字段级讲解、真实行号、关键路径的 ASCII 调用图、Rust 惯用法的比喻、结尾的"动手验证"小练习。

## 全景：LanceDB 是一座为 AI 设计的智能图书馆

```
   用户代码 (Python / Node.js / Java)
        │   通过 FFI 绑定 (pyo3 / napi-rs / jni)
        ▼
 ┌──────────────────────────────────────────┐
 │  核心引擎  rust/lancedb   ← 你贡献的主战场   │
 │  connection → database → table → query    │
 │  + index / embeddings / remote / catalog  │
 │  + rerankers / io / error / arrow / ipc   │
 └──────────────────────────────────────────┘
        │   依赖（"组装"而非"重造"）
        ▼
   lance(磁盘列存格式) · Arrow(内存列格式) · DataFusion(查询执行引擎)
```

核心心智模型：**LanceDB 自己不发明存储格式，它站在 `lance`/`Arrow`/`DataFusion` 三个巨人肩上做"组装与编排"。** 理解它"组装了什么"比"实现了什么"更重要。

## 文档清单（18 篇 · 4 部分）

### Part 0 · 地基与心智模型
| 编号 | 标题 | 解剖的源码 |
|---|---|---|
| `00` | 全景地图（森林视角 + 阅读路线） | `lib.rs`、整体目录 |
| `01` | Rust 垫脚石（所有权 / trait / `dyn` / async / `Arc` / builder） | 全用库内真实代码举例 |
| `02` | 三大基石：Arrow · DataFusion · lance —— 到底"组装"了什么 | 依赖接口层 |

### Part 1 · 请求的一生（端到端、逐行级链路）
| 编号 | 标题 | 解剖的源码 |
|---|---|---|
| `03` | 连接建立链路 | `connect()` → `ConnectBuilder` → `Database` trait → `ListingDatabase` |
| `04` | 写入链路 | `add()`：`IntoArrow`→`RecordBatch`→落盘；`data/sanitize`、`arrow.rs`、`ipc.rs` |
| `05` | 建索引链路 | `create_index()` → `IndexBuilder` → 向量/标量 → lance |
| `06` | 查询链路 | `query()` → 三层 query trait → `create_plan()` → DataFusion `ExecutionPlan` |

### Part 2 · 模块深潜（逐个解剖，覆盖全部模块）
| 编号 | 标题 | 解剖的源码 |
|---|---|---|
| `07` | Connection 与 Builder 体系（含 `const generics` builder） | `connection.rs` |
| `08` | Database 与 Catalog 抽象层 | `database.rs`、`catalog.rs`、`*/listing.rs` |
| `09` | **Table 解剖·上**：`BaseTable` 契约 + `Table` + `NativeTable` 骨架 | `table.rs` |
| `10` | Table 解剖·下：一致性锁 / merge / 时间旅行 / optimize / schema 演化 | `table.rs`、`table/dataset.rs`、`table/merge.rs` |
| `11` | Query 体系全解：三层 trait + `Query`/`VectorQuery` + `*Request` | `query.rs` |
| `12` | Index 体系：向量 5 种 + 标量 4 种 builder | `index.rs`、`index/vector.rs`、`index/scalar.rs` |
| `13` | Rerankers 与混合搜索 | `rerankers/rrf.rs`、`query/hybrid.rs` |
| `14` | Embeddings：注册表机制 + 三家实现 | `embeddings.rs` 及子模块 |
| `15` | Remote：`RemoteTable` 如何也实现 `BaseTable` | `remote/` |
| `16` | 横切关注点：错误模型 / 存储抽象 / 工具 | `error.rs`、`io/object_store.rs`、`utils.rs`、`arrow.rs`、`ipc.rs`、`data/` |

### Part 3 · FFI 与贡献
| 编号 | 标题 | 解剖的源码 |
|---|---|---|
| `17` | FFI 边界全解：一个方法如何穿过 pyo3 / napi-rs 暴露给 Python/Node | `python/`、`nodejs/src/` |
| `18` | 贡献你的第一行代码：build/test、风格、改核心如何同步三端绑定 | `CONTRIBUTING.md` 等 |

## 进度

- [x] 规格 / 索引（本文件）
- [x] `09 Table 解剖·上`（质量样板篇，已验收定调）
- [x] Part 0 地基组：`00 全景地图` / `01 Rust 垫脚石` / `02 三大基石`
- [x] Part 1 请求的一生：`03 连接` / `04 写入` / `05 建索引` / `06 查询`
- [x] Part 2 核心三件：`10 Table下` / `11 Query` / `12 Index`
- [x] Part 2 外围六篇：`07`/`08`/`13`/`14`/`15`/`16`
- [x] Part 3 FFI 与贡献：`17 FFI 边界全解` / `18 贡献你的第一行代码`

**✅ 全套 18 篇已完成**（每篇均经独立 agent 回查源码行号、逐处订正）。
