# Q4 · Chat / Messaging

> Difficulty: Medium ｜ archetype: state sync（long-lived connection）
> 这道题的难点不在"加一个 WebSocket"，而在于同时处理：real-time 投递路径、会话内保序、离线恢复、群聊扩展与端到端加密。本章主线：**real-time 功能到底怎么实现**——transport 怎么选、connection 怎么活、三种 tick 走哪条路、typing/presence 这类 ephemeral 信号怎么抄近道。

## 1. Problem Statement

"设计一个 WhatsApp 级别的 chat system：一对一聊天、group chat、online presence、typing indicator、read receipts、offline delivery、端到端加密（E2EE）。"

E2EE 不要求设计完整的密钥协商协议，但必须明确：**服务端路由和存储的是密文，不应读取消息明文**——这是架构边界，不是末尾随口补的一句。

## 2. Clarifying Questions

| 问题 | intent |
|------|------|
| group chat 最大多少人？（1:1 / 百人群 / 万人频道）| fan-out 复杂度差三个 order of magnitude |
| online presence 要多 real-time？| push model 的 selection 依据 |
| message 保序的 scope？（同一 session 内？global？）| **必须问**——保序是这题的技术 core |
| "已送达"由谁确认？（推送出去就算 / app 进程 ACK）| 决定 tick 状态机的确认点 |
| offline messages retention period？ | storage estimation |
| multi-device sync 要不要？撤回/编辑？ | sync semantics 复杂度翻倍 |
| 是否要求 E2EE？ | 决定服务端可见性、搜索能力、媒体上传流程 |
| app 被杀进程后 message 还要秒达吗？ | 决定 push notification（APNs/FCM）在架构里的位置 |

好的默认假设：消息正文保留一年；在线用户 ≈ DAU 的 10%–20%；presence 允许 30 秒左右 stale。

## 3. Estimation Walkthrough

```
50M DAU，per-user 40 条/天 ≈ 2B msg/day ≈ 23K msg/s average（peak ×3~5 ≈ 70K~115K/s）
同时 online user ≈ 5~10M long-lived connection
单 connection gateway node（16GB memory）扛 ~100K connection → 需要 50~100 台 connection layer（另加冗余与滚动发布余量）
（WhatsApp 真实参照：Erlang + FreeBSD 单机 ~2M connection——数量级参考，不是设计目标）
message 160B（类 WhatsApp）+ metadata ≈ 500B/条
storage = 2B × 500B × 365 ≈ 365TB/年，三副本 ≈ 1.1PB/年 → 必须按会话 sharding
媒体不计入此数，走独立 object storage 链路
typing indicator / presence 不入 storage：纯 in-flight 信号，零存储成本
```

估算的结论不是"必须用某个 database"，而是三条：消息必须按会话分片，单表写入不可行；connection layer 与 storage layer 独立扩展；媒体不能占文本消息的低延迟主链路。

## 4. High-Level Design

```
                        ┌──────────── connection layer（路由状态外置）────────────┐
client A ──MQTT/WS──▶ LB ──▶ Chat Gateway #1 ─┐
                                              ├─▶ routing service（user+device → gateway，Redis TTL）
client B ──MQTT/WS──▶ LB ──▶ Chat Gateway #2 ─┘
                                                     │
   message pipeline：                                 ▼
   client 本地 outbox → E2EE 加密 → API ──▶ message service（dedup + 分配 sequence_no）
                                  │
                                  ▼
                          Kafka（按 conversation_id partition）
                                  │
                                  ▼
                          message storage（Cassandra，按 session partition）
                                  │
                                  ▼
                          delivery service ──▶ 查 routing ──▶ target gateway ──▶ client B（device ACK）
                                                       │（offline / app killed）
                                                       ▼
                                                  push service(APNs/FCM)——只负责唤醒，不是可靠通道

real-time 信号分层（本章核心图）：
  durable     message 本体          → 走完整 pipeline（persist → deliver → device ACK）
  semi-durable delivery/read 游标   → inbox_state，reconnect 后可恢复
  ephemeral   typing / presence     → 不 persist、不保证送达，gateway 查 routing 直推 online 对端
```

一个容易被说错的点：**Gateway 不是完全 stateless 的**——TCP/WebSocket 连接就在它的内存里。真正外置的是 `user_id → gateway_id` 这份可恢复的路由状态。这样任一 Gateway 故障后，client 可以重连到其他实例，delivery service 也不依赖某台机器的私有内存。

## 5. Data Model

```sql
messages (
  conversation_id,       -- partition key
  bucket,                -- 按日期或 sequence 范围切分，避免无限大分区
  sequence_no,           -- server 分配，会话内单调递增——展示与补洞的依据，绝不依赖 client 时间戳
  message_id,            -- canonical ID（TIMEUUID 兼容）
  sender_id,
  ciphertext,            -- E2EE：存储的是密文
  type,
  created_at,
  PRIMARY KEY ((conversation_id, bucket), sequence_no)
)

user_conversation_state (           -- 每 user 的游标：reconnect 增量 sync 全靠它
  user_id,
  conversation_id,
  last_delivered_seq,               -- double tick 游标
  last_read_seq,                    -- blue tick 游标
  unread_count,
  updated_at,
  PRIMARY KEY (user_id, conversation_id)
)

message_dedup (
  sender_id,
  client_message_id,                 -- client 重试前生成，幂等键
  canonical_message_id,
  PRIMARY KEY (sender_id, client_message_id)
)
```

核心想法：

- `messages` 是每个会话的**权威消息日志（单份）**，不为收发双方各复制一份正文；
- `sequence_no` 由 server 在写入路径上分配，承载"会话内保序"这个 core 需求（partition key + sequence 的设计直接实现它）；
- `client_message_id` 实现 idempotent dedup；
- 不必死记某一种 database——Cassandra / DynamoDB / 按会话分片的关系型都可能合适，关键是写入模式、会话内顺序、按游标读取、分片扩容路径。

## 6. Deep Dives

### Deep Dive A · Real-time transport 怎么选（must-know，本章核心之一）

| 方案 | 方向 | latency | mobile 电耗 | 适用 |
|------|------|---------|------------|------|
| HTTP short polling | pull | 差（间隔）| 高（radio 反复唤醒）| 反面教材 |
| long polling | pull→伪 push | 中 | 中 | WS 不可用时的 fallback |
| SSE | server→client 单向 | 好 | 低 | 只下行（行情推送），聊天不够 |
| WebSocket | 双向 | 好 | 低 | Web 端主力 |
| MQTT（over TCP/TLS）| 双向 | 好 | 极低 | **移动端 messaging 的行业答案**（WhatsApp/Facebook Messenger 类协议、大量 IM SDK）|

**为什么移动端 IM 偏好 MQTT 而不是裸 WebSocket**——E5 的差异化得分点：

- **协议开销**：MQTT fixed header 最小 2 字节，长连接心跳 PINGREQ 只有 2 字节；WebSocket frame + 应用层 JSON 的心跳动辄上百字节。千万级 connection 下，心跳流量差两个数量级。
- **QoS 内建**：MQTT 原生 QoS 0（fire-and-forget，配 typing indicator）/ QoS 1（at-least-once + message id 去重，配 message 本体）/ QoS 2（exactly-once 四步握手，IM 一般不用——太贵）。**"ephemeral 信号走 QoS 0、durable message 走 QoS 1"一句话体现分层思维**。
- **persistent session**：MQTT broker 可以替 client 保存 subscription + 离线 message（QoS 1），client 重连后自动补投——session resume 的协议级实现。
- **keepalive 内建**：协议自带 keepalive 协商，正好承载 presence。

**真实系统参照**：WhatsApp 服务端是 Erlang/OTP + FreeBSD——BEAM 的轻量 process 模型每 connection 一个 process，内存开销 KB 级，单机撑到 ~2M connection；协议是 XMPP 演化出的私有协议（后期向 MQTT 风格收敛）。面试一句 "Erlang 的 per-connection process 模型是 connection layer 选型的底层逻辑" 价值极高。

### Deep Dive B · Connection 生命周期与 session resume

一条 connection 的完整生命：

```
1. TCP + TLS 握手 → 2. 应用层 auth（token + device_id 绑定）
→ 3. 注册 routing（user+device → gateway + connection_id，Redis，TTL = 3×heartbeat）
→ 4. 稳态：heartbeat（keepalive）+ 收发
→ 5. 断线 → jittered backoff 重连 → resume（带 last_received_seq 游标）
```

必须主动讲的四个坑：

- **heartbeat 间隔是被 NAT/防火墙逼出来的**：运营商 NAT 表项超时常见 5 分钟级，keepalive 必须小于它；但又不能太密——移动端 radio 从 idle 唤醒一次耗电可观。**生产常见 30s~2min，且前台/后台用不同间隔**（前台 30s，后台 5min 或直接断开靠 push notification）。
- **session resume 而不是全量 resync**：client 重连时带 `(conversation_id, last_received_seq)` 游标，delivery service 按 `user_conversation_state` 差额 back-fill（`GET /conversations/{id}/messages?after_sequence_no=N`）。**每次地铁换乘都全量拉历史是灾难**——设计目标是 resume 成本 O(gap) 而不是 O(history)。
- **reconnect storm**：地铁到站、机房抖动 → 百万 client 同时重连。解法组合：client 侧 random jittered backoff（打散到达时间）；gateway 侧 admission control（按连接建立速率限流，宁可慢 5 秒不能雪崩）；auth 服务优先保障 resume 请求（区分"新 session"和"resume"）。
- **routing 表清理**：gateway crash 时它的 routing entries 靠 TTL 过期消失；优雅下线时主动清除并让 client 重连。重连后的正确性不靠"连接永远不断"，而靠按游标补拉。

### Deep Dive C · Real-time delivery pipeline：三种 tick 是三条不同的路径

WhatsApp 的 UI 状态机（✓ / ✓✓ / 蓝色✓✓）背后是三条不同的 server 路径，能拆开讲就是 E5 水准：

| UI 状态 | 语义 | 确认方 | 触发路径 |
|---------|------|--------|---------|
| ✓ 单勾 | server 已接收并进入可恢复的持久化路径 | server | message service 写入 Cassandra 成功 → ack 回 sender（经 sender 的 gateway）|
| ✓✓ 双勾 | 已送达接收方设备 | **接收端设备** | delivery service 投递且 client 已收到并本地持久化（不是"推出去就算"）→ 状态更新 event 回 sender |
| 蓝色✓✓ | 已读 | 接收端设备 | 用户读到该消息 → client 上报 `last_read_seq` 游标 → server 更新 `user_conversation_state` → 通知 sender |

三个必须讲清的细节：

- **ACK 回环本身也是 real-time 推送**：双勾/蓝勾不是 sender 轮询来的，是 server 把状态变更 event 经 sender 自己的 long-lived connection 推下去。sender 断线期间 tick 更新怎么办？——上线时按发送箱视角的游标补齐，**tick 状态最终一致，不是强实时**。
- **delivered 的定义在 clarification 阶段锁死**："推到设备"（FCM/APNs 收到即算）还是"app 进程确认"（应用层 ACK）？后者更准但 app 被杀时永远到不了双勾——**这正是 push notification 路径存在的意义**：app 被杀 → FCM/APNs 投 system notification（只负责唤醒，不是可靠通道）→ 用户点开 → app 冷启动 → 走 resume 增量拉 → 补 ACK。
- **typing indicator 是 ephemeral 信号的标准案例**：不 persist、不保证送达（QoS 0）、client 侧 throttle（按 key 每几秒最多发一条）、gateway 直接查 routing fan-out 给对端——**完全绕开 message pipeline**。在白板上画出"durable 走管道、ephemeral 抄近道"的两条线，是本章最出彩的一笔。
- **presence 同理**：heartbeat 上报 → state service（Redis `user:{id}` TTL）→ 变更 event 只推给"当前打开与你的 session"的人（subscription 模型）。trade-off："state 允许 stale 30 秒，省下的是全员广播的数百万 QPS。没人会为'好友明明 online 却显示 offline'投诉。"

### Deep Dive D · 可靠投递：dedup、顺序与补洞

发送端超时后必然重试。若 server 已成功写入但确认包丢失，没有幂等键就会产生重复消息。组合拳：

- **dedup**：client 重试前 generate `client_message_id`，server 以 `(sender_id, client_message_id)` idempotent dedup——重复请求返回第一次写入的结果
- **ordering**：`sequence_no` 由 server 在写入路径分配（同一 conversation 路由到同一分区/顺序权威）；不同会话不需要全局排序。接收端若发现序号有洞：短暂等待 → 请求补拉 → 按 `sequence_no` 展示
- **offline delivery**：上线后按游标拉增量——"推拉结合：online 走 push，push 失败/offline 走 pull fallback"
- **语义表述**：至少一次投递 + idempotent 处理 + 按会话序号重排 = **用户可见的近似一次语义**——注意措辞，不要直接宣称严格的 end-to-end exactly-once

### Deep Dive E · E2EE（WhatsApp 题不能漏）

发送端使用接收端**每台设备**的公钥材料加密；server 只看到密文、会话路由信息和必要 metadata。server 需要支持公钥/预密钥分发，但不持有可解密正文的密钥。

由此带来的取舍（主动讲，是加分项）：

- server 不能做明文全文搜索、内容理解、基于正文的推荐；
- multi-device 意味着每台设备有独立密钥材料与投递状态；
- 图片/视频也应在 client 加密后上传，聊天消息只携带对象引用与解密所需信息；
- server 仍可基于频率、连接行为等 metadata 做部分反滥用，但能力受限。

密钥协商协议细节（如 Signal 的 Double Ratchet）明确列为后续深入项——重点是把 E2EE 作为架构边界画进图里。

### Deep Dive F · group chat fan-out

| 场景 | 主要策略 | 原因 |
|--------|---------|------|
| 一对一与小群（≤ 200）| write fan-out：一条 message × N 个收件人状态记录 | 成员少，在线投递与未读状态易维护，延迟低 |
| 大群（数千+）| write/read 混合：正文单份存储，成员 conversation list 写一条指针 | 全员复制正文写放大会失控 |
| 超大频道 | 单份会话日志 + 成员游标，通知驱动拉取 | "所有成员立刻收到每条消息"降级为按需同步 |

**分界不是固定的"200 人"，而是 `群成员数 × 消息频率 × 可接受投递延迟`**——这是 real-world systems（微信/微博）的成熟做法。

Real-time 信号的群内 fan-out 另算：typing indicator 在 200 人群里若全员广播就是 200 倍放大——**ephemeral 信号可以采样/聚合/干脆只在 ≤N 人的群开放**，典型的 scope 控制。

### Deep Dive G · multi-device、offline sync 与媒体

- **multi-device**：每设备独立 `device_id` + 独立 connection routing + 独立 delivery/read 游标；message 发送后投递到同一用户的所有活动设备；已读状态按产品选"任一设备已读即全局已读"或"各设备独立"；删除/撤回是 **sync event**（走 message pipeline push-down tombstone）。
- **媒体**：client 加密后向 object storage 申请 pre-signed URL 直传 → 成功后发一条携带对象引用的聊天消息 → 大文件不阻塞文本低延迟通道；CDN 分发下载。
- **offline 统一走游标**：推送通知只负责提醒/唤醒，不能当作可靠 message 通道。

## 7. Red Flags

- 用 HTTP short polling 做 core message 通道且讲不出 cost
- 讲了 WebSocket/MQTT 但答不出"server 怎么把 message 推给特定 user"（= 没有 routing 表概念）
- **没有 session resume 设计**（每次 reconnect 全量拉 → 被问"地铁通勤场景"当场卡住）
- **reconnect storm 没有应对**（million-scale 长连接系统的必考 follow-up）
- 把"server 收到请求"当作"已发送"，却不说明持久化确认点
- 把"至少一次 + 幂等"直接称为严格 exactly-once
- 用 client 时间戳保证顺序，或声称需要全局消息顺序
- message 存 MySQL 单表、没有 sharding 故事（2B/day write 入）
- 没有 dedup/ordering 讨论（被问"network retry 重发怎么办"当场卡住）
- online presence 全员广播（没算过 QPS）
- read receipts feature 没有游标设计（每次全量比对？）
- typing indicator 也要 persist / 也要 ACK（没分层，资源浪费）
- 将消息正文为每个群成员完整复制，却不讨论大群写放大
- 把 APNs / FCM 当作可靠消息存储
- 题目明确是 WhatsApp，却完全遗漏端到端加密
- 一个会话永久放在单个无界 database 分区中

## 8. One-Minute Elevator Pitch

"connection layer 用 MQTT-over-TLS 长连接（移动端比裸 WebSocket 省两个数量级的心跳开销，QoS 0/1 天然区分 ephemeral 信号和 durable message），50–100 台 gateway、routing 表外置 Redis + TTL，gateway 故障后 client 重连任意实例、按游标恢复。real-time 功能分三层：message 本体在 client 完成 E2EE 加密后走 API → Kafka 按 conversation partition → Cassandra（(conversation_id, bucket) 分区 + server 分配的 sequence_no 保证会话内可解释顺序，`(sender_id, client_message_id)` 幂等去重）；delivery/read tick 是独立的状态 event，server 经对端自己的长连接推回，断线靠游标补齐；typing/presence 是 ephemeral 信号，不 persist、gateway 查 routing 直推。delivery semantics：at-least-once + idempotent + 按序号重排 = 用户可见的近似一次。断线走 jittered backoff + session resume（增量补投，成本 O(gap)），app 被杀走 FCM/APNs 唤醒 + 冷启动增量拉。group chat 按 `成员数 × 频率 × 延迟容忍` 分界：小群 write fan-out、大群单份日志 + 游标读取。E2EE 是架构边界：server 只路由和存储密文；媒体由 client 加密后 pre-signed URL 直传对象存储，不占聊天链路。"

---

← [Q3 Top-K](03-top-k-heavy-hitters.md) ｜ [Q5 News Feed →](05-news-feed.md)
