# Q4 · Chat / Messaging

> Difficulty: Medium ｜ archetype: state sync（long-lived connection）
> 把「connection layer / real-time delivery pipeline / message delivery semantics / group chat fan-out」四件事讲清楚，这题就赢了。deep dive 可以无限深，是 impression 功底的好题。本章重点：**real-time 功能到底怎么实现**。

## 1. Problem Statement

"设计一个 WhatsApp 级别的 chat system：一对一聊天、group chat、online presence、typing indicator、read receipts、offline delivery。"

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| group chat 最大多少人？（1:1 / 百人群 / 万人直播群）| fan-out 复杂度差三个 order of magnitude |
| online presence 要多 real-time？| push model 的 selection 依据 |
| message 保序的 scope？（同一 session 内？global？）| **必须问**——保序是这题的技术 core |
| offline messages retention period？| storage estimation |
| multi-device sync 要不要？| sync semantics 复杂度翻倍 |
| 媒体 message（图片/videos）？| 分离媒体 pipeline（本题可先略，提一句）|
| app 被杀进程后 message 还要秒达吗？| 决定 push notification（APNs/FCM）在架构里的位置 |

## 3. Estimation Walkthrough

```
50M DAU，per-user 40 条/天 ≈ 2B msg/day ≈ 23K msg/s average（peak ×3 ≈ 70K/s）
同时 online user ≈ DAU × 10~20% ≈ 5~10M long-lived connection
单 connection gateway node（16GB memory）扛 ~100K connection → 需要 50~100 台 connection layer
（WhatsApp 真实数据：Erlang + FreeBSD 单机 ~2M connection——数量级参考，不是设计目标）
message 160B（类 WhatsApp）+ metadata ≈ 500B/条
storage = 2B × 500B × 2(收发双方) × 1 年 ≈ 700TB/年 → 必须 sharding
typing indicator / presence 不入 storage：纯 in-flight 信号，是 storage 估算里唯一"零成本"的部分
```

## 4. High-Level Design

```
                        ┌──────────── connection layer（stateless 化）────────────┐
client A ──MQTT/WS──▶ LB ──▶ Chat Gateway #1 ─┐
                                              ├─▶ routing service（user+device → gateway 映射，Redis TTL）
client B ──MQTT/WS──▶ LB ──▶ Chat Gateway #2 ─┘
                                                     │
   message pipeline：API ──▶ message service ──▶ Kafka（按 conversation_id partition）
                                  │
                                  ▼
                          message storage（Cassandra，按 session partition）
                                  │
                                  ▼
                          delivery service ──▶ 查 routing ──▶ target gateway ──ws──▶ client B
                                                       │（offline / app killed）
                                                       ▼
                                                  push service(APNs/FCM)

real-time 信号分层：
  durable    message本体      → 走完整 pipeline（persist → deliver → ack）
  semi-durable delivery/read state → 游标（inbox_state），reconnect 后可恢复
  ephemeral  typing indicator、presence → 不 persist，丢了就丢了，直接 fan-out 给 online 的对端
```

## 5. Data Model

```
Cassandra（write-heavy read point lookups、按 session clustering、naturally shardable）：
messages (
  conversation_id UUID, -- partition key
  message_id TIMEUUID,  -- clustering key，TIMEUUID 天然按时间 ordered = session 内保序
  sender_id, content, type,
  created_at
)

inbox_state (
  user_id,              -- partition key
  conversation_id,
  last_read_msg_id TIMEUUID,      -- blue tick 游标
  last_delivered_msg_id           -- double tick 游标
) -- 每 user 的 conversation list、未 read 数、receipts 游标——reconnect 后增量 sync 全靠它
```

**为什么 Cassandra**："按 conversation_id partition → 一个 session 的所有 message 在同一 node → session 内 ordering 由 clustering key 保证，**partition key + clustering key 的设计直接实现'session 内保序'这个 core 需求**，这是 KV wide-column model 和需求天然咬合的案例。"

## 6. Deep Dives

### Deep Dive A · Real-time transport 怎么选（must-know，本章核心之一）

| 方案 | 方向 | latency | mobile 电耗 | 适用 |
|------|------|---------|------------|------|
| HTTP short polling | pull | 差（间隔）| 高（radio 反复唤醒）| 反面教材 |
| long polling | pull→伪 push | 中 | 中 | WS 不可用时的 fallback |
| SSE | server→client 单向 | 好 | 低 | 只下行（股票行情），聊天不够 |
| WebSocket | 双向 | 好 | 低 | Web 端主力 |
| MQTT（over TCP/TLS）| 双向 | 好 | 极低 | **移动端 messaging 的行业答案**（WhatsApp/Facebook Messenger 类协议、大量 IM SDK）|

**为什么移动端 IM 偏好 MQTT 而不是裸 WebSocket**——这段是 E5 的差异化得分点：

- **协议开销**：MQTT fixed header 最小 2 字节，长连接心跳 PINGREQ 只有 2 字节；WebSocket frame + 应用层 JSON 的心跳动辄上百字节。千万级 connection 下，心跳流量差两个数量级。
- **QoS 内建**：MQTT 原生 QoS 0（fire-and-forget，配 typing indicator）/ QoS 1（at-least-once + message id 去重，配 message 本体）/ QoS 2（exactly-once 四步握手，IM 一般不用——太贵）。**"ephemeral 信号走 QoS 0、durable message 走 QoS 1"一句话就能体现分层思维**。
- **persistent session**：MQTT broker 可以替 client 保存 subscription + 离线 message（QoS 1），client 重连后自动补投——这就是 session resume 的协议级实现。
- **keepalive 内建**：协议自带 keepalive 协商，正好承载 presence。

**真实系统参照**：WhatsApp 服务端是 Erlang/OTP + FreeBSD（BEAM 的轻量 process 模型每 connection 一个 process，单机撑到 ~2M connection），协议是 XMPP 演化出的私有协议（后期向 MQTT 风格收敛）。面试不必背细节，但一句 "WhatsApp 用 Erlang 就是因为 per-connection process 的内存开销是 KB 级，这是 connection layer 选型的底层逻辑" 价值极高。

### Deep Dive B · Connection 生命周期与 session resume

一条 connection 的完整生命：

```
1. TCP + TLS 握手 → 2. 应用层 auth（token + device_id 绑定）
→ 3. 注册 routing（user+device → gateway，Redis，TTL = 3×heartbeat）
→ 4. 稳态：heartbeat（keepalive 间隔）+ 收发
→ 5. 断线 → jittered backoff 重连 → resume（带 last received id）
```

必须主动讲的四个坑：

- **heartbeat 间隔是被 NAT/防火墙 逼出来的**：运营商 NAT 表项超时常见 5 分钟级别，keepalive 必须小于它；但又不能太密——移动端 radio 从 idle 唤醒一次耗电可观。**生产常见 30s~2min，且 app 在前台/后台用不同间隔**（前台 30s，后台 5min 或直接断开靠 push notification）。
- **session resume 而不是全量 resync**：client 重连时带 `(conversation_id, last_received_msg_id)` 游标，gateway/delivery service 按 inbox_state 差额 back-fill。**每次地铁换乘都全量拉历史是灾难**——设计目标是 resume 成本 O(gap) 而不是 O(history)。
- **reconnect storm**：地铁到站、机房抖动 → 百万 client 同时重连。解法组合：client 侧 random jittered backoff（打散到达时间）；gateway 侧 admission control（按 connection 建立速率限流，宁可慢 5 秒不能雪崩）；auth 服务优先保障重连请求（区分"新 session"和"resume"）。
- **routing 表清理**：gateway crash 时它的 routing entries 靠 TTL 过期消失；gateway 优雅下线时主动清除并让 client 重连——"connection layer stateless 化"的关键就是 routing state 外置 + TTL 兜底。

### Deep Dive C · Real-time delivery pipeline：三种 tick 是三条不同的路径

WhatsApp 的 UI 状态机（✓ / ✓✓ / 蓝色蓝色✓✓）背后是三条不同的 server 路径，能拆开讲就是 E5 水准：

| UI 状态 | 语义 | 触发路径 |
|---------|------|---------|
| ✓ 单勾 | server 已接收并 persist | message service 写入 Cassandra 成功 → ack 回 sender（经 sender 的 gateway）|
| ✓✓ 双勾 | 已送达接收方设备 | delivery service 投递成功且 **client device 回 ACK**（不是"推出去就算"）→ 状态更新 event 回 sender |
| 蓝色✓✓ | 已读 | 接收方 app 打开 session → client 发 `last_read_msg_id` 游标 → server 更新 inbox_state → 通知 sender |

三个必须讲清的细节：

- **ACK 回环本身也是 real-time 推送**：双勾/蓝勾不是 sender 轮询来的，是 server 把状态变更 event 经 sender 自己的 long-lived connection 推下去。sender 断线期间 tick 更新怎么办？——上线时按 inbox_state（发送箱视角）补齐，**tick 状态最终一致，不是强实时**。
- **delivered 的定义要在 clarification 阶段锁死**："推到设备"（FCM/APNs 收到即算）还是"app 进程确认"（应用层 ACK）？后者更准但 app 被杀时永远到不了双勾——**这正是 push notification 路径存在的意义**：app 被杀 → FCM/APNs 通道投 system notification → 用户点开 → app 冷启动 → 走 resume 增量拉 → 补 ACK。
- **typing indicator 是 ephemeral 信号的标准案例**：不 persist、不保证送达（QoS 0）、client 侧 throttle（按 key 每几秒最多发一条）、gateway 直接查 routing fan-out 给对端——**完全绕开 message pipeline**。在白板上画出"durable 走管道、ephemeral 抄近道"的两条线，是本章最出彩的一笔。
- **presence 同理**：heartbeat 上报 → state service（Redis `user:{id}` TTL）→ 变更 event 只推给"当前打开与你的 session"的人（subscription 模型）。trade-off："state 允许 stale 30 秒，省下的是全员广播的数百万 QPS。没人会为'好友明明 online 却显示 offline'投诉。"

### Deep Dive D · message delivery semantics（E5 core 区）

三个 state machine：**发送中 → 已送达（server ack）→ read receipts（recipient ack）**。

必须主动讲的坑：

- **dedup**：client 重发（timeout retry）导致重复 → client generate message UUID，server idempotent dedup（QoS 1 的 message id 语义）
- **ordering**：接收端按 (conversation_id, message_id) ranking，不依赖到达 ordering
- **offline delivery**：上线后按 inbox_state 游标拉增量——"推拉结合：online 走 push，push 失败/offline 走 pull fallback"
- 至少一次 + idempotent = 事实上的恰好一次（**这套 combination 拳是 distributed message 的万能答案**）

### Deep Dive E · group chat fan-out

| 群 scale | 策略 |
|--------|------|
| ≤ 200 | write fan-out：一条 message × N 个收件人 inbox record（WhatsApp model）|
| 数千+ | read fan-out + write fan-out 混合：大群单独存一份，成员 pull；只给成员的 conversation list write 一条指针 |

"write fan-out 简单但 N 倍 storage/write amplification；read fan-out 省 write 但 online members 都要 real-time 拉。**dividing line 按群 scale 划**，这是 real-world systems（微信/微博）的成熟做法。"

Real-time 信号的群内 fan-out 另算：typing indicator 在 200 人群里若全员广播就是 200 倍放大——**ephemeral 信号可以采样/聚合/干脆只在 ≤N 人的群开放**，这是典型的 scope 控制。

### Deep Dive F · multi-device sync

每设备独立 `device_id` + 独立 read receipts/delivered 游标；message 按 session multi-device delivery；删除/撤回是 **sync event**（同样走 message pipeline push-down tombstone）。提一句就够，除非 interviewer 明确要。

## 7. Red Flags

- 用 HTTP short polling 做 core message 通道且讲不出 cost
- 讲了 WebSocket/MQTT 但答不出"server 怎么把 message 推给特定 user"（= 没有 routing 表概念）
- **没有 session resume 设计**（每次 reconnect 全量拉 → 被问"地铁通勤场景"当场卡住）
- **reconnect storm 没有应对**（million-scale 长连接系统的必考 follow-up）
- message 存 MySQL 单表、没有 sharding 故事（2B/day write 入）
- 没有 dedup/ordering 讨论（被问"network retry 重发怎么办"当场卡住）
- online presence 全员广播（没算过 QPS）
- read receipts feature 没有游标设计（每次全量比对？）
- typing indicator 也要 persist / 也要 ACK（没分层，资源浪费）

## 8. One-Minute Elevator Pitch

"connection layer 用 MQTT-over-TLS 长连接（移动端比裸 WebSocket 省 2 个数量级的心跳开销，QoS 0/1 天然区分 ephemeral 信号和 durable message），50–100 台 gateway、routing 表外置 Redis + TTL 实现 stateless 化。real-time 功能分三层：message 本体走 API → Kafka 按 conversation partition → Cassandra，partition key + TIMEUUID clustering 天然 session 内保序；delivery/read tick 是独立的状态 event，server 经对端自己的长连接推回去，断线靠 inbox_state 游标补齐；typing/presence 是 ephemeral 信号，不 persist、gateway 查 routing 直推。delivery semantics：QoS 1 语义的 client UUID idempotent dedup + 接收端按 ID 重排 + at-least-once pipeline = 事实 exactly-once。断线走 jittered backoff + session resume（增量补投，成本 O(gap)），app 被杀走 FCM/APNs system notification → 冷启动增量拉。group chat 按 scale 分界：小群 write fan-out、大群 read fan-out。媒体 message 走独立的 object storage + CDN pipeline，不占聊天链路。"

---

← [Q3 Top-K](03-top-k-heavy-hitters.md) ｜ [Q5 News Feed →](05-news-feed.md)
