# Q2 · Rate Limiter

> Difficulty: Easy–Medium ｜ archetype：algorithmic deep dives + distributed consistency
> 题目小，但**algorithm 层四档 comparison + distributed clock 问题**能一路问到 E5 的知识底。

## 1. Problem Statement

"设计一个 API rate limiter，防止 abuse。要支持每 user 每秒/每分钟 N 个 request。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| 限谁的 dimension？（IP / user / API key / global）| 决定 rate limiting key design |
| hard limit 还是 soft limit？（超了 return 429 还是 queueing）| availability 取向 |
| distributed or single-machine rate limiting？ | 本题 dividing line |
| rule 要动态 push-down 吗？ | 引入 config center |
| rate limiting 的 semantics：average rate vs burst capacity | 直接指向 algorithm 选择 |

## 3. estimation（简短，本题不在 numbers 上纠缠）

"假设 100K QPS peak、2000 万活跃 key——rate limiting 判断本身必须 <1ms P99 且不成为新 bottleneck，所以它必须是**memory 级操作**，这是本题第一设计约束。"

## 4. High-Level Design

```
                        ┌────────────────────────────┐
client ──▶ API Gateway ──▶ rate limiting middleware（in-process）──▶ business service
                          │ ↑ rule push-down（config center）
                          │ ↕ async counter sync（Redis，eventually consistent）
                          └────────────────────────────┘
rate-limited request ──▶ 429 + Retry-After ──▶ （可选）Kafka record rate-limited event
```

**关键 architecture 决策**："rate limiting 在**gateway/middleware in-process 做**，Redis 只做跨 instance 的 counter convergence（async/quasi-sync）。每 request 一次 Redis sync call 会把 rate limiter 自己变成 20 万 QPS 的 Redis cluster——得不偿失。"

## 5. Data Model

Redis：`rate:{user_id}:{window_start}` → counter / token bucket state（Lua 脚本 atomic read-modify-write）。
rule 中心：`rule_id → {key_template, algorithm, limit, window, burst}`。

## 6. Deep Dives

### Deep Dive A · 四种 algorithm（must-know，背成 comparison 表）

| algorithm | 原理 | 优点 | fatal 缺点 |
|------|------|------|---------|
| fixed window | 每 N 秒 counter 清零 | 最简单、省 memory | **window boundary burst**：两个 window 交界可放行 2N |
| sliding window log | record 每个 request timestamp | exact | memory O(QPS×window)，大 traffic 不可用 |
| **sliding window counter** | 当前 window counter ×(1-elapsed fraction) + previous window counter ×elapsed fraction | approximate 平滑、O(1) memory | 两 window 交界处仍是 approximate |
| **token bucket** | bucket capacity b，rate r tokens added at a constant rate | **允许受控 burst**、industry default（Guava/WFF） | parameter semantics 要讲清（b 和 r separate） |

E5 表达："default 答案 token bucket——因为它把『average rate』和『burst capacity』分成两个 orthogonal parameter，符合真实 traffic formats。如果 business 明确不允许任何 burst（比如保护脆弱 downstream），换 leaky bucket shaping。Cloudflare production 用的是 sliding window counter（他们的 blogs 值得 read）。"

### Deep Dive B · distributed：sync vs async（本题的 E5 dividing line）

**sync mode**（每 request 查 Redis + Lua atomic 判断）：
- exact，但 Redis 成 critical path——挂了 rate limiter 全挂
- mitigate：Redis cluster + consistent hashing 分 key

**async mode**（local 判断为主，定期和 Redis reconciliation）：
- Redis 挂了照样 rate limiting（degrade to standalone rate limiting）→ **availability 优先**
- cost：多 instance 合计会**短暂 over-limit**（比如允许 1.1N）——"对 anti-abuse scenario，10% 的 over-limit 换取 rate limiting system 自身永不成为 failure 点，这个 trade-off 我主动接受并 write 进设计文档。"

被问"必须 exact 怎么办"→ sync mode + Redis HA + "exact rate limiting 本身就是个 strongly consistent 需求，cost 要讲给 business 听"。

### Deep Dive C · clock 与 fairness

- 各 instance clock drift 对 window 计算的影响（用 monotonic clock + Redis 中心 timestamp calibration）
- sliding window 的 memory 问题在多 key 下被放大 → sharding 按 key hash
- cold start：service 重启后 local counter 清零 → 从 Redis 快速 warm-up

### Deep Dive D · rate limiting 之后的 response 设计

- 429 + `Retry-After` 头 + `X-RateLimit-Remaining`（API 友好性，很多 senior 都漏）
- tiered rate limiting：gateway tier coarse-grained（IP）+ service tier fine-grained（user+endpoint）
- rate-limited event 进 Kafka → attack detection / rule tuning

## 7. Red Flags

- 只会"fixed window counter + Redis INCR"一档，说不出 boundary burst 问题
- 把 Redis sync call 放 critical path 却不讨论它挂掉的 scenario
- 不知道 token bucket 的 burst 与 rate 是两个 parameter
- 没有 degradation approach（rate limiter 自己把全站打挂 = 设计 incident）

## 8. One-Minute Elevator Pitch

"in-process token bucket 做判断（r 控 average、b 控 burst），Redis 做跨 instance counter convergence——sync mode exact 但把 Redis 放进 critical path，我 default async reconciliation + degrade to standalone rate limiting，接受 10% over-limit 换 availability，anti-abuse scenario 这个 trade-off 成立。rule 从 config center push-down 支持 hot reload，rate-limited request 带 Retry-After return，event 进 Kafka 做分析。如果 downstream 是 billing 类必须 exact，我会换 sync + Redis cluster 并把 cost 讲清楚。"

---
← [Q1 short URL](01-url-shortener.md) ｜ [Q3 Top-K →](03-top-k-heavy-hitters.md)
