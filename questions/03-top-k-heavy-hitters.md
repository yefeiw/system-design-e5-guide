# Q3 · Top-K / Heavy Hitters

> Difficulty: Medium ｜ archetype：write-heavy + peak shaving（streaming algorithm）
> E5 high-frequency problems。考的是**accuracy vs memory vs latency**的三角 trade-off，一道题覆盖整个流处理世界观。

## 1. Problem Statement

"设计一个 system，real-time 统计我们 service 中访问量最高的 10 个 URL / 被播放最多的歌曲 / 最热门的标签，比如按小时和按天两个粒度。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| "real-time"要多 real-time？（second-level / minute-level / hour-level）| 决定整套 architecture 是流还是批 |
| Top-K 要**exact**还是**approximate**（±1% error margin acceptable 吗）| 本题最重要的问题，直接决定 algorithm |
| 并列第 K 名怎么处理？ | boundary awareness |
| 全量所有时间 vs 滚动 window？ | 决定要不要 window aggregation |
| traffic 多大？（1B events/day order of magnitude?）| memory 可行性计算 |

## 3. Estimation Walkthrough

```
1B events/day ≈ 12K events/s average，peak ×5 ≈ 60K/s
dedup 后不同 item 数（cardinality）：假设 100M 个不同 URL
exact counter memory = 100M × (8B key 指纹 + 8B counter) ≈ 1.6GB —— 其实可行！
takeaway → 先问清楚：如果 cardinality 可控，exact approach memory 扛得住，别急着上 approximate algorithm
```

**这步是 E5 关键**：很多人背了 Count-Min Sketch 就无脑上，但 100M cardinality exact hash table 也就 2GB，single machine 都放得下。**先算，再选 algorithm**。

## 4. High-Level Design

```
event 源 ──▶ Kafka（按 item_id partition，保证同 key 聚到同一 partition）
              │
              ├──▶ stream aggregator（每 partition 维护 item→count hash table + min-heap(top K)）
              │ │ 周期性（minute-level）输出 (item, count)
              │ ▼
              │ aggregation Top-K merger ──▶ query service（Redis/memory 存当前 Top-K）
              │
              └──▶ data lake（raw event 留存，nightly batch processing 校正 = exact fallback）
```

**architecture 叙事**："hot path Kafka 按 partition 分流，每 partition 流式 counter + local Top-K heap，轻量 merge 层把各 partition local Top-K merge 成 global——因为 **local Top-K 的并集必然包含 global Top-K**（每项的 global counter ≥ 任一 partition local counter，第 K 名在它最强的那个 partition 里必然进 local Top-K），所以 merger 只需要处理 K×partition count 个 candidates，不需要全量 ranking。"

能讲出上面括号里那个论证，就是这道题的 E5 时刻。

## 5. Data Model

- partition 内：`HashMap<item_id, count>` + min-heap（size K，O(n log K)）
- window data：minute-level local aggregation 落 Redis / 按小时 partition 落 Parquet
- service tier：`(window, rank) → (item, count)`，短 TTL

## 6. Deep Dives

### Deep Dive A · memory 不够怎么办：approximate algorithm 家族

当 cardinality 到几十亿、exact hash table 放不下时：

| algorithm | 空间 | error margin 方向 | 备注 |
|------|------|---------|------|
| Count-Min Sketch | O(k/ε·log n) | **只高不低**（one-way error margin）| 重击者 error margin 相对小；可 merge（各 partition sketch 可相加）|
| Space-Saving | O(K/ε) | one-way | 直接维护 approximate Top-K，工程常用 |
| Lossy Counting | batch processing 式 | one-way | 老牌但已被前两者盖过 |

E5 表达："approximate algorithm 在重击者（heavy hitter）上的相对 error margin 小——counter 100 万的 item 估成 102 万无所谓；error margin 全集中在 long tail，而 long tail 本来就不进 Top-K。**这就是 approximate algorithm 恰好适配 Top-K 问题的原因**，不是巧合是 structure。"

以及必说的 fallback："approximate 给 real-time 视图，nightly batch processing（Spark/Beam 对全天 data exact recompute）覆盖修正——**lambda architecture 的取舍**：real-time layer 牺牲精度，batch layer 牺牲时效，query 层 merge。"

### Deep Dive B · window semantics

- 滚动 window（tumbling）vs sliding window（sliding）vs session window——**主动问 interviewer 要哪种**
- sliding window 的流式实现：分钟 bucket + approximate combination（"最近 60 分钟 = 最近 60 个分钟 bucket 求和"，error margin ≤ 1 分钟 bucket）
- out-of-order event：watermark 机制——"data late-arriving 5 分钟内我会更新结果，超过 watermark 的丢进 side output offline 补偿"

### Deep Dive C · consistency 与 failure

- aggregator 挂了 → Kafka offset 还在，**从上次提交的 offset replay**（at-least-once）
- 重复 counter 怎么处理：idempotent（offset + item combination dedup window）或接受 error margin（counter 类 business 通常 acceptable）
- "如果这是**billing**不是统计呢？"——那就不能用 at-least-once 糊弄，要 exactly-once（Kafka transaction / two-phase commit）或 event sourcing + reconciliation。**问出这题就是 interviewer 在测你懂不懂 delivery semantics 的 cost**

### Deep Dive D · query 侧

- Top-K query QPS 高 → 预计算 + Redis cache，refresh 频率 = business"real-time"requirement
- 多 dimension Top-K（按国家×按品类）→ dimension combination 爆炸，只预计算 high-frequency combination，long tail 走 on-the-fly aggregation

## 7. Red Flags

- 上来就 Count-Min Sketch（没先算 exact approach 的 memory 可行性）
- 把全量 event 塞进一个 global heap（没有 partition local Top-K 的洞察）
- "real-time"不问清楚就开始设计
- 讲不出 local Top-K merge 的 correctness 论证
- data 丢了就丢了，没 delivery semantics awareness

## 8. One-Minute Elevator Pitch

"event 进 Kafka 按 item partition，每 partition 流式维护 counter hash + min-heap，local Top-K merge 出 global——merge 的 correctness 来自 global 第 K 名必然是其最强 partition 的 local 前 K。cardinality 一亿内 exact hash 才 2GB，我先算再决定要不要 approximate；cardinality 到十亿级换 Count-Min Sketch（error margin one-way 且集中在 long tail，恰好不伤 Top-K），nightly batch processing 做 exact fallback，这就是 lambda。window semantics、late data watermark、at-least-once replay 的重复 counter——统计 business acceptable，billing business 必须换 exactly-once，cost 是 throughput 减半。"

---
← [Q2 rate limiter](02-rate-limiter.md) ｜ [Q4 chat system →](04-chat-system.md)
