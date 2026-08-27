# Q4 · Chat / Messaging

> Difficulty: Medium ｜ archetype：state sync（long-lived connection）
> 把「connection layer / message delivery semantics / group chat 扇出」三件事讲清楚，这题就赢了。deep dive 可以无限深，是 impression 功底的好题。

## 1. Problem Statement

"设计一个 WhatsApp/微信级别的 chat system：一对一聊天、online presence、message read receipts。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| group chat 最大多少人？（1:1 / 百人群 / 万人直播群）| 扇出复杂度差三个 order of magnitude |
| online presence 要多 real-time？| push model 的 selection 依据 |
| message 保序的 scope？（同一 session 内？global？）| **必须问**——保序是这题的技术 core |
| offline messages retention period？| storage estimation |
| multi-device sync 要不要？| sync semantics 复杂度翻倍 |
| 媒体 message（图片/videos）？| 分离媒体 pipeline（本题可先略，提一句）|

## 3. Estimation Walkthrough

```
50M DAU，per-user 40 条/天 ≈ 2B msg/day ≈ 23K msg/s average
同时 online user ≈ DAU × 10~20% ≈ 5~10M long-lived connection
单 connection gateway node（16GB memory）扛 ~100K connection → 需要 50~100 台 connection layer
message 160B（类 WhatsApp）+ 元 data ≈ 500B/条
storage = 2B × 500B × 2(收发双方) × 1 年 ≈ 700TB/年 → 必须 sharding
```

## 4. High-Level Design

```
                          ┌──────────── connection layer（stateless 化）────────────┐
client A ──ws──▶ LB ──▶ Chat Gateway #1 ─┐
                                          ├─▶ routing service（user → gateway 映射）
client B ──ws──▶ LB ──▶ Chat Gateway #2 ─┘ │
                                                   ▼
   message pipeline：API ──▶ message service ──▶ Kafka（按 conversation_id partition）
                                  │
                                  ▼
                          message storage（Cassandra，按 session partition）
                                  │
                                  ▼
                          delivery service ──▶ 查 routing ──▶ target gateway ──ws──▶ client B
                                                       │（offline）
                                                       ▼
                                                  push service(APNs/FCM)
online presence：heartbeat(minute-level) → state service(Redis TTL) → subscription 者增量 sync
read receipts：client ACK 携带 last_read_msg_id → write 回 → 反向通知对方
```

## 5. Data Model

```
Cassandra（write-heavy read point lookups、按 session clustering、naturally shardable）：
messages (
  conversation_id UUID, -- partition key
  message_id TIMEUUID, -- clustering key，TIMEUUID 天然按时间 ordered = session 内保序
  sender_id, content, type,
  created_at
)

inbox_state (
  user_id, -- partition key
  conversation_id,
  last_read_msg_id TIMEUUID,
  last_delivered_msg_id
) -- 每 user conversation list、未 read 数、read receipts 游标
```

**为什么 Cassandra**："按 conversation_id partition → 一个 session 的所有 message 在同一 node → session 内 ordering 由 clustering key 保证，**partition key + clustering key 的设计直接实现'session 内保序'这个 core 需求**，这是 KV wide-column model 和需求天然咬合的案例。"

## 6. Deep Dives

### Deep Dive A · long-lived connection 与 delivery model（must-know）

- **polling / long polling / WebSocket / SSE** 四档 comparison——手机端省电 vs real-time 性的 trade-off
- WebSocket 双向低 latency 但：connection state 在 gateway memory 里 → **gateway 必须处理重连 storm**（地铁里一车厢人同时断线重连：random backoff + connection count rate limiting）
- **routing 表**（user → gateway）放 Redis：gateway 上下线时 cleanup（heartbeat TTL），delivery service 查表 routing——"connection layer stateless 化"的关键就是 routing 外置

### Deep Dive B · message delivery semantics（本题的 E5 core 区）

三个 state machine：**发送中 → 已送达（server ack）→ read receipts（recipient ack）**。

必须主动讲的坑：
- **dedup**：client 重发（timeout retry）导致重复 → client generate message UUID，server idempotent dedup
- **ordering**：接收端按 (conversation_id, message_id) ranking，不依赖到达 ordering
- **offline delivery**：上线后按 inbox_state 游标拉增量——"推拉结合：online 走 push，push 失败/offline 走 pull fallback"
- 至少一次 + idempotent = 事实上的恰好一次（**这套 combination 拳是 distributed message 的万能答案**）

### Deep Dive C · online presence

- heartbeat（WebSocket ping，minute-level）→ Redis `user:{id}` 带 TTL——TTL 到期自动"下线"，不需要 cleanup 任务
- state storm：上线/下线 event 别全员广播 → **按 session subscription**（你的好友打开 session 才拉 state）
- trade-off："state 允许 stale 30 秒，省下的是数百万 QPS 的 state sync。聊天 scenario 没人会为'好友明明 online 却显示 offline'投诉。"

### Deep Dive D · group chat 扇出

| 群 scale | 策略 |
|--------|------|
| ≤ 200 | write fan-out：一条 message × N 个收件人 inbox record（WhatsApp model）|
| 数千+ | read fan-out + write fan-out 混合：大群单独存一份，成员 pull；只给成员的 conversation list write 一条指针 |

"write fan-out 简单但 N 倍 storage/write amplification；read fan-out 省 write 但 online members 都要 real-time 拉。**dividing line 按群 scale 划**，这是 real-world systems（微信/微博）的成熟做法。"

### Deep Dive E · multi-device sync

每设备独立 `device_id` + 独立 read receipts/已送达游标；message 按 session multi-device delivery；删除/撤回是**sync event**（同样走 message pipeline push-down tombstone）。提一句就够，除非 interviewer 明确要。

## 7. Red Flags

- 用 HTTP 短 polling 做 core message 通道且讲不出 cost
- message 存 MySQL 单表、没有 sharding 故事（2B/day write 入）
- 没有 dedup/ordering 讨论（被问"network retry 重发怎么办"当场卡住）
- online presence 全员广播（没算过 QPS）
- read receipts feature 没有游标设计（每次全量比对？）

## 8. One-Minute Elevator Pitch

"connection layer WebSocket gateway cluster（10M concurrency connection，50–100 台），routing 表外置 Redis 实现 connection layer stateless 化；message 走 API → Kafka 按 conversation partition → Cassandra 按 session partition，partition key+TIMEUUID clustering 天然实现 session 内保序。delivery semantics：client message UUID idempotent dedup + 接收端按 ID 重排 + at-least-once pipeline = 事实 exactly-once；offline 走 push service + 上线按游标增量拉。online presence heartbeat + TTL、按 session subscription、容忍 30 秒 stale。group chat 按 scale 分界：小群 write fan-out、大群 read fan-out。媒体 message 走独立的 object storage + CDN pipeline，不占聊天链路。"

---
← [Q3 Top-K](03-top-k-heavy-hitters.md) ｜ [Q5 News Feed →](05-news-feed.md)
