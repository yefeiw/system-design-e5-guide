# Q7 · Ticket Booking (Flash Sales)

> Difficulty: Medium ｜ archetype：concurrency resource contention（consistency vs availability）
> Hello Interview flagged **E5 high-frequency problems**。精髓在于：ticket-grabbing 的瞬间是**strongly consistent 性问题**，其余一切（search、impression、payment）都是干扰项。

## 1. Problem Statement

"设计一个 Ticketmaster：user search 活动、seat selection 位（或 ticket-grabbing）、place order + pay。Taylor Swift 级别的开售：50 万人同时抢 5 万张票。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| seat-selection mode（specific seat）还是 seat-grab mode（first-come-any-seat）？| 两种 concurrency model 完全不同 |
| 允许 overselling 吗？（航空公司故意 105%）| **consistency requirement 的定盘星** |
| place the order 后锁定多久？（购物车持有期）| 锁的粒度和时长 |
| payment async 吗？ | order state machine 复杂度 |
| 候补 queue（waitlist）要不要？ | architecture 的第四种答案 |

## 3. Estimation Walkthrough

```
50 万人开售瞬间涌入；peak request 500K user × 每秒 refresh 1 次 = 500K QPS
可售 inventory：5 万张 —— 真正需要"strongly consistent 竞争"的 write 只有 ≤ 5 万次
takeaway → 99% 的 traffic（浏览、refresh、query 余票）根本碰不到 inventory，
       只要把 read 和 write separate，write 冲突其实很小 —— 这个洞察决定整个 architecture
```

## 4. High-Level Design

```
开售前：静态化活动页 + CDN（search/详情 100% cache，零回源）
开售中：
  user ──▶ queueing gateway（Virtual Waiting Room：ID issuance + 令牌 + heartbeat）
              │（按批次放行，如每 5 秒放 2000 人）
              ▼
          余票 query（Redis cache inventory 快照，second-level refresh）
              │有余票
              ▼
          place the order service ──▶ inventory 扣减（atomic 操作，见 deep dive A）
              │成功 │失败
              ▼ ▼
          order 创建（15 min payment window） 重新 queueing/候补
              ▼
          payment（async 回调）──▶ 出票（座位/票码，write 库，eventually consistent）
```

**architecture 叙事**："我用**queueing 室把不可控的 traffic surge 变成可控的匀速流**——user 拿号入场而不是全量打到 business 层；进场的 request 才碰 inventory。这个决定同时解决了 overload 和 fairness 性（先到先得）两个需求。"

## 5. Data Model

```sql
events (event_id, venue, onsale_at, total_seats)
seats (seat_id, event_id, section, row, num, status) -- seat-selection mode core 表
       status: AVAILABLE → HELD → SOLD / 回 AVAILABLE
orders (order_id, user_id, event_id, status, expires_at)
       status: PENDING → PAID → ISSUED / EXPIRED/CANCELLED
inventory counter: Redis inventory:{event_id}（seat-grab mode 用 counter，seat-selection mode 按 seat 行锁/optimistic locking）
```

## 6. Deep Dives

### Deep Dive A · inventory 扣减的 concurrency correctness（主菜）

**approach 谱系（从低到高）**：

1. **database optimistic locking**：`UPDATE seats SET status='HELD' WHERE seat_id=? AND status='AVAILABLE'`（CAS semantics，影响行数=1 即成功）
   - 简单正确；单行 hot spot 下 DB 可能到几千 QPS —— 配合 queueing rate limiting 后**往往就够用了**
2. **Redis atomic 扣减**：`DECR` / Lua（查+扣 atomic）—— 扛 10 万 QPS
   - **关键坑：Redis 成功、DB 失败怎么办** → async 落库 + reconciliation；Redis 挂了怎么办（AOF + 快照，最坏丢 second-level inventory state → 恢复时从 DB 全量重建）
3. **预扣额度到 memory 分段**：inventory 分 100 段，每段独立扣——分散 hot spot，最后一段 merge
   - 复杂度高，只在单行成为 bottleneck 时才上

**E5 收口**："先算真实冲突量：5 万张票最多 5 万次成功扣减，queueing 后 concurrency write 只有几千 QPS——**单行 optimistic locking 就是正解**，Redis 预扣是过早优化。真正难的不是扛量，是**state machine 的 correctness**。"（impression"approach matching scale"的判断力）

### Deep Dive B · 锁与持有期（seat-selection mode）

- HELD state + `expires_at`：15 分钟内不 payment auto-release
- release 路径：定时扫描（慢）→ **latency queue**（Redis ZSet/LevelDB 按到期时间）→ 到期 event 驱动 release
- "过期 release 必须是 system 的主动行为而不是等 user 动作——否则 scalper 脚本锁座不付款能锁死全场"

### Deep Dive C · payment 的 eventually consistent

- order state machine：PENDING → PAID → ISSUED，每个转换 idempotent（payment 回调会 retry）
- payment 成功但出票失败：reconciliation 任务扫 PENDING_PAID → 补偿（退款 or 人工）
- "payment gateway 回调至少 delivery 一次，我的 state 迁移必须 idempotent：`UPDATE orders SET status='PAID' WHERE order_id=? AND status='PENDING'`——又是 CAS。"

### Deep Dive D · 不 overselling 的形式化论证

被问"你怎么保证绝对不 overselling"：
> "所有路径汇到同一个串行点：每个 seat 行的 CAS。Redis 快照可以 stale（显示有余票但 place the order 失败 → 提示 retry，user 体验问题，不是 correctness 问题）；**impression 层允许乐观，成交层绝对悲观**。不 overselling 的证明 = 所有成交都过了那一条 CAS UPDATE。"
（"impression 乐观、成交悲观"这句话本身就是 E5 level 的总结）

### Deep Dive E · anti-scalper（加 partition）

- 设备指纹 + 账号风控前置（进 queueing 室之前过滤）
- 限购（per-user quota，place the order service memory 校验 + 落库 reconciliation）
- 手机验证码拉开人类 response 节奏

## 7. Red Flags

- 上来讨论 search architecture（审题失败——这题考 consistency 不是考 search）
- 用 distributed lock 锁整个 event（粒度灾难）而不讨论行级 CAS
- Redis 扣 inventory 但不讲 DB consistency
- 没有 payment timeout release 路径
- 用 zookeeper/etcd leader election 当卖点（解决的是错误的问题）

## 8. One-Minute Elevator Pitch

"先把 read write separate：静态页全 CDN、余票 read cache 快照（impression 层允许乐观）；traffic surge 用虚拟 queueing 室整流成匀速流再进 place the order。成交层的 correctness 靠每座位一行的 CAS UPDATE（AVAILABLE→HELD→SOLD state machine + 过期 latency queue release + idempotent payment 回调），Redis 预扣只有在单行 hot spot 实测不够时才加，并配 reconciliation fallback。真实冲突量只有 5 万次成功 write，approach matching scale 即可，不 overselling 的证明是'所有成交汇于 SPOF 串行化'。search 和推荐不是本题采分点。"

---
← [Q6 message queue](06-distributed-message-queue.md) ｜ [Q8 ad click aggregation →](08-ad-click-aggregator.md)
