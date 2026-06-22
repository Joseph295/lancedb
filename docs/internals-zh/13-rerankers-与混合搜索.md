# 13 · Rerankers 与混合搜索：两路召回如何融成一张榜

> **本篇解剖的源码**：
> - `rust/lancedb/src/rerankers.rs`（111 行）—— `Reranker` 合同、`NormalizeMethod` 枚举、`merge_results` 默认实现、`check_reranker_result` 校验
> - `rust/lancedb/src/rerankers/rrf.rs`（223 行）—— RRF 算法的完整实现
> - `rust/lancedb/src/query/hybrid.rs`（347 行）—— `rank` / `query_schemas` / `normalize_scores` 三个工具函数
> - `rust/lancedb/src/query.rs:1014-1086` —— `execute_hybrid` 控制流，把上面这些零件串起来
>
> **覆盖区间**：从"用户调 `.full_text_search(...)` 触发混合搜索"到"返回带 `_relevance_score` 列的单一结果批次"的全过程。
>
> **前置阅读**：先读 `01 Rust 垫脚石`（`trait` / `dyn` / `Arc` / `async` 四节）；强烈建议先读 `11 Query 体系全解`（本篇是它的延续——`QueryRequest.reranker` / `norm` 两个字段在 11 里只挂了名，本篇兑现）。
>
> **与哪些篇互补**：`06 查询链路`讲"一次普通向量查询的一生"，本篇专讲"混合查询这条岔路"；`12 Index 体系`讲向量索引与 FTS 倒排索引各自怎么建，本篇讲它们的结果怎么合。本篇是**模块深潜**，重在 reranker 模块自身结构，不重复链路细节。

---

## 0. 这一篇要回答的三个问题

如果你只带走三句话：

1. **混合搜索 = 两场比赛，一张榜**：向量召回跑一场、关键词召回跑一场，最后要把两份成绩单合成一张总榜。`Reranker` 就是那位**合并成绩单的评审岗**。
2. **融合靠的是身份证号，不是分数**：向量给的是"距离"、FTS 给的是"BM25 分"，单位天差地别。唯一能确认"两场里这是同一个文档"的钥匙，是 lance 的 `_rowid`。所以两路结果都被强制带上 `_rowid`。
3. **这个模块几乎没自研算法**：排序用 Arrow 的 `sort_to_indices`+`take`，去重用 `filter_record_batch`，归一化用 Arrow 的 `div`/`sub` 算子。reranker 模块真正自己写的核心代码，只有 RRF 那段 `BTreeMap` 累加循环。又一次"组装而非重造"。

把这三句话画成图，就是混合搜索的骨架：

```
  你的代码:  table.query()
                .nearest_to(vec)            ← 给了向量
                .full_text_search("猫粮")    ← 又给了关键词  ⇒ 触发混合搜索
                .execute()
                    │
                    │ VectorQuery::execute_with_options 检测到 full_text_search.is_some()
                    ▼                                            (query.rs:1113)
         ┌──────────────────────────────────────┐
         │          execute_hybrid               │  query.rs:1014  ← 餐厅经理拆单
         │                                       │
         │   fan-out：把一张查询拆成两张子查询      │
         └───────┬───────────────────┬──────────┘
                 │                   │
          fts_query           vector_query        ← 都强制 with_row_id()
        (清空向量部分)        (清空 full_text_search)   (query.rs:1021,1023)
                 │  try_join! 并发跑   │
                 ▼                   ▼
         _score 列 + _rowid     _distance 列 + _rowid
                 │                   │
                 └────── fan-in ─────┘
                          │
                  归一化 normalize_scores → [0,1]   (query.rs:1049-1050)
                          │
                          ▼
         ┌──────────────────────────────────────┐
         │   Reranker::rerank_hybrid（评审岗）    │  rerankers.rs:60
         │   缺省 = RRFReranker(k=60)             │  query.rs:1057
         │   产出必须含 _relevance_score 列        │  rerankers.rs:99
         └──────────────────┬───────────────────┘
                            ▼
              按 _relevance_score 降序的单一结果批次
                （超 limit 截断、用户没要 row_id 则丢掉）
```

> **比喻总钥匙**（延续全系列家族）：餐厅经理（LanceDB）收到一张"我要又快又好"的订单，把它拆成两张工单——一张给**计时灶台**（向量搜索，按距离打分）、一张给**品鉴灶台**（全文搜索，按 BM25 打分）。两个灶台**同时开火**（`try_join!` 并发），出锅后端给**评审岗**（`Reranker`）合并摆盘，最后这盘菜必须盖上"综合评分"的章（`_relevance_score` 列）才算合格出菜。

---

## 1. 为什么需要混合搜索：两种召回各有盲区

先讲清"为什么"，再讲"怎么做"。

**向量召回**（语义搜索）擅长理解意思：搜"幼猫食品"，它能召回标题写着"小猫粮"的商品，因为两者在向量空间里挨得近。但它对**精确关键词、罕见专有名词、型号编号**很弱——搜 "SKU-X7齿轮油"，语义模型可能根本没见过这个词，向量化成一团糊。

**关键词召回**（FTS / BM25 全文检索）恰好相反：它对精确词、专名、编号极准，但**完全不懂同义词与语义**——搜"幼猫食品"它就死磕"幼猫""食品"这两个词，标题写"小猫粮"的商品因为一个字都不匹配而被漏掉。

> **比喻**：向量召回像一位**懂行的导购**，你说个大概意思他就能给你推相近的；关键词召回像一位**严谨的仓管**，你报准确货号他分毫不差，但你说"差不多那种"他就两手一摊。混合搜索就是让导购和仓管**各列一张推荐单**，再请评审岗综合两张单子出最终榜——既不漏语义近似的，也不漏关键词精确命中的。

这就是混合搜索的动机：**两路召回的盲区互补**。剩下的全部工程问题，归结为一句话——**怎么把两张计分标准不同的成绩单，公平地合成一张？** 这正是 `Reranker` 要解决的。

---

## 2. `Reranker`：评审岗的"用工合同"

`rerankers.rs:53-97`：

```rust
#[async_trait]                                            // ①
pub trait Reranker: std::fmt::Debug + Sync + Send {      // ②
    // TODO support vector reranking and FTS reranking. Currently only hybrid reranking is supported.

    async fn rerank_hybrid(                               // ③ 唯一抽象方法（必须实现）
        &self,
        query: &str,                                     //   原始查询串
        vector_results: RecordBatch,                     //   向量这路的结果（餐盘）
        fts_results: RecordBatch,                        //   FTS 这路的结果（餐盘）
    ) -> Result<RecordBatch>;                            //   合并后的总榜

    fn merge_results(                                    // ④ 带默认实现（可白嫖）
        &self,
        vector_results: RecordBatch,
        fts_results: RecordBatch,
    ) -> Result<RecordBatch> { /* 去重逻辑，见 §2.3 */ }
}
```

这是整个混合搜索的**唯一扩展点**。理解它的四个零件，就理解了贡献一个自定义 reranker 要做什么。

### 2.1 拆 `trait` 这一行（写给不懂 Rust 的你）

| 零件 | 含义 | 类比你熟悉的语言 |
|---|---|---|
| `#[async_trait]`（:53） | 一个宏。Rust 原生 trait 当年不能直接把 `async fn` 当方法（返回类型擦除/对象安全问题），这个宏把 `async fn` 改写成"返回 `Box<dyn Future>` 的普通方法" | Python 的 `abstractmethod` + `async def`，但底层是宏展开 |
| `trait Reranker`（:54） | 定义一个**接口**：想当 reranker 必须会做哪些事 | Java 的 `interface`、Python 抽象基类 |
| `: Debug + Sync + Send`（:54） | **supertrait 约束**：实现 `Reranker` 前必须先满足这三个 | "你还得先实现这些" |
| `Debug` | 能被 `{:?}` 打印（调试用） | `repr()` |
| `Sync + Send` | **能安全跨线程共享/传递**——并发的"通行证" | "线程安全标记" |

> **为什么要 `Sync + Send`？** reranker 要被塞进 `Arc<dyn Reranker>`（见 §5）在多个并发查询间共享。`Sync + Send` 是编译器层面的通行证：没带证的类型，编译器根本不让你跨线程共享它。这不是建议，是强制。

### 2.2 `rerank_hybrid`：合同里唯一必做的活

`rerankers.rs:60-65`。这是**唯一的抽象方法**——没有默认实现，谁实现 `Reranker` 谁就必须写它。它的契约是：

- **拿到**：原始查询串 `query`、向量这路的 `RecordBatch`、FTS 这路的 `RecordBatch`。
- **返回**：一个合并后的 `RecordBatch`，且**必须含 `_relevance_score` 列**（否则下游 `check_reranker_result` 报错，见 §5）。

注意 `rerankers.rs:55` 那条 `// TODO`：目前**只支持混合（hybrid）重排**，还不支持对单独的向量结果或单独的 FTS 结果重排。这是给贡献者的明确信号——这里有坑可填。

> **比喻**：`rerank_hybrid` 是用工合同里**唯一一项必做工作**——把计时灶台和品鉴灶台的两份成绩单重排成一张总榜；交付物还必须盖上"综合评分"的章（`_relevance_score`）。怎么排是你的自由（合同没规定算法），但章必须盖。

### 2.3 `merge_results`：合同附带的"标准去重流程"（默认实现）

`rerankers.rs:67-96`。这是 trait 里**唯一带方法体的方法**——它把两路结果上下拼接、再按 `_rowid` 去重。因为带默认实现，自定义 reranker **可以直接白嫖**，不用自己写。

```rust
fn merge_results(&self, vector_results, fts_results) -> Result<RecordBatch> {
    // ① 上下拼接两路（用 fts 的 schema 当模板）
    let combined = concat_batches(&fts_results.schema(),
                                  [vector_results, fts_results].iter())?;   // :72

    let mut mask = BooleanArray::builder(combined.num_rows());
    let mut unique_ids = BTreeSet::new();                                   // ② 签到表
    let row_ids = combined.column_by_name(ROW_ID)                          // :76 找 _rowid 列
        .ok_or(Error::InvalidInput { /* 报错：没有 _rowid 没法去重 */ })?;
    let row_ids: UInt64Array = downcast_array(row_ids);                    // :88 向下转型

    row_ids.values().iter().for_each(|id| {
        mask.append_value(unique_ids.insert(id));                          // :90 ★关键
    });

    let combined = filter_record_batch(&combined, &mask.finish())?;        // :93 按 mask 过滤
    Ok(combined)
}
```

第 `:90` 行是整段的灵魂。`BTreeSet::insert` 的返回值是 `bool`：**插入新元素返回 `true`，已存在返回 `false`**。代码直接把这个布尔当成"保留/丢弃"的 mask：

- 某个 `_rowid` **首次出现** → `insert` 返回 `true` → 这一行保留；
- 同一个 `_rowid` **再次出现**（这文档两路都命中了）→ `insert` 返回 `false` → 这一行丢弃。

> **比喻**：`BTreeSet` 去重就是门口的**签到表**。每个 `_rowid` 来报到，签到表上没有就记 `true`（放行、首次保留），已经签过就记 `false`（重复的同一人，拦下）。

> **Rust 陷阱：`BTreeSet::insert` 返回 `bool`**。Python 的 `set.add()` 返回 `None`，所以 Python 程序员会本能地写 `if id not in seen: seen.add(id)`（两次查找）。Rust 这里一次 `insert` 既插入又告诉你"是不是新的"，省一次查找。这是 Rust 标准库 API 的小巧思，初见容易忽略。

> **Rust 陷阱：`downcast_array`（:88）**。Arrow 的列对外是 `ArrayRef`（一个 trait object，类似"通用数组"）。要按具体类型（这里是 `UInt64Array`）取值，必须**向下转型**。`downcast_array` 类型不符会 **panic**——是运行期断言，不是 `Result`。所以上游必须保证 `_rowid` 真是 `UInt64`。

---

## 3. RRF（Reciprocal Rank Fusion）：倒数排名融合详解

缺省的 reranker 是 `RRFReranker`（`query.rs:1057`）。RRF 是混合搜索里最经典、最稳的融合算法，必须吃透。

### 3.1 先用大白话讲清公式

RRF 全称 **Reciprocal Rank Fusion**，直译"**倒数排名融合**"。核心就一句话：

> **不看分数，只看名次。一个文档在某一路里排第 `i` 名，就拿 `1/(i+k)` 分；它在哪几路出现就把那几路的分加起来；最后按总分降序排榜。**

`i` 是从 **0** 开始的名次（第 1 名 `i=0`，第 2 名 `i=1`……），`k` 是一个常数。公式写出来就是：

```
score(文档 d) = Σ  1 / (rank_in_list(d) + k)
              各路
```

举个例子（`k=1`，对应 `rrf.rs:182-187` 的测试注释）：

| 文档 | 向量这路名次(i) | FTS 这路名次(i) | 总分 |
|---|---|---|---|
| foo | 第1名 (i=0) | 没出现 | 1/(0+1) = 1.0 |
| bar | 第2名 (i=1) | 第1名 (i=0) | 1/(1+1) + 1/(0+1) = **1.5** |
| bean | 第4名 (i=3) | 第2名 (i=1) | 1/4 + 1/2 = 0.75 |
| dog | 第5名 (i=4) | 第3名 (i=2) | 1/5 + 1/3 ≈ 0.533 |
| baz | 第3名 (i=2) | 没出现 | 1/3 ≈ 0.333 |

按总分降序：**bar > foo > bean > dog > baz**。bar 因为"两路都靠前"逆袭夺冠，哪怕它在单独某一路都不是第一。这正是融合的价值——**两路都认可的文档，比某一路独占鳌头的更可信**。

> **比喻**：RRF 就是赛事的**综合积分榜**。向量赛道和全文赛道各跑一场，选手按名次拿分（第1名 `1/(0+k)`、第2名 `1/(1+k)`……），两场都参赛就积分相加，最后按总积分排座次。`k` 是"名次贴水系数"——调大就让冠亚军的分差变小，名次不再那么金贵。

### 3.2 `RRFReranker` 结构体与默认 `k=60`

`rrf.rs:22-43`：

```rust
#[derive(Debug)]
pub struct RRFReranker {
    k: f32,                              // 唯一字段：那个常数 k
}

impl RRFReranker {
    pub fn new(k: f32) -> Self { Self { k } }     // :34 自定义 k
}

impl Default for RRFReranker {
    fn default() -> Self { Self { k: 60.0 } }     // :39-43 缺省 k=60
}
```

`k=60` 不是拍脑袋——`rrf.rs:30-33` 的注释引用了 Cormack 等人 SIGIR'09 的论文，实验表明 **`k=60` 近乎最优，但这个选择并不关键**（fusion 对 `k` 不敏感）。

> **Rust 知识点：`impl Default`**。Rust 没有"默认值"的语言级概念，要表达"不传参数时用 60"，惯用法是为类型实现 `Default` trait。下游 `RRFReranker::default()` 就拿到 `k=60` 的实例。

### 3.3 逐步剖 `rerank_hybrid`：RRF 的打分循环

`rrf.rs:46-150`。`RRFReranker` 只覆写了 `rerank_hybrid` 一个方法，`merge_results` 沿用 §2.3 的 trait 默认实现。分四步看。

**第一步：取出两路的 `_rowid` 列（`rrf.rs:53-83`）**

```rust
let vector_ids = vector_results.column_by_name(ROW_ID).ok_or(/* 报错 */)?;  // :53
let fts_ids    = fts_results.column_by_name(ROW_ID).ok_or(/* 报错 */)?;     // :67
let vector_ids: UInt64Array = downcast_array(&vector_ids);                  // :82
let fts_ids:    UInt64Array = downcast_array(&fts_ids);                     // :83
```

注意方法签名第一个参数是 `_query`（`rrf.rs:49`）——**RRF 根本不看查询串**，下划线前缀是 Rust 表达"我知道这参数没用、别警告我"的惯用法。

**第二步：用 `BTreeMap` 累加每个文档的 RRF 分（`rrf.rs:85-102`，核心中的核心）**

```rust
let mut rrf_score_map = BTreeMap::new();                       // _rowid -> 累计分
let mut update_score_map = |(i, result_id)| {                 // :86 闭包
    let score = 1.0 / (i as f32 + self.k);                    // :87 ★RRF 公式
    rrf_score_map
        .entry(result_id)
        .and_modify(|e| *e += score)                          // :90 有则累加
        .or_insert(score);                                    // :91 无则初始化
};
vector_ids.values().iter().enumerate()                       // :93-97 遍历向量这路
    .for_each(&mut update_score_map);                        //   enumerate() 给出名次 i
fts_ids.values().iter().enumerate()                          // :98-102 再遍历 FTS 这路
    .for_each(&mut update_score_map);
```

`.enumerate()` 把迭代器变成"(下标, 元素)"对，下标就是名次 `i`（从 0 起）——这就是公式里 `i` 的来源。两路依次过一遍 `update_score_map`，同一个 `_rowid` 在第二路出现时走 `and_modify` 把分加上去，这就实现了"两路分数相加"。

> **Rust 陷阱：闭包捕获可变状态**。`let mut update_score_map = |...| {...}` 这个闭包**借用了 `&mut rrf_score_map`**（它要往里写），所以闭包变量本身要声明 `mut`，调用处也得传 `&mut update_score_map`（`:97`、`:102`）。Python/JS 的闭包改外部变量随便改，Rust 的借用检查器在这里逼你显式标注"我要可变借用"。这是初学者最容易卡的语法。

> **Rust 知识点：`entry().and_modify().or_insert()`**。这是 `BTreeMap` 的 entry API，**一次查找**就完成"有则改、无则插"。等价 Python 的 `d[k] = d.get(k, 0) + score`，但 Python 那样写要查两次。`*e += score` 里的 `*e` 是解引用赋值（`e` 是指向 map 里那个值的可变引用）。

**第三步：按 `combined` 的行顺序生成 relevance 列（`rrf.rs:104-114`）**

```rust
let combined_results = self.merge_results(vector_results, fts_results)?;  // :104 去重（白嫖默认实现）
let combined_row_ids: UInt64Array =
    downcast_array(combined_results.column_by_name(ROW_ID).unwrap());     // :106-107
let relevance_scores = Float32Array::from_iter_values(
    combined_row_ids.values().iter()
        .map(|row_id| rrf_score_map.get(row_id).unwrap())                // :112 查表
        .copied(),
);
```

去重后的 `combined_results` 每一行有个 `_rowid`，拿它去 `rrf_score_map` 查出刚算好的总分，按**当前行顺序**排成一个 `Float32Array`。此刻这列还没排序，只是和数据行一一对齐。

**第四步：排序 + 把 relevance 列拼回去（`rrf.rs:116-148`）**

```rust
let sort_indices = sort_to_indices(&relevance_scores,                    // :117 算出降序的行序
    Some(SortOptions { descending: true, ..Default::default() }), None).unwrap();

let mut columns = combined_results.columns().to_vec();
columns.push(Arc::new(relevance_scores));                                // :128-129 追加分数列
let columns = columns.iter()
    .map(|c| take(c, &sort_indices, None).unwrap()).collect();           // :132-135 每列按序重排

let mut fields = combined_results.schema().fields().to_vec();
fields.push(Arc::new(Field::new(RELEVANCE_SCORE, DataType::Float32, false))); // :138-143 schema 加一列
let schema = Schema::new(fields);
let combined_results = RecordBatch::try_new(Arc::new(schema), columns)?; // :146 组装新批次
```

这一步把"组装而非重造"演到极致：**排序不是自己写循环**，而是用 Arrow 的 `sort_to_indices`（算出"按分数降序，行该怎么重排"的下标数组）+ `take`（对每一列按这个下标数组取值，等于整体重排）。

> **Rust 知识点：Arrow 的 schema 是不可变的**。要"加一列"，不能原地 `push`，必须**构造新 schema + 新 `RecordBatch`**（`:138-146`）。这就是为什么这里又是 `fields.push` 又是 `Schema::new` 又是 `RecordBatch::try_new`——Arrow 没有原地 mutation，"改"本质都是"造新的"。

最终产出：一个按 `_relevance_score` 降序、且确实含 `_relevance_score` 列的 `RecordBatch`。合同履约。

---

## 4. `query/hybrid.rs`：归一化与 rank 的三个工具函数

`hybrid.rs` 不定义类型，只提供三个自由函数（外加两个空 schema 辅助函数），是 `execute_hybrid` 在调 reranker **之前**对两路结果做的预处理。

| 函数 | 行号 | 干什么 | 给谁用 |
|---|---|---|---|
| `rank` | `:19-58` | 把"分数列"换成"名次列" | `norm==Rank` 时 |
| `query_schemas` | `:65-86` | 应对空批次，凑出两路的 schema | 拼接前 |
| `normalize_scores` | `:123-174` | min-max 归一到 `[0,1]` | 每次都用 |
| `empty_fts_schema` / `empty_vec_schema` | `:88` / `:95` | 两路都空时的兜底 schema | `query_schemas` 内部 |
| `with_field_name_replaced` | `:102-118` | 把 `_distance` 列名换成 `_score` | `query_schemas` 内部 |

### 4.1 `normalize_scores`：百分制换算

`hybrid.rs:123-174`。把一列原始分用 min-max 公式拉到 `[0,1]`：`(x - min) / range`。

```rust
let max = max(&scores).unwrap_or(0.0);                            // :146 Arrow 的 max 算子
let min = min(&scores).unwrap_or(0.0);                            // :147
let rng = if max - min < 10e-5 { max } else { max - min };       // :150 ★防除零
if rng != 0.0 {                                                  // :153
    let tmp = div(                                              // :154 Arrow 的 div/sub 算子
        &sub(&scores, &Float32Array::new_scalar(min))?,        //   (x - min)
        &Float32Array::new_scalar(rng),                        //   / rng
    )?;
    scores = downcast_array(&tmp);
}
if invert.unwrap_or(false) {                                    // :161 可选反转
    let tmp = sub(&Float32Array::new_scalar(1.0), &scores)?;    //   1 - score
    scores = downcast_array(&tmp);
}
```

几个要点：

- **防除零（`:150`）**：当 `max-min` 小于 `10e-5`（约等于"所有分数都一样"），`rng` 直接取 `max`；若连 `max` 都是 0，则 `rng==0`，`:153` 的 `if` 不进，**分数原样保留**。注释 `:149` 说这等价 Python 的 `np.isclose`——刻意和 Python 实现保持一致的行为。
- **`invert`（`:161`）**：可选地做 `1 - score`。这是为"距离越小越好、但归一后想让大值代表好"这种场景准备的反转开关。
- **空批次（`:141-143`）**：0 行直接原样返回，不 panic。
- 同样**全靠 Arrow 算子**（`div`/`sub`/`max`/`min`）——本模块不手写循环算归一。

> **比喻**：`normalize_scores` 是**百分制换算**。各赛道原始分五花八门（向量距离可能 0.3，BM25 分可能 12.7），先统一拉到 0–1 才好横向比。

### 4.2 `rank`：把分数换成名次

`hybrid.rs:19-58`。直接调 Arrow 的 `arrow::compute::kernels::rank::rank`（`:39`），把分数列原地换成名次列。

```rust
descending: !ascending.unwrap_or(true),     // :42 默认 ascending=true → descending=false
```

默认 `ascending=true` → `descending=false` → **分数小的名次靠前（rank=1）**。这是为向量距离量身定的：距离越小越相似，自然该排第一。算出的名次转成 `Float32` 写回原列（`:38-53`）。空批次同样原样返回（`:33-35`）。

### 4.3 `query_schemas`：空批次的四象限兜底

`hybrid.rs:65-86`。两路结果在拼接前都要有 schema，但**任一路可能完全没命中（空批次列表）**，此时 `.first()` 拿不到 schema。`query_schemas` 用一个 `match` 处理四种组合：

| FTS 这路 | 向量这路 | 处理 |
|---|---|---|
| 有 | 有 | 各用各的 schema |
| 空 | 有 | 拿向量的 schema，把 `_distance` 列名换成 `_score` 当 FTS 的 schema（`:75`） |
| 有 | 空 | 反过来，拿 FTS 的 schema 换名当向量的（`:79`） |
| 空 | 空 | 用 `empty_fts_schema` / `empty_vec_schema` 兜底（`:82`） |

> **给贡献者的提醒**：`query_schemas`、`rank`、`normalize_scores` 三个函数**都有专门的空批次分支**（`:82`、`:33`、`:141`），并且有对应测试 `test_hybrid_search_empty_table`（在 `query.rs` 的测试区）。你改这块逻辑时，**务必保证空表/空结果路径不 panic**——这是审查重点。

---

## 5. 与 `query.rs` 的衔接：`reranker` / `norm` 两个字段如何被用上

`11 Query 体系全解`里，`QueryRequest` 有两个字段一直没讲透，本节兑现。

### 5.1 两个字段与两个 builder 方法

`query.rs:642-647`：

```rust
pub reranker: Option<Arc<dyn Reranker>>,   // :644 重排器（缺省 None）
pub norm: Option<NormalizeMethod>,         // :647 归一化方式（缺省 None）
```

`Default` 把两者都设为 `None`（`query.rs:661-662`）。用户通过 `QueryBase` 的两个 builder 方法设置它们：

| builder 方法 | 声明 | 实现 | 作用 |
|---|---|---|---|
| `rerank(self, Arc<dyn Reranker>)` | `query.rs:445` | `query.rs:501-504` | 指定自定义 reranker |
| `norm(self, NormalizeMethod)` | `query.rs:450` | `query.rs:506-509` | 指定 `Score` 还是 `Rank` |

`NormalizeMethod`（`rerankers.rs:21-25`）是个两变体枚举 `Score` / `Rank`，还实现了 `FromStr`（`:27-39`，大小写不敏感，非法值报 `Error::InvalidInput`）和 `Display`（`:41-48`，输出 `"score"`/`"rank"`），方便从字符串配置和打印。

### 5.2 入口判定：什么时候走混合搜索

`query.rs:1109-1121`：`VectorQuery::execute_with_options` 一进来就检查——

```rust
if self.request.base.full_text_search.is_some() {     // :1113 ★给了 FTS 就走混合
    let hybrid_result = async move { self.execute_hybrid(options).await }
        .boxed().await?;                              // :1114-1116
    return Ok(hybrid_result);
}
self.inner_execute_with_options(options).await        // :1120 否则普通向量查询
```

**判定就一个条件**：一个 `VectorQuery`（已经有向量了）如果**还设了 `full_text_search`**，就是混合搜索。

> **Rust 知识点：`.boxed()`（`:1115`）**。`execute_hybrid` 是个递归较深的 async fn，直接 await 会让 future 类型在编译期无限膨胀。`.boxed()` 把它装进 `Box<dyn Future>`（堆上一个固定大小的指针），切断类型递归。这是 async Rust 处理"自引用/深层 future"的常用招。

### 5.3 `execute_hybrid` 完整控制流（`query.rs:1014-1086`）

把前四节的零件串起来，带行号：

```
execute_hybrid(query.rs:1014)
│
├─① fan-out 拆两路子查询
│   ├ fts_query    = Query::new(parent) + base.clone() + with_row_id()  (:1019-1021)
│   └ vector_query = self.clone().with_row_id(); 再 full_text_search=None (:1023-1025)
│         ↑ 注释明说 row_id "can be needed for reranking"  (:1018)
│
├─② try_join! 并发执行两路                                  (:1026-1029)
│   └ 再 try_join! 并发 try_collect 成 Vec<RecordBatch>      (:1031-1034)
│
├─③ query_schemas 应对空批次拿 schema                       (:1038)
│   └ 各自 concat_batches 拼接                              (:1041-1042)
│
├─④ 若 norm==Some(Rank) 先把分数转排名                       (:1044-1047)
│     vec → rank(DIST_COL),  fts → rank(SCORE_COL)
│
├─⑤ 无条件 normalize_scores 归一到 [0,1]                     (:1049-1050)
│     vec → DIST_COL,  fts → SCORE_COL
│
├─⑥ 取 reranker，缺省 RRFReranker::default()(k=60)          (:1052-1057)
│
├─⑦ 调 rerank_hybrid（query 串取自 fts_query.query.query()） (:1068-1070)
│
├─⑧ check_reranker_result 校验有 _relevance_score 列         (:1072)
│
├─⑨ 行数 > limit 则 slice(0, limit) 截断                     (:1074-1077)
│
└─⑩ 用户没要 row_id 则 drop_column(ROW_ID)                   (:1079-1081)
      └ 包成单批次 stream 返回                               (:1083-1085)
```

几个最易被问到的点：

**为什么两路都强制 `with_row_id()`（`:1021`、`:1023`）？** 向量这路给 `_distance`、FTS 这路给 `_score`，**列名不同、命中的行集合也不同**，唯一能跨两路对齐"这是同一个文档"的钥匙就是 `_rowid`。没有它，reranker 的 `merge_results` 去重和 RRF 的 `BTreeMap` 累加全都无从下手。注释 `:1018` 把这点写得很清楚。

> **比喻**：`_rowid` 是选手的**身份证号**。计时灶台按时间记成绩、品鉴灶台按口味打分，计分单位不同，唯有身份证号能确认"两场里是同一个人"。所以融合前必须给每条结果贴上 `_rowid`；而用户若没主动要这列，末尾再悄悄撕掉（`:1079-1081`）。

**`norm` 字段到底怎么影响流程？**
- `norm == Some(Rank)`：先 `rank` 把原始分换成名次（`:1044-1047`），**再** `normalize_scores`（`:1049-1050`）。
- `norm == Some(Score)` 或 `None`：跳过 rank，直接 `normalize_scores` 原始分。

即：**`Rank` 模式 = 先转名次再归一，`Score` 模式 = 直接归一原分**。

> **⚠️ 初学者最大困惑点：RRF 根本不看归一化分数！** `RRFReranker::rerank_hybrid` 只用 `_rowid` 的**出现顺序**（`enumerate` 名次）算分，完全无视 `:1049-1050` 辛苦算好的 0–1 分数。所以 **`norm` 字段对默认的 RRFReranker 实际无影响**——`norm` 和 `normalize_scores` 是给那些**真的看分数**的 reranker（比如按 score 加权融合的）准备的。你第一次读会觉得"既然 RRF 不看分，为啥还归一"，答案是：归一是为通用流程铺路，不是为 RRF。

> **Rust 知识点：`matches!` 与 `unwrap_or`**。`matches!(self.request.base.norm, Some(NormalizeMethod::Rank))`（`:1044`）是模式匹配宏，等价 Python 的 `norm == NormalizeMethod.Rank`，但能匹配枚举结构。`reranker.clone().unwrap_or(Arc::new(RRFReranker::default()))`（`:1057`）= 有值用值、无值用缺省（`Option` 的链式处理）。

> **Rust 陷阱：`try_join!`（`:1026`、`:1031`）**。并发跑多个 future，**任一返回 `Err` 就整体短路返回 `Err`，全成功才返回元组**。类比 JS 的 `Promise.all`，但带 Rust 的 `Result` 错误传播。这就是"两个灶台同时开火"的实现。

### 5.4 `check_reranker_result`：出菜前的盖章检查

`rerankers.rs:99-110`。reranker 返回后，`execute_hybrid:1072` 立刻调它校验：

```rust
pub fn check_reranker_result(result: &RecordBatch) -> Result<()> {
    if result.schema().column_with_name(RELEVANCE_SCORE).is_none() {
        return Err(Error::Schema { /* "must return ... _relevance_score" */ });
    }
    Ok(())
}
```

`RELEVANCE_SCORE` 常量 = `"_relevance_score"`（`rerankers.rs:19`）。这就是 §2.2 说的"硬约束"的执行点：**任何 reranker 不产出 `_relevance_score` 列，这里就报 `Error::Schema`**，无论它内部排得多漂亮。

---

## 6. 如何自定义一个 Reranker（给贡献者）

把以上拼起来，贡献一个自定义 reranker 的最小清单：

1. **定义一个结构体**（带你需要的配置字段，如权重），`#[derive(Debug)]`。
2. **`#[async_trait] impl Reranker for 你的类型`**，只需写 `async fn rerank_hybrid`——`merge_results` 去重逻辑可白嫖默认实现（`rerankers.rs:67`）。
3. **在 `rerank_hybrid` 里**：用 `_rowid` 对齐两路、按你的算法算分、产出一个**含 `_relevance_score` 列**的 `RecordBatch`。
4. **用户侧**：`query.rerank(Arc::new(你的Reranker))`（`query.rs:501`）即可挂上。

一个"加权融合"的骨架示意（看分数而非名次，所以 `norm` 对它有意义）：

```rust
#[derive(Debug)]
struct WeightedReranker { vector_weight: f32, fts_weight: f32 }

#[async_trait]
impl Reranker for WeightedReranker {
    async fn rerank_hybrid(&self, _query: &str,
        vector_results: RecordBatch, fts_results: RecordBatch) -> Result<RecordBatch> {
        // ① 读两路已归一化的 _distance / _score 列
        // ② 按 _rowid 对齐，combined_score = w_v * vec_norm + w_f * fts_norm
        // ③ self.merge_results(...) 去重（白嫖默认实现）
        // ④ 追加名为 RELEVANCE_SCORE 的 Float32 列、按它降序排
        // ⑤ RecordBatch::try_new(新schema, 重排后的列)
        //    —— 照抄 rrf.rs:116-148 的"sort_to_indices + take + 加列"套路即可
    }
}
```

> **给贡献者的三条铁律**：
> 1. **必须产出 `_relevance_score` 列**，否则 `check_reranker_result`（`rerankers.rs:99`）拦你。
> 2. **必须靠 `_rowid` 对齐**两路——`execute_hybrid` 已保证两路都带 `_rowid`，你直接用。
> 3. **空批次别 panic**——参考 `hybrid.rs` 的空批次分支与 `test_hybrid_search_empty_table` 测试，你的 reranker 也要扛得住某一路 0 行。
>
> 还有那条 `// TODO`（`rerankers.rs:55`）：目前只支持 hybrid 重排，扩展到"单独向量/FTS 重排"是一个现成的贡献切入点。

> **Rust 知识点：`Arc<dyn Reranker>`（`query.rs:644`）**。`dyn` 是动态分发（类似 Java 接口引用/Python 鸭子类型），`Arc` 是原子引用计数智能指针（多线程安全共享所有权）。trait object 必须包在指针后面——所以你的 reranker 总是以 `Arc::new(...)` 的形态挂上去。

---

## 7. 本篇出现的 Rust 语法点 · 速查

| 语法 | 一句话 | 出现处 |
|---|---|---|
| `#[async_trait]` | 让 trait 能写 `async fn` 的宏 | `rerankers.rs:53` |
| `trait T: A + B + C` | 接口 + supertrait 约束（实现前必须先满足） | `Reranker` :54 |
| `Sync + Send` | 跨线程安全的"通行证"，编译器强制 | :54 |
| `Arc<dyn Reranker>` | 引用计数 + 运行时多态；trait object 必须包指针后 | `query.rs:644` |
| `Option<T>` + `unwrap_or(...)` | 有值用值、无值用缺省 | `query.rs:1057` |
| `matches!(x, Pat)` | 模式匹配宏，匹配枚举结构 | `query.rs:1044` |
| 闭包 + `mut` 捕获 | 闭包改外部变量要 `mut`，调用处传 `&mut` | `rrf.rs:86,97,102` |
| `entry().and_modify().or_insert()` | `BTreeMap` 一次查找完成"有则改、无则插" | `rrf.rs:88-91` |
| `BTreeSet::insert -> bool` | 首见返 `true`、重复返 `false`，巧当去重 mask | `rerankers.rs:90` |
| `downcast_array` | Arrow `ArrayRef` 向下转具体类型，失败 panic | `rerankers.rs:88` 等 |
| `try_join!` | 并发跑多 future，任一 Err 即短路 | `query.rs:1026,1031` |
| `.boxed()` | 把 async fn 装进 `Box<dyn Future>`，切断类型递归 | `query.rs:1115` |
| `_query`（下划线前缀） | "我知道这参数没用，别警告" | `rrf.rs:49` |
| Arrow schema 不可变 | "加一列" = 构造新 schema + 新 `RecordBatch` | `rrf.rs:138-146` |

---

## 8. 动手验证（建议亲手做一遍）

1. **跑通 RRF 测试**：`cargo test -p lancedb test_rrf_reranker`（测试函数在 `rrf.rs:158-222`），对照 `rrf.rs:182-187` 的手算注释块，手算 bar=1.5、foo=1.0……验证最终顺序 `[bar,foo,bean,dog,baz]`（ids `[4,1,5,3,2]`）。把 `RRFReranker::new(1.0)` 改成 `new(60.0)`，观察名次是否还稳——体会"k 越大名次差异越被压平"。
2. **追入口判定**：在 `query.rs:1113` 打个断点或加 `dbg!`，写一段"只给向量、不给 FTS"和"两者都给"的查询，确认前者走 `inner_execute_with_options`、后者走 `execute_hybrid`。
3. **证明 `norm` 对 RRF 无效**：同一组数据，分别 `.norm(NormalizeMethod::Rank)` 和不设 `norm`，用默认 RRFReranker 跑混合搜索，确认结果顺序**完全一样**（因为 RRF 不看分数）。这是理解 §5.3 那个困惑点的最好实验。
4. **故意违约**：照 §6 写一个**不加 `_relevance_score` 列**的玩具 reranker，挂上去跑，确认 `check_reranker_result`（`rerankers.rs:99`）报出 `Error::Schema`。
5. **空表健壮性**：建一张空表跑混合搜索，跟踪 `query_schemas`（`hybrid.rs:65`）走到哪个 `match` 分支、`rank`/`normalize_scores` 的空分支（`hybrid.rs:33,141`）是否生效、全程不 panic。

---

## 9. 小结 & 下一篇

- **混合搜索**因"向量召回懂语义但漏关键词、FTS 召回精确但不懂语义"而生，两路盲区互补（§1）。
- **`Reranker`** 是唯一扩展点（评审岗合同）：必做 `rerank_hybrid`、可白嫖 `merge_results`、硬约束是产出 `_relevance_score` 列（§2）。
- **RRF**（倒数排名融合）不看分数只看名次，文档在某路第 `i` 名得 `1/(i+k)` 分，多路相加按总分降序；缺省 `k=60`。真正自研的核心只有那段 `BTreeMap` 累加循环（§3）。
- **`hybrid.rs`** 的 `rank`/`normalize_scores`/`query_schemas` 是调 reranker 前的预处理，全靠 Arrow 算子，对空批次都有兜底分支（§4）。
- **`execute_hybrid`**（`query.rs:1014`）是 fan-out/fan-in 控制流：拆两路 → `try_join!` 并发 → 强制 `_rowid` 对齐 → 归一 → 交 reranker → 校验 → 截断 → 出菜（§5）。
- 全模块再次印证"**组装而非重造**"：排序、去重、归一、排名全是 Arrow 现成算子。

**贡献切入点**：`rerankers.rs:55` 的 TODO（扩展单独向量/FTS 重排）、自定义加权 reranker、混合搜索的更多归一化策略——都是好的 good-first-issue 落点。

下一篇将走出查询体系，进入 **`14 Embedding 体系`**：用户写入文本时，那些"由 embedding 函数算出来的列"（呼应 09 篇 `ColumnKind::Embedding`）是怎么注册、怎么在写入与查询时自动向量化的。
