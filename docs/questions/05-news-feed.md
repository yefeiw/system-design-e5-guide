# Q5 · News Feed（信息流）

> 难度：Medium ｜ 母题：Fan-out 写放大 + 读多写少
> Meta 系最爱。**Push vs Pull 的混合方案**是主轴，一致性细节是 E5 区分度。

## 1. 题目原文

"设计 Facebook/Twitter 的 News Feed：用户发帖、关注别人、刷到自己关注者的时间线 Feed。"

## 2. 澄清问题清单

| 问题 | 意图 |
|------|------|
| Feed 排序：时间序还是算法排序？ | 决定要不要排序服务（本题先按时间序，排序提一句）|
| 关注上限多少？（关注 300 人 vs 100 万人）| **Push/Pull 的分水岭** |
| 粉丝结构里有没有大 V（百万人关注）？| 同上，热点问题的根源 |
| 新 Feed 的刷新语义？ | 读己之写 / 最终一致的选择题 |
| 要不要支持"推文浏览数"这类轻互动？ | 写入风暴变体 |

## 3. 估算演示

```
100M DAU；人均关注 300 人、日发 2 帖、刷 20 次
写：200M posts/day ≈ 2.3K posts/s，峰值 10K/s
读：2B feed 请求/day ≈ 23K QPS 均值，峰值 100K+ QPS   ← 读是写的 20 倍
写扩散代价：一篇帖 × 关注者均值 300 = 平均 600K feed-item 写/s（峰值 3M/s）
结论 → 写扩散的写 QPS 是发帖 QPS 的 ~300 倍，这就是本题全部矛盾的来源
```

## 4. 高层设计

```
发帖：客户端 ──▶ Post API ──▶ 帖子服务 ──▶ 帖子库（Post Store, Cassandra/分片）
                                        │
                                        ▼
                              Kafka（fanout 事件）
                                        │
                              ┌─────────┴─────────┐
                              ▼                   ▼
                     Fanout Worker（push 路线）   关注图谱服务（Graph）
                       给每个粉丝的 feed cache
                       追加 post_id                    │
                              │                        │
                              ▼                        ▼
刷 Feed：Feed API ──▶ 用户 Feed Cache（Redis List，推的时间线）
                       │（缓存 miss / 混合路线）
                       └──▶ Feed 现场组装（pull）：查关注列表 → 批量拉对方近期帖子 → 归并
```

## 5. 数据模型

```
posts (post_id PK, user_id, content, media_refs, created_at)   -- 分片：post_id
follows (follower_id, followee_id, created_at)  -- 边表，分片：follower_id
            （倒排表 followees_of:{user} 用于 pull；正排 followers_of:{user} 用于 push）
feed_cache: Redis List per user（存 post_id，长度封顶如 800）
inbox (user_id, last_read_post_id)  -- 游标
```

## 6. 深挖

### 深挖 A · Push vs Pull vs 混合（主菜）

| | Push（写扩散）| Pull（读扩散）|
|---|---|---|
| 读延迟 | 极低（缓存现成）| 高（现场聚合 300 人的帖子）|
| 写放大 | 巨大（×平均粉丝数）| 无 |
| 存储 | 每人一份 feed 副本 | 无副本 |
| 新帖可见性 | 异步延迟（秒级）| 立即 |
| 大 V | **灾难**（一条帖写 100 万次）| 天然免疫 |

E5 标准答案——**混合**：
- 普通用户（粉丝 < 阈值如 10K）：push 预算进粉丝缓存
- 大 V：不扩散，**拉取时现场合并**（feed API 返回 = 缓存里的普通帖 + 在线拉取的大 V 帖，按时间归并）
- "阈值是可调参数，按 fanout worker 的消费延迟监控调——**把系统行为和运维指标挂钩**才是完整设计。"

### 深挖 B · 一致性：发完帖刷新看不到（Meta 爱问）

场景：用户 A 发帖 → 消息还在 Kafka → A 立刻刷新 Feed。
- 方案 1：**read-your-own-writes**——Feed API 检查 `inbox.last_read`，若用户自己有未进缓存的最新帖，从帖子库直接拼上
- 方案 2：发帖 API 同步写自己的 feed cache（自己必然是自己的粉丝……不是），至少同步写自己的"主页时间线"
- 话术："对外人，最终一致秒级收敛完全可接受；对作者本人，人类对'我发的东西立刻可见'零容忍——**一致性预算按用户关系分配**，这是产品的心理学，不只是技术。"

### 深挖 C · Feed Cache 细节

- 结构：Redis List of post_id，LPUSH 新帖 + LTRIM 封顶（800 条，翻页深处回源帖子库）
- 翻页游标：`(last_post_id, offset)` 双参数——"纯 offset 在有新帖写入时会跳条/重复，cursor 必须锚定 post_id"
- 缓存未命中（冷用户）：现场 pull 组装 + 回填
- 热点用户 Feed 缓存分片：按 user_id 一致性哈希

### 深挖 D · 删帖与编辑的传播

Push 模型的隐藏债务：**删帖要从所有粉丝的缓存里抠掉**。
- feed cache 存 post_id 而不是内容 → 删帖只需在帖子库打 tombstone，读侧渲染时过滤失效帖（惰性删除）
- "存 ID 不存内容，把内容收敛到单一来源（Single Source of Truth），缓存才敢大"——这是 push 模型能活下来的前提

### 深挖 E · 排序（一句话级别）

"真实产品是算法排序（互动预测分），那会把 feed 组装变成**候选检索 + 打分**两段式（召回几百条 → 轻量模型打分取几十条），架构主体不变，加一个排序服务。今天我先做时间轴。"

## 7. 红牌答案

- 纯 push 且没讨论大 V 写放大
- 纯 pull 且没算 300 人现场聚合的读延迟
- feed cache 存帖子全文（删帖/编辑灾难）
- read-your-own-writes 没意识（被面试官问"你自己发的刷不到"才反应）
- 分页用纯 offset

## 8. 一分钟电梯版

"读是写的 20 倍，核心矛盾是 fan-out 写放大。混合方案：普通用户 push——帖子经 Kafka 进 fanout worker 写进各粉丝的 Redis feed 缓存（存 post_id 不存内容，删帖走 tombstone 惰性过滤）；大 V 不扩散，读时现场拉取归并，阈值按 fanout 消费延迟调。一致性分层：对外人最终一致，对作者本人 read-your-own-writes 同步拼入。分页用 post_id 锚定的 cursor。算法排序是把组装层换成召回+打分两段式，架构骨架不变。"

---
← [Q4 聊天系统](04-chat-system.md) ｜ [Q6 消息队列 →](06-distributed-message-queue.md)
