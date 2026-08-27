# Q6 · Distributed Message Queue (Design a Kafka)

> Difficulty: Medium–Hard ｜ archetype：write-heavy + peak shaving（自己造轮子版）
> 逆向题——不是"用 Kafka 设计 X"，而是"设计 Kafka 本身"。最考察**storage engine + delivery semantics**的底层功力。

## 1. Problem Statement

"设计一个 distributed message queue：producer 发 message，consumer subscription 处理。要高 throughput、不丢 message、支持一个 consumer group 并行消费。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| throughput target？latency target？（百万 QPS + ms 级？）| storage model 选择的输入 |
| message 大小？（KB 级 vs MB 级）| append-only log vs 逐条 index |
| delivery semantics requirement？at-least-once 还是 exactly-once？| 整题的 deep dive 方向 |
| message 要 retention period？（consume-and-delete vs 按 retention 保留）| **Kafka 与传统 MQ 的 dividing line** |
| 要不要 transaction message / latency message？| scope control |

## 3. estimation（本题用来论证设计决策）

```
1M msg/s × 1KB = 1 GB/s 入流；retention 7 天 = 600TB
单 node NVMe sequential writes ~2 GB/s → ordering append-only log 是唯一可行 write model
random write 只有 ~100 MB/s → 任何"每 message 一条 record"的设计直接死亡
takeaway → append-only log（append-only log）不是选择，是物理规律
```

## 4. High-Level Design

```
Producer ──▶ Broker cluster
              ┌───────────────────────────────────────┐
              │ Topic: orders │
              │ Partition 0 (broker 1) [log: seg0 seg1 seg2...] │
              │ Partition 1 (broker 2) [log: ...] │
              │ Partition 2 (broker 3) [log: ...] │
              └───────────────────────────────────────┘
              每 partition = Leader（write）+ ISR replica（synchronous replication）
元 data/leader election：Controller / ZooKeeper / Raft
Consumer Group：每组每 partition 恰好分配一个 consumer instance
              offset 提交到内部 topic（__consumer_offsets）
```

## 5. Data Model（core 洞察所在）

```
Partition = disk 上的 append-only log file 序列（segment）
每条 message 只有 8 字节的"primary key"：offset（partition 内的 monotonic 递增序号）
index = 稀疏 index（每 4KB 一条 offset→file position），二分查找定位 segment
```

**三连击 script**：
1. "message 不删除不修改、只追加——**把 random write 变成 sequential writes**，throughput 差 2 个数 order of magnitude，这是整个设计的第一性原理"
2. "消费不删 message，只移动 offset——**消费是 read + 指针**，所以一个 topic 可以被任意多组 consumer 重复消费（Kafka 'consume-and-delete'的 RabbitMQ 的本质区别）"
3. "partition 是并行度的 atomic 单位：Producer 按 key hash 到 partition（同 key ordered），消费并行度 ≤ partition count——**想清楚 business 要什么级别的 ordered**再定 partition count"

## 6. Deep Dives

### Deep Dive A · 不丢 message 的三段论（must-know）

逐段讲，每段都有坑：

**① Producer → Broker**：
- acks=0（fire and forget）/ acks=1（leader 落盘）/ **acks=all（ISR 全部落盘）**
- "acks=all + min.insync.replicas=2 + replication.factor=3 才是'不丢'的完整配置——只说 acks=all 不提 ISR 收缩（min.insync 不满足时拒绝 write）等于没配"
- retry + idempotent producer（broker 按 PID+seq dedup）

**② Broker 内部**：
- ISR（in-sync replicas）机制：落后太多的 replica 踢出 ISR
- "unclean.leader.election：允许落后 replica 当 leader = data 丢；禁止 = availability 降——**这就是 CAP 在 MQ 里的具象**"

**③ Broker → Consumer**：
- 先消费后提交 offset：at-least-once（崩溃 replay，可能重复）
- 先提交后消费：at-most-once（崩溃丢 message）
- "offset 提交时机 = delivery semantics 的选择器"

### Deep Dive B · Exactly-once 怎么实现

- idempotent producer 只解决单 partition 单 session → 跨 partition 需要**transaction**（two-phase commit 到内部 topic）
- 端到端 exactly-once = transaction + consumer 端**只 read 已提交**（read_committed）
- "Kafka Streams / Flink two-phase commit checkpoint 把 sink 一并纳入 transaction——cost 是 throughput 显著下降。**先问 business 能不能用'at-least-once + idempotent 消费'凑合**，大多数时候能，那就别为 exactly-once 付费。"（这句话本身是 E5 signal）

### Deep Dive C · 消费组 rebalancing（Rebalance）

- consumer 上下线 → partition reassignment → **rebalancing 期间全组暂停消费**（stop-the-world）
- 协调者（group coordinator）heartbeat/session timeout 探测
- 坑：GC 停顿被误判死亡 → jitter storm；调 session.timeout + heartbeat 频率 + rebalancing 协议升级（增量 rebalancing/cooperative）
- "消费端的 long tail GC 会放大成整个 queue 的消费 latency——**queue 的稳定性被最慢的 consumer 绑架**"

### Deep Dive D · 与传统 MQ 的 comparison wrap-up

| | Kafka model | RabbitMQ model |
|---|---|---|
| storage | log retention、offset 指针 | queue、consume-and-delete |
| throughput | 百万级 | 万级 |
| routing | 简单（topic+key） | 灵活（exchange routing）|
| 重复消费 | 天然支持 | 需要 DLX 变通 |
| latency message | 不原生 | 插件/原生 |

"selection 看三个问题：要 throughput 吗？要回放吗？要复杂 routing 吗？"

## 7. Red Flags

- 每条 message 一个 database 行 / 一条 Redis record（没 awareness 到 sequential writes 的物理优势）
- 说"acks=all 就不丢了"（没提 ISR 收缩）
- exactly-once 与 at-least-once + idempotent 混为一谈
- 没有 partition 与 ordered 性的讨论
- 不知道 rebalancing 会 stop-the-world

## 8. One-Minute Elevator Pitch

"第一性原理是把 random write 变 sequential writes：partition = append-only log + 稀疏 index，message immutable、消费只是移动 offset——因此支持多组重复消费和回放。不丢 message 三段配置：producer acks=all+idempotent retry、broker ISR synchronous replication + min.insync.replicas 拒绝 write、consumer 先消费后提交（at-least-once），需要 exactly-once 时上 transaction + read_committed，但我会先问 business 能不能用 idempotent 消费替代。partition 是并行和 ordered 的 atomic 单位——同 key 同 partition 才 ordered，partition count 即消费并行上限。rebalancing 是 stop-the-world storm，消费端稳定性（GC、长任务）是 queue 稳定性的隐藏约束。"

---
← [Q5 News Feed](05-news-feed.md) ｜ [Q7 Ticket Booking →](07-ticket-booking.md)
