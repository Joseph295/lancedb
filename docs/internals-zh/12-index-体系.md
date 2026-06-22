# 12 · Index 体系：从"索引点菜单"到 lance 工单

> **本篇解剖的源码**：`rust/lancedb/src/index.rs`（顶层抽象）+ `index/vector.rs`（向量 Builder）+ `index/scalar.rs`（标量/全文 Builder）+ `table.rs` 的 `create_*_index` 私有方法（翻译柜台）+ `remote/table.rs` 的云端分支。
> **覆盖区间**：`index.rs:1-211` 全文、`index/vector.rs:1-370` 全文、`index/scalar.rs:1-86` 全文、`table.rs:1446-2019` 的索引控制流、`remote/table.rs:741-789` 云端分发。
> **前置阅读**：`01 Rust 垫脚石`（trait / `Arc` / `Option` / 宏）、`05 建索引链路`（本篇是它的"模块切面"——05 讲"一跳到下一跳"，本篇讲"index 模块内部怎么搭"）。
> **与哪些篇互补**：与 `09 Table 解剖（上）` 同为"模块深潜"篇（09 讲门面/合同/本地实现，本篇讲索引子系统）；`create_index` 这一跳在 `05 建索引链路` 已走过链路，这里只剖模块自身结构，不重复链路。

---

## 0. 这一篇要回答的问题

读完本篇，你应该能回答：

1. 用户写 `table.create_index(&["vec"], Index::IvfPq(...))` 时，这一串 `Index::IvfPq(...)` 到底是什么数据结构？它怎么变成 lance 看得懂的索引参数？
2. 向量索引五选一（IVF / PQ / HNSW / SQ / Flat）各自解决什么问题？每个 Builder 的 `num_partitions`/`num_sub_vectors`/`m`/`ef_construction` 该填多少？
3. 标量索引四种（BTree / Bitmap / LabelList / FTS）分别适合什么过滤？
4. `Index::Auto` 是怎么"自动"挑索引的？它是个新算法吗？
5. 如果我想给某个向量索引加一个新参数，要改哪几个文件？

**一句话锚点**：整个 index 模块**没有一行索引算法**。它只是一张"索引点菜单"——声明"要建什么索引、用什么参数"，真正的 kmeans / PQ 量化 / HNSW 建图全在 `lance` 与 `lance_index` crate 里。这是"组装而非重造"最纯粹的样例。

把模块骨架画成一张图：

```
              你的代码
                 │  table.create_index(cols, Index::IvfPq(builder))
                 ▼
   ┌───────────────────────────────────┐
   │  Index enum (索引点菜单, index.rs:23)│  9 个变体，每个内嵌一张 Builder 子菜单
   │  Auto │ BTree │ Bitmap │ ... │ IvfPq│  Auto 是唯一不带 Builder 的变体
   └──────────────┬────────────────────┘
                  │  装进
                  ▼
   ┌───────────────────────────────────┐
   │  IndexBuilder (通用选项, index.rs:67)│  parent / index / columns / replace
   │  .replace(bool) → .execute()       │  通用配料单：索引专有参数在内嵌 index 里
   └──────────────┬────────────────────┘
                  │  execute() 仅一行委托 (index.rs:95)
                  ▼
   ┌───────────────────────────────────┐
   │  BaseTable::create_index (合同方法)  │  table.rs:453
   └──────┬───────────────────┬────────┘
          │                   │
          ▼                   ▼
 ┌─────────────────┐  ┌──────────────────┐
 │ NativeTable     │  │ RemoteTable      │
 │ (本地后厨)       │  │ (远程分店)        │
 │ table.rs:1989   │  │ remote/table.rs  │
 │ match→create_*  │  │ :741 序列化成JSON │
 └────────┬────────┘  └──────────────────┘
          │  翻译柜台: Builder 字段 → lance *BuildParams
          ▼
   lance / lance_index (真正的 kmeans / PQ / HNSW / BTree 算法)
```

> **比喻总钥匙**：`Index` enum 是餐厅的**索引点菜单**——9 道菜，每道菜附一张更细的**配料单**（Builder）；`Index::Auto` 是"看着上一道合适的"。`IndexBuilder` 是写在菜单顶部的**通用选项**（要不要覆盖旧索引）。`table.rs` 里的 `create_*_index` 是**翻译柜台**，把中文配料单翻成后厨灶台（lance）看得懂的 `*BuildParams` 工单。本篇就按"先看菜单结构 → 再看每道菜的配料 → 最后看翻译柜台怎么翻"的顺序展开。

---

## 1. 总览：两个 enum + 一个 Builder

index 模块顶层（`index.rs`）只有三个公共主角，加一组反序列化用的内部镜像。先把它们的角色分清，否则后面会混。

### 1.1 `Index`：输入用的"点菜单"（9 个变体）

`index.rs:22-62`：

```rust
#[derive(Debug, Clone)]                 // ① 只 derive 这俩，无 PartialEq
pub enum Index {
    Auto,                               // ② 唯一不带 Builder 的变体
    BTree(BTreeIndexBuilder),           // ③ 其余 8 个都内嵌一张 Builder 子菜单
    Bitmap(BitmapIndexBuilder),
    LabelList(LabelListIndexBuilder),
    FTS(FtsIndexBuilder),
    IvfFlat(IvfFlatIndexBuilder),
    IvfPq(IvfPqIndexBuilder),
    IvfHnswPq(IvfHnswPqIndexBuilder),
    IvfHnswSq(IvfHnswSqIndexBuilder),
}
```

`Index` 是**建索引时的输入**：你想建哪种索引，就选哪个变体，并把配好参数的 Builder 塞进去。`Auto`（`index.rs:24`）是唯一不带任何 Builder 的变体——因为它的意思是"你来挑"，没什么可配的。

> **Rust 知识点：带数据的枚举（enum with data）**。Rust 的 `enum` 不只是一组常量名（不像 C 的 `enum`、不像 JS 的字符串联合）。每个变体可以**自带一个不同类型的字段**：`Index::Auto` 是空的，`Index::IvfPq(IvfPqIndexBuilder)` 内含一个完整 struct。这相当于 TypeScript 的 discriminated union（`{kind:'auto'} | {kind:'ivfpq', builder:...}`），但由编译器强制保证你 `match` 时不漏分支。

### 1.2 `IndexBuilder`：通用选项 + 下单按钮

`index.rs:67-97`：

```rust
pub struct IndexBuilder {
    parent: Arc<dyn BaseTable>,         // ① 目标表（后厨电话），私有
    pub(crate) index: Index,            // ② 上面那张点菜单（含专有参数）
    pub(crate) columns: Vec<String>,    // ③ 要建索引的列
    pub(crate) replace: bool,           // ④ 是否覆盖同名旧索引
}

impl IndexBuilder {
    pub(crate) fn new(parent: Arc<dyn BaseTable>, columns: Vec<String>, index: Index) -> Self {
        Self { parent, index, columns, replace: true }   // ⑤ replace 默认 true，硬编码于此
    }

    pub fn replace(mut self, v: bool) -> Self {  // ⑥ 链式 setter：消费 self、改、还回
        self.replace = v;
        self
    }

    pub async fn execute(self) -> Result<()> {
        self.parent.clone().create_index(self).await    // ⑦ 整个方法体就这一行
    }
}
```

`IndexBuilder` 是用户实际拿在手里的对象（由 `Table::create_index`，`table.rs:761` 返回，详见 `05 建索引链路`）。它只承载**对所有索引类型通用的选项**——目前只有一个 `replace`。**索引专有的参数（分区数、子向量数……）不在这层，而在 `index` 字段内嵌的那张 Builder 子菜单里。**

最关键的一行是 `execute`（`index.rs:95`）：它**不做任何索引逻辑**，只是把整个 `IndexBuilder`（连同里面的 `Index` 子菜单）原样交还给 `BaseTable::create_index`。下单按钮按下去，单子整张递进厨房，菜单本身不炒菜。

> **`replace` 的语义陷阱**（`index.rs:84-88` 文档注释）：默认 `true`，即"覆盖同列同名的旧索引"。若你设成 `false` 且同列同名索引已存在，会**报错**——注释特别强调"即使那个旧索引已经过期（out of date）也照样报错"。贡献者改默认行为时要分清：这个 `true` 是写死在 `new()`（`index.rs:80`）里的。

> **Rust 陷阱：链式 setter 是 `mut self`，不是 `&mut self`**（`index.rs:89`）。`fn replace(mut self, v: bool) -> Self` 取得 `self` 的**所有权**，改完把整个 `self` 还回去。这才能写出 `builder.replace(false).execute()` 的链式调用。代价：调一次后原变量被"移动"走，不能再用。JS/Python 读者容易误以为这是引用链（改的是同一对象），其实每一步都是**移动语义**——把值搬进方法再搬出来。

### 1.3 `IndexType`：输出用的"纯标签"（8 个变体）

注意：`Index`（输入）和 `IndexType`（输出）是**两个不同的 enum**，新手极易混。

`index.rs:99-120`：

```rust
#[derive(Debug, Clone, PartialEq, Deserialize)]   // ① 比 Index 多了 PartialEq + Deserialize
pub enum IndexType {
    #[serde(alias = "IVF_FLAT")] IvfFlat,          // ② serde 别名：兼容多种拼写
    #[serde(alias = "IVF_PQ")]   IvfPq,
    #[serde(alias = "IVF_HNSW_PQ")] IvfHnswPq,
    #[serde(alias = "IVF_HNSW_SQ")] IvfHnswSq,
    #[serde(alias = "BTREE")]    BTree,
    #[serde(alias = "BITMAP")]   Bitmap,
    #[serde(alias = "LABEL_LIST")] LabelList,
    #[serde(alias = "INVERTED", alias = "Inverted")] FTS,   // ③ FTS 在 lance 里叫 INVERTED
}
```

`IndexType` 是**查询已建索引时拿到的标签**——它不带任何 Builder、不带参数，只是个名字。它出现在 `list_indices()` 返回的 `IndexConfig`、`index_stats()` 返回的 `IndexStatistics` 里。它有"三件套"：

| 件 | 位置 | 作用 |
|---|---|---|
| `enum` 定义 | `index.rs:100` | 8 个变体（注意比 `Index` 少 1：没有 `Auto`，因为 `Auto` 不是一种真实索引，只是建索引时的占位意图） |
| `Display` | `index.rs:122-135` | 把变体打印成大写下划线串（`IvfPq` → `"IVF_PQ"`） |
| `FromStr` | `index.rs:137-155` | 反向：从字符串解析回变体，失败返回 `Error::InvalidInput`（`index.rs:150-152`） |

> **Rust 知识点：`#[serde(alias)]` 与 serde 派生宏**。`#[derive(Deserialize)]` 让 serde **自动生成**反序列化代码（无需手写）。`#[serde(alias = "...")]` 允许一个变体匹配多种字符串拼写——这样无论 lance 返回的 JSON 写的是 `"IVF_FLAT"` 还是 `"IvfFlat"` 都能解析。`FTS` 还兼容 lance 内部的历史名 `INVERTED`（`index.rs:118`），因为全文索引在 lance 那层叫"倒排索引"。

> **三个同名 `Index` 别混**（重要陷阱）：
> 1. 本模块 `index.rs:23` 的 `Index`——建索引的输入枚举（带 Builder）。
> 2. 本模块 `index.rs:100` 的 `IndexType`——查询索引的输出标签（纯名字）。
> 3. `index/vector.rs:11` import 的 `lance::table::format::Index`——lance 内部的索引元数据结构（带 `fields`/`uuid`），与前两者**毫不相干**，只是恰好同名。
>
> Rust 允许同名类型共存，靠**模块路径**区分。Python/JS 读者会被重名绊倒——记住：看到 `Index` 先问"哪个模块的"。

### 1.4 索引元数据：一个公共类型 + 三个反序列化镜像

先分清两类东西，**它们的派生宏不一样，别混为一谈**：

**(a) 公共返回类型 `IndexConfig`（`index.rs:158-159`）**——`#[derive(Debug, PartialEq, Clone)]`，注意它**没有** `Deserialize`、也**没有** `#[skip_serializing_none]`，因为它是 `list_indices()` 直接交给用户的"成品"，不参与 JSON 反序列化：

| 类型 | 位置 | 可见性 | 角色 |
|---|---|---|---|
| `IndexConfig` | `index.rs:159` | `pub` | `list_indices()` 的返回元素：`name` + `index_type` + `columns`（注释 `:163-165` 说目前 `columns` 恒为 size 1，复合索引尚未支持） |

**(b) 三个反序列化镜像**——这三个才**全标 `#[skip_serializing_none]` + `#[derive(..., Deserialize)]`**（让值为 `None` 的字段不出现在序列化 JSON 里），专门用来把 lance `index_statistics()` 返回的 JSON 解析成结构体：

| 类型 | 位置 | 可见性 | 角色 |
|---|---|---|---|
| `IndexMetadata` | `index.rs:172` | `pub(crate)` | 单个索引的元数据镜像：`metric_type` / `index_type` / `loss`，全是 `Option`（注释 `:174` 说 index_type 有时在这一层） |
| `IndexStatisticsImpl` | `index.rs:184` | `pub(crate)` | 顶层镜像，直接反序列化 lance `index_statistics()` 的 JSON：`num_indexed_rows`/`num_unindexed_rows`/`indices: Vec<IndexMetadata>`/`index_type`/`num_indices` |
| `IndexStatistics` | `index.rs:195` | `pub` | 由上面镜像加工后、`index_stats()` 暴露给用户的统计快照；`distance_type`/`num_indices`/`loss` 都是 `Option`（`distance_type` 仅向量索引有） |
| `IndexMetadata` | `index.rs:173` | `pub(crate)` | lance JSON 里 `indices` 数组的元素：`metric_type`/`index_type`/`loss` |

> 为什么要 `Impl` 镜像 + 对外结构两套？因为 lance 返回的 JSON 形状不规整（同一字段有时在外层有时在内层，`index.rs:188` 的注释就在吐槽这点）。`*Impl` 负责"照单全收地接住乱 JSON"，对外的 `IndexStatistics` 负责"给用户一个干净规整的结构"。这是把"外部数据的脏"挡在 crate 内部的常见手法。

---

## 2. 向量索引五选一：各解决什么问题（通俗版）

向量索引要解决的核心难题是：**几百万条几百维的向量里，怎么快速找到和查询向量最像的 K 条？** 暴力比对每一条太慢。五种索引是五种"抄近路"的策略，可以单用也可以叠加。先用比喻把它们讲透，下一节再看参数。

### 2.1 IVF —— 倒排分桶（先分区，只搜命中的几桶）

**比喻**：图书馆把书按主题分到几百个书架（partition/分区），每个书架贴一个"主题代表"标签（centroid/质心）。你要找一本书，先看哪几个书架的标签和你的需求最接近，只翻那几个架子，不翻全馆。

- 建索引时用 **kmeans** 算法把所有向量聚成 `num_partitions` 个簇，每簇记一个质心。
- 查询时先比质心挑出最近的几个簇，只在这些簇里细搜。
- 这就是 `IvfFlatIndexBuilder` 的全部：**只分桶，桶里存原始向量不压缩**（`vector.rs:166-176` 文档）。精度最高，但存储和内存开销大。

### 2.2 PQ —— 乘积量化（把每个向量压成几个字节）

**比喻**：把一幅高清照片切成小块，每块用"最接近的一个预设色卡编号"代替。原来每块要存完整像素，现在只存一个编号，体积骤减——代价是有损。

- PQ（Product Quantization）把每个向量切成 `num_sub_vectors` 段子向量，每段用一个小码本编号（`num_bits` 位）替代（`vector.rs:203-222` 文档）。
- 一个 768 维 float32 向量原本 3072 字节，PQ 后可能只剩几十字节。**省内存、加速搜索，代价是精度有损**。
- `IvfPqIndexBuilder` = IVF 分桶 + 桶里存 PQ 压缩向量。**这是最常用的向量索引，也是 `Index::Auto` 对向量列的默认选择**（`table.rs:1662`）。

### 2.3 HNSW —— 图索引（在向量间织一张"近邻高速网"）

**比喻**：社交网络找人。不是挨个问，而是从一个人出发，沿"朋友的朋友"边逐跳逼近目标。HNSW 把向量织成一张多层图，每个向量连着 `m` 个近邻，查询时沿边贪心走，几跳就到目标附近。

- HNSW（Hierarchical Navigable Small World）查询极快、精度高，代价是**建图慢、内存占用高**。
- `m`（`vector.rs:298`，setter 叫 `num_edges` 但写入字段 `m`）= 每个节点连几个邻居；`ef_construction`（`vector.rs:299`）= 建图时每步考察多少候选。两者都是"越大越准越慢"。
- lancedb 把 HNSW 和 IVF 叠用：先 IVF 分桶，**每个桶内再建一张 HNSW 图**（`vector.rs:281-288` 文档）。

### 2.4 SQ —— 标量量化（更轻的压缩，每维压成 1 字节）

**比喻**：PQ 是"切块换色卡编号"，SQ 更简单粗暴——把每一维的 float32 直接映射成一个 8 位整数（0~255），4 倍压缩（`vector.rs:335-336` 文档）。

- SQ（Scalar Quantization）比 PQ 简单，压缩率固定（float32 → 4x）。`IvfHnswSqIndexBuilder` = IVF + HNSW + SQ 压缩。
- 注意 `IvfHnswSqIndexBuilder` **没有 `num_sub_vectors`/`num_bits` 字段**（`vector.rs:349` 留了 TODO 注释：等 SQ 支持 8 以外的 `num_bits` 再加）——因为 SQ 当前固定 8 位，无可调量化粒度。

### 2.5 五种 Builder 的速查对照

| Builder | 位置 | = 哪些技术叠加 | 压缩 | 适合场景 |
|---|---|---|---|---|
| `IvfFlatIndexBuilder` | `vector.rs:178` | IVF（仅分桶） | 无（存原始向量） | 精度优先、数据量中等 |
| `IvfPqIndexBuilder` | `vector.rs:224` | IVF + PQ | 有损（高压缩） | **默认首选**，省内存 |
| `IvfHnswPqIndexBuilder` | `vector.rs:290` | IVF + HNSW + PQ | 有损 | 查询极快 + 省内存 |
| `IvfHnswSqIndexBuilder` | `vector.rs:338` | IVF + HNSW + SQ | 有损（4x 固定） | 查询快 + 精度比 PQ 略好 |

> **`VectorIndex` 是另一回事**（`vector.rs:15`，别和上面五个 Builder 混）。它不是建索引的菜单，而是**从 lance 元数据重建"这个索引盖了哪几列"**的辅助结构。`new_from_format(manifest, index)`（`vector.rs:22`）遍历 lance `Index` 的 `fields`（一组 field_id），用 `manifest.schema.field_by_id` 反查列名（`vector.rs:27-29`）。找不到时它直接 `panic!`（`vector.rs:30-35`）——见下方 Rust 陷阱。

> **Rust 陷阱：库代码里也会 `panic!`**。一般库代码用 `Result` 返回可恢复错误，但 `VectorIndex::new_from_format`（`vector.rs:30-35`）在 field_id 查不到列名时直接 `panic!`。为什么？因为"索引里记录的列居然不在 schema 里"意味着**元数据已损坏、不变量被破坏**——这不是用户能恢复的错误，而是"绝不该发生"的状态。Rust 用 `Result`/`panic!` 的分野来区分"可恢复错误"与"程序员错误/数据损坏"。

---

## 3. 向量 Builder 的参数表 + 宏复用

这一节回答"每个参数填多少"，并揭示这个模块最精巧的设计：**4 个宏复用 setter**。

### 3.1 四个宏：配料单的"印章"

五个向量 Builder 字段高度重叠（都有 `distance_type`、`num_partitions`、`sample_rate`……）。如果每个 Builder 手写一遍这些 setter，会有大量重复。lancedb 用 4 个 `macro_rules!` 宏解决（`vector.rs:48-164`）：

| 宏 | 位置 | 展开出哪些 setter | 装在哪些 Builder |
|---|---|---|---|
| `impl_distance_type_setter!` | `vector.rs:48` | `distance_type()` | 全部 5 个 |
| `impl_ivf_params_setter!` | `vector.rs:67` | `num_partitions()` / `sample_rate()` / `max_iterations()` | 全部 5 个 |
| `impl_pq_params_setter!` | `vector.rs:118` | `num_sub_vectors()` / `num_bits()` | IvfPq、IvfHnswPq |
| `impl_hnsw_params_setter!` | `vector.rs:143` | `num_edges()`（写入 `m`）/ `ef_construction()` | IvfHnswPq、IvfHnswSq |

每个 Builder 的 `impl` 块只是"盖几个印章"。例如 `IvfHnswPqIndexBuilder`（`vector.rs:321-326`）四个宏全上：

```rust
impl IvfHnswPqIndexBuilder {
    impl_distance_type_setter!();   // ① 盖上 distance_type setter
    impl_ivf_params_setter!();      // ② 盖上 IVF 三件套
    impl_hnsw_params_setter!();     // ③ 盖上 HNSW 两件套
    impl_pq_params_setter!();       // ④ 盖上 PQ 两件套
}
```

> **Rust 陷阱：`macro_rules!` 不是函数也不是泛型**。它是**声明宏**——编译期把宏体的文本"原样展开"进调用处（`vector.rs:199` 那行 `impl_ivf_params_setter!();` 会被替换成三个完整方法定义）。类比 C 的 `#define`，但它是**卫生的（hygienic）**：不会意外捕获/污染外部变量名。非 Rust 读者会困惑"为什么 5 个 struct 都有 `num_partitions` 方法却看不到定义"——因为它们是宏展开出来的，源码里只有一行 `impl_ivf_params_setter!();`。
>
> **给贡献者的实操点**：想给**所有**向量索引加一个新通用参数 → 改对应的宏（一处改、五处生效）；想给**单个**索引加专有参数 → 在那个 Builder 的 `impl` 块里写独立方法。`setter 名 num_edges 实际写字段 m`（`vector.rs:149-151`）是个历史命名遗留——方法叫 `num_edges`，字段叫 `m`，别被绕晕。

### 3.2 完整参数表（含默认值，逐一核对）

| 参数 | 字段类型 | 静态默认（Builder 的 `Default`） | 含义 / 调大调小的影响 |
|---|---|---|---|
| `distance_type` | `DistanceType` | `L2`（`vector.rs:190` 等） | 距离度量；建索引和查询必须一致，否则结果错 |
| `num_partitions` | `Option<u32>` | `None` → 运行时算 | IVF 分桶数；太大则"挑桶"慢，太小则"桶内搜"慢 |
| `sample_rate` | `u32` | `256`（`vector.rs:192`） | kmeans 训练采样率；训练向量数 = `sample_rate * num_partitions` |
| `max_iterations` | `u32` | `50`（`vector.rs:193`） | kmeans 最大迭代数；通常用不满，提前收敛 |
| `num_sub_vectors` | `Option<u32>` | `None` → 运行时算 | PQ 子向量段数；越多压缩越小、越准 |
| `num_bits` | `Option<u32>` | `None` | PQ 每段编号位数（见 §3.4 的"装饰品"警告） |
| `m` | `u32` | `20`（`vector.rs:315`） | HNSW 每节点邻居数；越大越准越慢 |
| `ef_construction` | `u32` | `300`（`vector.rs:316`） | HNSW 建图候选数；不应小于查询期的 `ef` |

> **Rust 知识点：`Option<u32>` 表达"用户没指定"**。`num_partitions: Option<u32>`（`vector.rs:182`）里，`None` = "我不指定，系统你来算"，`Some(256)` = "我就要 256"。这比 Python 的"默认值 None 再 if 判断"或某些语言的"magic 默认值（-1 代表自动）"更**显式、更类型安全**——编译器逼你处理 `None` 这种情况。

### 3.3 默认值的"两段式"：静态 vs 数据驱动

注意上表里 `num_partitions`/`num_sub_vectors` 的静态默认是 `None`——这不是真正的默认值，而是"**留给运行时按数据算**"的信号。真正的默认在 `table.rs` 建索引时才用三个 `suggested_*` 函数算出：

| 推断函数 | 位置 | 公式 | 比喻 |
|---|---|---|---|
| `suggested_num_partitions(rows)` | `vector.rs:256-259` | `max(1, sqrt(rows))` | 桌上人越多，分的菜桌越多 |
| `suggested_num_partitions_for_hnsw(rows, dim)` | `vector.rs:261-264` | `max(1, rows*dim/(256*5_000_000))` | HNSW 按"总数据量"估桶数 |
| `suggested_num_sub_vectors(dim)` | `vector.rs:266-279` | `dim%16==0 → dim/16`；`dim%8==0 → dim/8`；否则 `log::warn` 并返回 `1` | 按维度能否整除挑切法 |

> **比喻**：`suggested_*` 函数是厨师的"看人下菜"。你不指定分量（`None`），厨师就按桌上人数（`rows`）用 `sqrt` 公式估一个量。`suggested_num_sub_vectors` 还会在维度不能被 8 整除时 `log::warn` 提醒"这会拖慢 PQ"——因为 8/16 的整除能用上高效 SIMD 指令（`vector.rs:126-131` 文档）。
>
> **给贡献者**：改默认行为要分清在**哪一层**改。改 Builder 的 `Default`（如把静态默认从 `None` 改成 `Some(x)`）会让"自动推断"失效；改 `suggested_*` 函数则改的是"数据驱动默认"的公式。两者作用层不同。

### 3.4 翻译柜台：Builder 字段 → lance `BuildParams`

现在看"翻译柜台"——`table.rs` 里 `create_*_index` 怎么把 Builder 字段映射成 lance 参数。以 `create_ivf_hnsw_pq_index` 为例，它最清楚地展示了**逐字段映射**（`table.rs:1573-1589`）：

```rust
let mut dataset = self.dataset.get_mut().await?;            // ① 拿写锁（见 09 篇"金库保安"）
let mut ivf_params = IvfBuildParams::new(num_partitions as usize);
ivf_params.sample_rate = index.sample_rate as usize;       // ② Builder.sample_rate → lance
ivf_params.max_iters = index.max_iterations as usize;      // ③ Builder.max_iterations → lance
let hnsw_params = HnswBuildParams::default()
    .num_edges(index.m as usize)                           // ④ Builder.m → lance num_edges
    .ef_construction(index.ef_construction as usize);
let pq_params = PQBuildParams {
    num_sub_vectors: num_sub_vectors as usize,             // ⑤ Builder.num_sub_vectors → lance
    ..Default::default()
};
let lance_idx_params = VectorIndexParams::with_ivf_hnsw_pq_params(  // ⑥ 三组参数组装成一个工单
    index.distance_type.into(), ivf_params, hnsw_params, pq_params,
);
```

`IvfBuildParams` / `HnswBuildParams` / `PQBuildParams` / `VectorIndexParams` 全都来自 lance crate。**lancedb 一行算法都不写，只负责把 Builder 字段搬进 lance 的结构体。** 这就是"翻译柜台"的字面意思。

`create_ivf_flat_index`（`table.rs:1446`）更简单，没有 PQ/HNSW 步骤，直接 `VectorIndexParams::ivf_flat(num_partitions, distance_type.into())`（`table.rs:1468-1471`）。`create_ivf_hnsw_sq_index`（`table.rs:1602`）结构同 HnswPq，但用 `SQBuildParams { sample_rate, .. }`（`table.rs:1638-1641`）替代 PQ。

### 3.5 ⚠️ `num_bits` 是装饰品：字段存在 ≠ 生效

这是本模块最隐蔽的陷阱。`IvfPqIndexBuilder` 暴露了 `num_bits` 字段和 setter（`vector.rs:234` / `136`），看起来你能调。但翻译柜台 `create_ivf_pq_index` 把它**硬编码成 8**，根本不读你填的值（`table.rs:1512-1518`）：

```rust
let lance_idx_params = VectorIndexParams::ivf_pq(
    num_partitions as usize,
    /*num_bits=*/ 8,            // ← 硬编码 8，忽略 index.num_bits 字段！
    num_sub_vectors as usize,
    index.distance_type.into(),
    index.max_iterations as usize,
);
```

> **比喻**：菜单上印着"可选辣度"，但厨房现在只会做中辣（写死 8），这个选项摆着暂未接通。
>
> **给贡献者**：如果你想真正启用 `num_bits`，**只改 Builder 不够**——必须改 `table.rs:1514` 这行把硬编码 8 换成 `index.num_bits.unwrap_or(8)` 之类。这是"字段存在不等于字段生效"的典型，也是一个潜在的 good-first-issue 切入点。

---

## 4. 标量索引四种：各适合什么过滤

标量索引（`index/scalar.rs`）解决的是**非向量列的过滤加速**：`x > 10`、`x = 'foo'`、`tags 包含 'a'`、全文搜索（`scalar.rs:4-11` 文档）。四种各有擅长。

### 4.1 三个空 struct + 一个有参数的

| Builder | 位置 | 字段 | 适合的过滤 | 比喻 |
|---|---|---|---|---|
| `BTreeIndexBuilder` | `scalar.rs:31` | 空 `{}` | 范围/比较/等值（`>`,`<`,`=`，高选择性列） | 排好序的电话簿，二分查找 |
| `BitmapIndexBuilder` | `scalar.rs:43` | 空 `{}` | 低基数列（<1000 唯一值，如性别/状态） | 每个取值一张"谁有它"的勾选表 |
| `LabelListIndexBuilder` | `scalar.rs:52` | 空 `{}` | `List<T>` 多值标签列的 `array_contains_all/any` | 给"一行多个标签"建索引 |
| `FtsIndexBuilder` | `scalar.rs:58` | `with_position` + `tokenizer_configs` | 全文搜索（BM25） | 倒排索引，按词找文档 |

- **BTree**（`scalar.rs:13-29` 文档）：存一份排序副本，每 4096 行一个 header 条目（block size 固定 4096）。是**标量列的默认索引**（`index.rs:32`）。
- **Bitmap**（`scalar.rs:35-41`）：每个不同取值存一张 bitmap（哪些行有这个值）。唯一值越少越划算。
- **LabelList**（`scalar.rs:45-52`）：底层也是 bitmap，专门处理 `List<T>` 列——让"这一行的标签数组里是否同时/部分包含某些值"能走索引。
- 前三个都是 **`derive(Default)` 的空 struct `{}`**，没有任何可调参数。

> **教学点：为什么标量索引全是空 struct？** BTree/Bitmap/LabelList 当前确实**没有可调参数**（`scalar.rs:28-29` 注释明说 btree 暂无参数，未来可能加 block size）。留空 struct 是**前瞻性设计**：占位，将来加字段时不破坏 `Index::BTree(...)` 这个 enum 形状。
>
> **Rust 知识点：空 struct `{}` 仍是合法类型**。`pub struct BTreeIndexBuilder {}`（`scalar.rs:31`）带大括号、占 0 字节，但仍是有效类型。`#[derive(Default)]` 让 `Default::default()` 可用，于是 `Index::BTree(Default::default())` 能实例化它。Rust 里空 struct 很常见——它表达"我是一种类型/一个意图，但暂无状态"。

### 4.2 `FtsIndexBuilder`：唯一有参数的标量 Builder

`scalar.rs:58-81`：

```rust
#[derive(Debug, Clone)]
pub struct FtsIndexBuilder {
    pub with_position: bool,            // ① 是否存 token 位置（短语查询要用）
    pub tokenizer_configs: TokenizerConfig,  // ② 分词器配置，类型来自 lance_index
}

impl Default for FtsIndexBuilder {
    fn default() -> Self {
        Self { with_position: true, tokenizer_configs: TokenizerConfig::default() }  // ③ 默认存位置
    }
}

impl FtsIndexBuilder {
    pub fn with_position(mut self, with_position: bool) -> Self {  // ④ 唯一的链式 setter
        self.with_position = with_position;
        self
    }
}
```

`with_position=true`（`scalar.rs:69`）默认开启，存 token 在文档中的位置，从而支持**短语查询**（"机器 学习" 作为相邻词组）；关掉它能省空间但失去短语能力。

> **再证"组装而非重造"**：`TokenizerConfig`、`FullTextSearchQuery` 等全文相关类型都**不是 lancedb 自己定义的**，而是从 lance_index re-export（`scalar.rs:83-85`）：
> ```rust
> pub use lance_index::scalar::inverted::query::*;
> pub use lance_index::scalar::inverted::TokenizerConfig;
> pub use lance_index::scalar::FullTextSearchQuery;
> ```
> lancedb 连全文搜索的类型都直接借用 lance_index，自己只负责"把 Builder 两字段搬进 lance params"。

### 4.3 标量索引的翻译柜台

标量索引的翻译柜台比向量简单。BTree/Bitmap/LabelList **三者同构**，只换两个标签：`ScalarIndexType`（lance 内部类型）和 `IndexType`（建索引的 kind）。以 BTree 为例（`table.rs:1688-1700`）：

```rust
let lance_idx_params = lance_index::scalar::ScalarIndexParams {
    force_index_type: Some(lance_index::scalar::ScalarIndexType::BTree),  // ① 只换这个枚举值
};
dataset.create_index(&[field.name()], IndexType::BTree, None, &lance_idx_params, opts.replace).await?;
```

Bitmap（`table.rs:1704`）、LabelList（`table.rs:1731`）只是把 `ScalarIndexType::BTree`/`IndexType::BTree` 换成对应值。FTS 走另一条路——`InvertedIndexParams`（`table.rs:1775-1778`），把 Builder 两字段直接搬进去：

```rust
let fts_params = lance_index::scalar::InvertedIndexParams {
    with_position: fts_opts.with_position,        // Builder.with_position
    tokenizer_config: fts_opts.tokenizer_configs, // Builder.tokenizer_configs
};
```

### 4.4 列类型校验：哪些列能建哪种索引

每个 `create_*_index` 开头都先校验列类型，校验函数在 `utils.rs`：

| 校验函数 | 位置 | 接受的列类型 |
|---|---|---|
| `supported_vector_data_type` | `utils.rs:174` | `FixedSizeList` 内层浮点或 UInt8，或 `List` 递归满足 |
| `supported_btree_data_type` | `utils.rs:132` | 整数/浮点/Boolean/Utf8/时间日期/FixedSizeBinary |
| `supported_bitmap_data_type` | `utils.rs:148` | 整数或 Utf8 |
| `supported_label_list_data_type` | `utils.rs:152` | `List`/`FixedSizeList` 且内层满足 bitmap |
| `supported_fts_data_type` | `utils.rs:160` | Utf8/LargeUtf8 或其 List |

校验不过就提前返回 `Error`（向量索引返回 `Error::InvalidInput`，标量索引返回 `Error::Schema`）。

---

## 5. dispatch 与 `Index::Auto`：怎么分发、怎么自动选

### 5.1 NativeTable 的总分发器

所有路径汇集于 `NativeTable::create_index`（`table.rs:1989-2019`）——这是 `IndexBuilder::execute` 委托的最终落点（本地表场景）。它做两件事：先拒绝复合索引，再按变体分发：

```rust
async fn create_index(&self, opts: IndexBuilder) -> Result<()> {
    if opts.columns.len() != 1 {                       // ① 当前不支持多列复合索引
        return Err(Error::Schema {
            message: "Multi-column (composite) indices are not yet supported".to_string(),
        });
    }
    let schema = self.schema().await?;
    let field = schema.field_with_name(&opts.columns[0])?;   // ② 取出那一列

    match opts.index {                                  // ③ 按 9 个变体分发到各 create_*_index
        Index::Auto => self.create_auto_index(field, opts).await,
        Index::BTree(_) => self.create_btree_index(field, opts).await,
        Index::Bitmap(_) => self.create_bitmap_index(field, opts).await,
        Index::LabelList(_) => self.create_label_list_index(field, opts).await,
        Index::FTS(fts_opts) => self.create_fts_index(field, fts_opts, opts.replace).await,
        Index::IvfFlat(ivf_flat) => self.create_ivf_flat_index(ivf_flat, field, opts.replace).await,
        Index::IvfPq(ivf_pq) => self.create_ivf_pq_index(ivf_pq, field, opts.replace).await,
        Index::IvfHnswPq(x) => self.create_ivf_hnsw_pq_index(x, field, opts.replace).await,
        Index::IvfHnswSq(x) => self.create_ivf_hnsw_sq_index(x, field, opts.replace).await,
    }
}
```

> **Rust 知识点：`match` 的穷尽性**。这个 `match` 必须覆盖 `Index` 的全部 9 个变体，否则编译不过。这是 enum + match 的杀手锏：将来给 `Index` 加第 10 个变体，编译器会**强制**你在这里补一个分支，不会漏。注意 `Index::BTree(_)` 用 `_` 忽略内嵌的空 Builder（反正没参数），而 `Index::IvfPq(ivf_pq)` 则把 Builder 绑出来传给下游。

### 5.2 `Index::Auto`：不是新算法，是"按列类型挑默认 Builder"

`create_auto_index`（`table.rs:1660-1675`）：

```rust
async fn create_auto_index(&self, field: &Field, opts: IndexBuilder) -> Result<()> {
    if supported_vector_data_type(field.data_type()) {
        self.create_ivf_pq_index(IvfPqIndexBuilder::default(), field, opts.replace).await  // ① 向量列 → IVF_PQ
    } else if supported_btree_data_type(field.data_type()) {
        self.create_btree_index(field, opts).await                                         // ② 标量列 → BTree
    } else {
        Err(Error::InvalidInput { /* 该列没有支持的索引 */ })                                // ③ 都不支持就报错
    }
}
```

**`Auto` 完全不是新算法**——它只是一个 if-else：列是向量类型就用默认的 `IvfPqIndexBuilder`，是 BTree 支持的标量类型就建 BTree，否则报错。它复用的就是 §3/§4 已有的 `create_*_index`。

> **比喻**：`Auto` 是顾客说"随便上个合适的"，服务员看了眼这桌点的是向量列还是标量列，就替你点了 IVF_PQ 或 BTree——没有新菜，只是替你做了选择。

### 5.3 RemoteTable：另一条路，且少一道菜

云端表走完全不同的分发（`remote/table.rs:741-789`）：不调 lance，而是把 Builder **序列化成 JSON** 发给云端。它的 `match` 只提取 `(index_type 字符串, distance_type)`，不传具体参数（`remote/table.rs:742-743` 注释说"SaaS 还不支持具体参数"）：

```rust
let (index_type, distance_type) = match index.index {
    Index::IvfFlat(index) => ("IVF_FLAT", Some(index.distance_type)),
    Index::IvfPq(index)   => ("IVF_PQ",   Some(index.distance_type)),
    Index::IvfHnswSq(index) => ("IVF_HNSW_SQ", Some(index.distance_type)),
    Index::BTree(_) => ("BTREE", None),
    Index::Bitmap(_) => ("BITMAP", None),
    Index::LabelList(_) => ("LABEL_LIST", None),
    Index::FTS(fts) => { /* 把 tokenizer_configs 序列化进 body + with_position */ ("FTS", None) }
    Index::Auto => { /* 按列类型选 IVF_PQ 或 BTREE */ }
    _ => return Err(Error::NotSupported { message: "Index type not supported".into() }),  // ★ 兜底
};
```

> **⚠️ 关键差异：云端缺 `IvfHnswPq` 分支**。数一数这个 `match`：有 `IvfFlat`/`IvfPq`/`IvfHnswSq`，**唯独没有 `IvfHnswPq`**。它会落入 `_ =>` 兜底返回 `Error::NotSupported`（`remote/table.rs:784-788`）。也就是说，**云端不支持 IVF_HNSW_PQ 索引，本地 NativeTable 则 9 种全支持**。
>
> **比喻**：远程分店的菜单少印了一道菜（IVF_HNSW_PQ），点了就回你"本店暂不供应"。
>
> **教学点：两条平行路径，改 Builder 要管两头**。`NativeTable`（本地后厨，`table.rs`）直接调 lance `Dataset::create_index` 落盘；`RemoteTable`（远程分店，`remote/table.rs`）把 Builder 序列化成 JSON 发云端。你给某个 Builder 加新字段时，**两条路都要更新**，否则云端那条会落到 `NotSupported` 或悄悄丢字段。

---

## 6. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| 带数据的 `enum` | 每个变体可内嵌不同类型字段（类比 TS discriminated union） | `Index` :23 |
| `match` 穷尽性 | 必须覆盖所有变体，加变体编译器强制补分支 | `create_index` :1999 |
| `macro_rules!` 声明宏 | 编译期文本展开（类比卫生版 `#define`），非函数非泛型 | `vector.rs:48-164` |
| `mut self -> Self` 链式 setter | 消费自身、改、还回；调用后原变量被移动 | `IndexBuilder::replace` :89 |
| `Option<T>` 表"未指定" | `None`=交给系统算，`Some(x)`=用户显式给 | `num_partitions` `vector.rs:182` |
| `#[derive(Deserialize)]` | serde 自动生成反序列化代码，无需手写 | `IndexType` :99 |
| `#[serde(alias)]` | 一个变体匹配多种字符串拼写 | `IndexType` :102 |
| `#[skip_serializing_none]` | 让 `None` 字段不出现在序列化 JSON 中 | `IndexStatistics` :193 |
| 空 struct `{}` | 0 字节但合法类型，`derive(Default)` 可实例化 | `BTreeIndexBuilder` `scalar.rs:31` |
| `panic!` in 库代码 | 数据损坏/不变量破坏时用，区别于可恢复的 `Result` | `vector.rs:30-35` |
| 同名不同类型 | 三个 `Index` 靠模块路径区分 | `index.rs:23` vs `:100` vs `vector.rs:11` |
| `..Default::default()` | 结构体更新语法：填几个字段，其余取默认 | `PQBuildParams` `table.rs:1582` |

---

## 7. 动手验证（建议亲手做一遍）

1. **数变体**：打开 `index.rs:23`，确认 `Index` 是 9 个变体、`Auto`（`:24`）是唯一不带 Builder 的；再到 `:100` 确认 `IndexType` 是 8 个（少了 `Auto`）。问自己：为什么 `IndexType` 没有 `Auto`？
2. **找宏展开**：在 `vector.rs` 里搜 `impl_ivf_params_setter!`，确认它定义于 `:67`、被 5 个 Builder 的 `impl` 块各调一次。再在 IDE 里对 `IvfFlatIndexBuilder` 上点 `num_partitions(` 看跳转——它来自宏，源码里看不到独立定义。
3. **验证 `num_bits` 是装饰品**：从 `IvfPqIndexBuilder::num_bits` setter（`vector.rs:136`）出发，追到 `create_ivf_pq_index`（`table.rs:1484`），亲眼看到 `:1514` 那行 `/*num_bits=*/ 8` 硬编码、根本没读 `index.num_bits`。
4. **比对两条路径**：把 `table.rs:1999` 的 NativeTable `match`（9 个分支）和 `remote/table.rs:741` 的 RemoteTable `match` 并排看，找出后者缺 `IvfHnswPq` 分支、会落到 `:784` 的 `NotSupported`。
5. **跟一次 Auto**：从 `Index::Auto`（`:2000`）追进 `create_auto_index`（`:1660`），确认它只是个 if-else，复用 `create_ivf_pq_index` / `create_btree_index`，不含任何新算法。

---

## 8. 小结 & 下一篇

- **index 模块是声明层，不是算法层**：`Index` enum + 各 Builder 只声明"要建什么索引、什么参数"；真正的 kmeans/IVF/PQ/HNSW/SQ/BTree/Bitmap/Inverted 算法全在 `lance` 与 `lance_index`。改这个模块通常是"加一个 Builder 参数并映射到 lance"，而非"写算法"——典型的**组装而非重造**。
- **三层结构**：`index.rs`（`Index`/`IndexBuilder`/`IndexType` + 反序列化镜像）→ `index/vector.rs`（5 个向量 Builder，4 个宏共享 setter，3 个 `suggested_*` 推断函数）→ `index/scalar.rs`（4 个标量/全文 Builder，前三个空 struct）。
- **翻译柜台在 `table.rs`**：`NativeTable::create_index`（`:1989`）按变体 dispatch 到 `create_*_index`，每个负责"校验列类型 → 补数据驱动默认 → 构造 lance `*BuildParams` → 调 `dataset.create_index`"。
- **两条平行路径**：本地走 lance 落盘，云端（`remote/table.rs:741`）序列化成 JSON 且**不支持 `IvfHnswPq`**。
- **三个给贡献者的陷阱**：① `num_bits` 字段存在但被硬编码 8 覆盖（`table.rs:1514`）；② setter `num_edges` 实际写字段 `m`（`vector.rs:149`）；③ 改 Builder 字段必须同步管 NativeTable 和 RemoteTable 两条路。

**下一篇**将从"建好的索引怎么被查询用上"切入——回到 `06 查询链路` 的延伸，看 prefilter/向量召回如何在执行计划里真正读这些索引文件。
