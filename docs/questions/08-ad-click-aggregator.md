# Q8 · Ad Click Aggregator

> Difficulty: Medium–Hard ｜ archetype：write-heavy + peak shaving + billing accuracy
> Hello Interview flagged **E5 high-frequency problems**，覆盖流处理全家 bucket：dedup、window、exactly-once、lambda/kappa。和 Q3 Top-K 是姊妹题但多了「钱」的 dimension。

## 1. Problem Statement

"设计一个 ad data pipeline：收集 ad impression（impression）和 click（click）event，aggregation 出每个 ad 的 CTR（CTR）、按 dimension 出 reports，并给 billing system 提供准确的 click bill。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| event order of magnitude？latency requirement（billing 要 hour-level，reports 要 day-level？）| 决定流/批 tiered |
| **click 怎么和 impression 关联**（click late 怎么办）？| 本题 core 技术难点 |
| 重复 click（fraud/retry）dedup window？ | billing accuracy |
| reports dimension 有哪些（advertiser×时间×地域）？| dimensional modeling |
| real-time anti-fraud 要不要？ | scope control |

## 3. Estimation Walkthrough

```
1B impressions/day ≈ 12K/s average，peak 60K/s；CTR 1% → 10M clicks/day
raw event（retain 30 天用于 audit/recompute）：1.1B × 200B × 30 ≈ 6.6TB —— object storage
after aggregation（ad×小时×dimension）：GB 级 —— OLAP 库轻松
takeaway → 两个 storage 世界：raw event 进湖（immutable、可 recompute），
       aggregation results 进 service storage（低 latency query）。architecture 必然是 streaming + batch layering
```

## 4. High-Level Design

```
SDK/frontend ──▶ event ingestion API（校验、tagging）──▶ Kafka（impressions / clicks 各自 topic）
                                              │
             ┌────────────────────────────────┤
             ▼ ▼
      real-time 流（Flink/Beam） batch layer（夜间 Spark）
      · dedup（click 对 impression） · 全量 recompute exact 值
      · session window 关联（click↔imp） · audit reconciliation
      · minute-level pre-aggregation │
             │ │
             ▼ ▼
      Redis / memory OLAP（real-time 预览） OLAP（ClickHouse/BigQuery，reports）
             │ │
             └────────────► query service ◄────────┘
                                │
                                ▼
                      billing system（以 batch layer exact 值为准）
```

**叙事主线**："real-time layer 给 ops 看（minute-level、approximate、可 recompute），batch layer 给钱（小时/day-level、exact、audit）——**同一份 data 两条 timeline**，这是 lambda 的教科书 scenario。"

## 5. Data Model

```
event（immutable，JSON→Parquet into the data lake）:
  impression: {imp_id, ad_id, user_id, page, geo, ts}
  click: {click_id, imp_id(FK), ad_id, ts}

aggregation table（OLAP，按 (ad_id, hour, dim...) clustering）:
  ad_stats_hourly (ad_id, ts_hour, geo, imp_count, click_count, ctr)
dedup state（streaming layer，RocksDB/Flink state）: click_id → TTL window
```

**关键点**：click 里带 `imp_id` foreign key——"关联不是靠 event time 相近去猜，是靠 ID foreign key exact 关联；**event ordering 不可信，ID 关系才可信**。"

## 6. Deep Dives

### Deep Dive A · Click 和 Impression 的 out-of-order correlation（本题的 E5 拷问点）

问题：click event 先到、impression 后到（不同端、不同链路 latency）怎么办？

- **event time session window**：click 和 impression 按 `imp_id` keyBy 到同一流分组，window 内 join；out-of-order 靠 **watermark**（如允许 5 分钟 late-arriving）
- 超过 watermark 的 late-arriving click → side output → sink to the data lake，batch layer 收编
- "CTR 的分子分母必须来自**同一时间窗的同一 counting semantics**——streaming layer 5 分钟 watermark 内的 join 算 real-time counting semantics，全天 recompute 算最终 counting semantics，两者永远有差异，reports 上要 annotate。"（counting semantics awareness = data system 的 senior signal）

### Deep Dive B · dedup 与 anti-fraud（billing 的钱 dimension）

- 重复 click：`click_id` idempotent + window 内 `user_id+ad_id` frequency rule（连点 10 次只计 1）
- state storage：streaming layer local RocksDB state（不进外部 Redis，避免 state 双 write 不一致）；state TTL = dedup window
- fraud signal（提一句 impression 广度）：IP 段 clustering、无 mousemove、click/impression 时距分布异常 → rule engine tagging → **billing 侧 exclude**
- "billing 的单位是'valid clicks'——dedup 和 anti-fraud rule 必须**在 billing 之前**完成，这就是 streaming layer 存在的理由，不只是为了好看的大盘。"

### Deep Dive C · Exactly-once 落到 OLAP

- streaming layer 计算 exactly-once：Flink checkpoint（two-phase commit barrier）+ sink transactional writes
- **replay 能力是一切的 fallback**：Kafka retention 之外 raw event 全量 sink to the data lake——"streaming layer 挂了/逻辑改了，从湖里 replay recompute，**pipeline 是可 re-run 的程序而不是一次性 data flow**"（kappa 的可 replay 思想 + lambda 的批 exact 层，混合即答案）
- idempotent write OLAP：以 (ad_id, hour, dim) 为 primary key upsert 覆盖

### Deep Dive D · traffic surge 与 backpressure

- 大促/超级碗时刻 traffic ×50：采集 API 丢弃策略 tiered（billing events 绝不丢、telemetry 可 sampling）
- Kafka 做缓冲，consumer 按能力 pull；**backpressure 传导**到采集端 return 503+Retry-After 让 SDK backoff
- "丢什么的 priority 是产品决策不是技术决策——我会把三类 event 的 SLA write 成表让 business 签字。"

## 7. Red Flags

- 只有 real-time layer 没有 batch layer（billing 不能靠可能重复/丢失的流）
- click 和 impression 靠 timestamp nearest matching
- 没有 dedup 讨论（重复 click 直接进 bill = billing incident）
- aggregation results 直接 write MySQL 供 reports（dimensional aggregation query 会 overwhelm OLTP）
- 没有 late data/watermark 概念

## 8. One-Minute Elevator Pitch

"两层 data flow：Kafka 承接 raw event（impression/click 双 topic，click 带 imp_id foreign key exact 关联而非时间猜 matching）；real-time layer Flink 做 dedup（click_id idempotent + user frequency rule）、watermark out-of-order join、minute-level pre-aggregation 进 memory OLAP 给 ops 看；batch layer 夜间全量 recompute 落 ClickHouse，**billing 以 batch layer 为准**。raw event 全量 into the data lake 保证可 replay——streaming layer 挂了从湖 re-run。traffic surge 时按 event tiered dropping（billing events 永不丢）。real-time 和最终 numbers 永远有差，reports annotate counting semantics，这就是 lambda 的 cost 与 benefit。"

---
← [Q7 Ticket Booking](07-ticket-booking.md) ｜ [Back to 06 Overview](../06-classic-questions-overview.md)
