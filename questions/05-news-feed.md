# Q5 · News Feed

> Difficulty: Medium ｜ archetype：Fan-out write amplification + read-heavy
> Meta 系最爱。**Push vs Pull 的混合 approach**是主轴，consistency 细节是 E5 区分度。

## 1. Problem Statement

"设计 Facebook/Twitter 的 News Feed：user posting、follow 别人、刷到自己 follow 者的 timeline Feed。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| Feed ranking：time order 还是 algorithm ranking？ | 决定要不要 ranking service（本题先按 time order，ranking 提一句）|
| follow 上限多少？（follow 300 人 vs 100 万人）| **Push/Pull 的 dividing line** |
| followers structure 里有没有 celebrity user（百万人 follow）？| 同上，hot spot 问题的根源 |
| 新 Feed 的 refresh semantics？ | read-your-writes / eventually consistent 的选择题 |
| 要不要支持"推文浏览数"这类轻互动？ | write 入 storm variant |

## 3. Estimation Walkthrough

```
100M DAU；per-user follow 300 人、日发 2 帖、刷 20 次
write：200M posts/day ≈ 2.3K posts/s，peak 10K/s
read：2B feed request/day ≈ 23K QPS average，peak 100K+ QPS ← read 是 write 的 20 倍
write fan-out cost：一篇帖 × follow 者 average 300 = average 600K feed-item write/s（peak 3M/s）
takeaway → write fan-out 的 write QPS 是 posting QPS 的 ~300 倍，这就是本题全部矛盾的来源
```

## 4. High-Level Design

```
posting：client ──▶ Post API ──▶ 帖子 service ──▶ 帖子库（Post Store, Cassandra/sharding）
                                        │
                                        ▼
                              Kafka（fanout event）
                                        │
                              ┌─────────┴─────────┐
                              ▼ ▼
                     Fanout Worker（push 路线） follow 图谱 service（Graph）
                       给每个 followers 的 feed cache
                       追加 post_id │
                              │ │
                              ▼ ▼
刷 Feed：Feed API ──▶ user Feed Cache（Redis List，推的 timeline）
                       │（cache miss / 混合路线）
                       └──▶ Feed 现场组装（pull）：查 follow 列表 → 批量拉对方近期帖子 → 归并
```

## 5. Data Model

```
posts (post_id PK, user_id, content, media_refs, created_at) -- sharding：post_id
follows (follower_id, followee_id, created_at) -- 边表，sharding：follower_id
            （倒排表 followees_of:{user} 用于 pull；正排 followers_of:{user} 用于 push）
feed_cache: Redis List per user（存 post_id，长度封顶如 800）
inbox (user_id, last_read_post_id) -- 游标
```

## 6. Deep Dives

### Deep Dive A · Push vs Pull vs 混合（主菜）

| | Push（write fan-out）| Pull（read fan-out）|
|---|---|---|
| read latency | 极低（cache 现成）| 高（on-the-fly aggregation 300 人的帖子）|
| write amplification | 巨大（×average followers 数）| 无 |
| storage | 每人一份 feed replica | 无 replica |
| 新帖可见性 | async latency（second-level）| 立即 |
| celebrity user | **灾难**（一条帖 write 100 万次）| 天然免疫 |

E5 standard answers——**混合**：
- 普通 user（followers < 阈值如 10K）：push 预算进 followers cache
- celebrity user：不扩散，**pull 时现场 merge**（feed API return = cache 里的普通帖 + online pull 的 celebrity user 帖，按时间归并）
- "阈值是可调 parameter，按 fanout worker 的消费 latency monitoring 调——**把 system 行为和 operations 指标挂钩**才是完整设计。"

### Deep Dive B · consistency：发完帖 refresh 看不到（Meta 爱问）

scenario：user A posting → message 还在 Kafka → A 立刻 refresh Feed。
- approach 1：**read-your-own-writes**——Feed API 检查 `inbox.last_read`，若 user 自己有未进 cache 的最新帖，从帖子库直接拼上
- approach 2：posting API sync write 自己的 feed cache（自己必然是自己的 followers……不是），至少 sync write 自己的"主页 timeline"
- script："对外人，eventually consistent second-level convergence 完全 acceptable；对作者本人，人类对'我发的东西立刻可见'零容忍——**consistency 预算按 user 关系分配**，这是产品的心理学，不只是技术。"

### Deep Dive C · Feed Cache 细节

- structure：Redis List of post_id，LPUSH 新帖 + LTRIM 封顶（800 条，翻页深处回源帖子库）
- 翻页游标：`(last_post_id, offset)` 双 parameter——"纯 offset 在有新帖 write 入时会跳条/重复，cursor 必须锚定 post_id"
- cache 未命中（冷 user）：现场 pull 组装 + 回填
- hot spot user Feed cache sharding：按 user_id consistent hashing

### Deep Dive D · 删帖与编辑的传播

Push model 的隐藏债务：**删帖要从所有 followers 的 cache 里抠掉**。
- feed cache 存 post_id 而不是内容 → 删帖只需在帖子库打 tombstone，read 侧渲染时过滤 invalidation 帖（惰性删除）
- "存 ID 不存内容，把内容 convergence 到单一来源（Single Source of Truth），cache 才敢大"——这是 push model 能活下来的前提

### Deep Dive E · ranking（一句话级别）

"真实产品是 algorithm ranking（互动预测分），那会把 feed 组装变成**candidates 检索 + 打分**两段式（recall 几百条 → 轻量 model 打分取几十条），architecture 主体不变，加一个 ranking service。今天我先做时间轴。"

## 7. Red Flags

- 纯 push 且没讨论 celebrity user write amplification
- 纯 pull 且没算 300 人 on-the-fly aggregation 的 read latency
- feed cache 存帖子全文（删帖/编辑灾难）
- read-your-own-writes 没 awareness（被 interviewer 问"你自己发的刷不到"才反应）
- pagination 用纯 offset

## 8. One-Minute Elevator Pitch

"read 是 write 的 20 倍，core 矛盾是 fan-out write amplification。混合 approach：普通 user push——帖子经 Kafka 进 fanout worker write 进各 followers 的 Redis feed cache（存 post_id 不存内容，删帖走 tombstone 惰性过滤）；celebrity user 不扩散，read 时现场 pull 归并，阈值按 fanout 消费 latency 调。consistency tiered：对外人 eventually consistent，对作者本人 read-your-own-writes sync 拼入。pagination 用 post_id 锚定的 cursor。algorithm ranking 是把 assembly layer 换成 recall+打分两段式，architecture skeleton 不变。"

---
← [Q4 chat system](04-chat-system.md) ｜ [Q6 message queue →](06-distributed-message-queue.md)
