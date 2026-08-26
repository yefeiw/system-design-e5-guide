# Q6 · 分布式消息队列（设计一个 Kafka）

> 难度：Medium–Hard ｜ 母题：写多 + 削峰（自己造轮子版）
> 逆向题——不是"用 Kafka 设计 X"，而是"设计 Kafka 本身"。最考察**存储引擎 + 投递语义**的底层功力。

## 1. 题目原文

"设计一个分布式消息队列：生产者发消息，消费者订阅处理。要高吞吐、不丢消息、支持一个消费者组并行消费。"

## 2. 澄清问题清单

| 问题 | 意图 |
|------|------|
| 吞吐目标？延迟目标？（百万 QPS + ms 级？）| 存储模型选择的输入 |
| 消息大小？（KB 级 vs MB 级）| 追加日志 vs 逐条索引 |
| 投递语义要求？at-least-once 还是 exactly-once？| 整题的深挖方向 |
| 消息要保留多久？（消费即删 vs 按 retention 保留）| **Kafka 与传统 MQ 的分水岭** |
| 要不要事务消息 / 延迟消息？| 范围控制 |

## 3. 估算（本题用来论证设计决策）

```
1M msg/s × 1KB = 1 GB/s 入流；retention 7 天 = 600TB
单节点 NVMe 顺序写 ~2 GB/s → 顺序追加日志是唯一可行写模型
随机写只有 ~100 MB/s → 任何"每消息一条记录"的设计直接死亡
结论 → 追加日志（append-only log）不是选择，是物理规律
```

## 4. 高层设计

```
Producer ──▶ Broker 集群
              ┌───────────────────────────────────────┐
              │ Topic: orders                         │
              │  Partition 0 (broker 1) [log: seg0 seg1 seg2...]  │
              │  Partition 1 (broker 2) [log: ...]     │
              │  Partition 2 (broker 3) [log: ...]     │
              └───────────────────────────────────────┘
              每分区 = Leader（写）+ ISR 副本（同步复制）
元数据/选主：Controller / ZooKeeper / Raft
Consumer Group：每组每分区恰好分配一个 consumer 实例
              offset 提交到内部 topic（__consumer_offsets）
```

## 5. 数据模型（核心洞察所在）

```
Partition = 磁盘上的 append-only 日志文件序列（segment）
每条消息只有 8 字节的"主键"：offset（分区内的单调递增序号）
索引 = 稀疏索引（每 4KB 一条 offset→file position），二分查找定位 segment
```

**三连击话术**：
1. "消息不删除不修改、只追加——**把随机写变成顺序写**，吞吐差 2 个数量级，这是整个设计的第一性原理"
2. "消费不删消息，只移动 offset——**消费是读 + 指针**，所以一个 topic 可以被任意多组消费者重复消费（Kafka '消费即删'的 RabbitMQ 的本质区别）"
3. "分区是并行度的原子单位：Producer 按 key 哈希到分区（同 key 有序），消费并行度 ≤ 分区数——**想清楚业务要什么级别的有序**再定分区数"

## 6. 深挖

### 深挖 A · 不丢消息的三段论（必考）

逐段讲，每段都有坑：

**① Producer → Broker**：
- acks=0（fire and forget）/ acks=1（leader 落盘）/ **acks=all（ISR 全部落盘）**
- "acks=all + min.insync.replicas=2 + replication.factor=3 才是'不丢'的完整配置——只说 acks=all 不提 ISR 收缩（min.insync 不满足时拒绝写）等于没配"
- 重试 + 幂等 producer（broker 按 PID+seq 去重）

**② Broker 内部**：
- ISR（in-sync replicas）机制：落后太多的副本踢出 ISR
- "unclean.leader.election：允许落后副本当 leader = 数据丢；禁止 = 可用性降——**这就是 CAP 在 MQ 里的具象**"

**③ Broker → Consumer**：
- 先消费后提交 offset：at-least-once（崩溃重放，可能重复）
- 先提交后消费：at-most-once（崩溃丢消息）
- "offset 提交时机 = 投递语义的选择器"

### 深挖 B · Exactly-once 怎么实现

- 幂等 producer 只解决单分区单会话 → 跨分区需要**事务**（两阶段提交到内部 topic）
- 端到端 exactly-once = 事务 + consumer 端**只读已提交**（read_committed）
- "Kafka Streams / Flink 两阶段提交 checkpoint 把 sink 一并纳入事务——代价是吞吐显著下降。**先问业务能不能用'at-least-once + 幂等消费'凑合**，大多数时候能，那就别为 exactly-once 付费。"（这句话本身是 E5 信号）

### 深挖 C · 消费组再平衡（Rebalance）

- consumer 上下线 → 分区重新分配 → **再平衡期间全组暂停消费**（stop-the-world）
- 协调者（group coordinator）心跳/会话超时探测
- 坑：GC 停顿被误判死亡 → 抖动风暴；调 session.timeout + 心跳频率 + 再平衡协议升级（增量再平衡/cooperative）
- "消费端的长尾 GC 会放大成整个队列的消费延迟——**队列的稳定性被最慢的消费者绑架**"

### 深挖 D · 与传统 MQ 的对比收尾

| | Kafka 模型 | RabbitMQ 模型 |
|---|---|---|
| 存储 | 日志保留、offset 指针 | 队列、消费即删 |
| 吞吐 | 百万级 | 万级 |
| 路由 | 简单（topic+key） | 灵活（exchange 路由）|
| 重复消费 | 天然支持 | 需要 DLX 变通 |
| 延迟消息 | 不原生 | 插件/原生 |

"选型看三个问题：要吞吐吗？要回放吗？要复杂路由吗？"

## 7. 红牌答案

- 每条消息一个数据库行 / 一条 Redis 记录（没意识到顺序写的物理优势）
- 说"acks=all 就不丢了"（没提 ISR 收缩）
- exactly-once 与 at-least-once + 幂等混为一谈
- 没有分区与有序性的讨论
- 不知道再平衡会 stop-the-world

## 8. 一分钟电梯版

"第一性原理是把随机写变顺序写：分区 = 追加日志 + 稀疏索引，消息不可变、消费只是移动 offset——因此支持多组重复消费和回放。不丢消息三段配置：producer acks=all+幂等重试、broker ISR 同步复制 + min.insync.replicas 拒绝写、consumer 先消费后提交（at-least-once），需要 exactly-once 时上事务 + read_committed，但我会先问业务能不能用幂等消费替代。分区是并行和有序的原子单位——同 key 同分区才有序，分区数即消费并行上限。再平衡是 stop-the-world 风暴，消费端稳定性（GC、长任务）是队列稳定性的隐藏约束。"

---
← [Q5 News Feed](05-news-feed.md) ｜ [Q7 票务预订 →](07-ticket-booking.md)
