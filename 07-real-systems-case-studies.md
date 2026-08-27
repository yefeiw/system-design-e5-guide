# 07 · Real-World System Case Studies

> classic problems 练的是「设计过程」，case studies 补的是「现实约束」。面试里一句"Netflix 实际上是这么做的，但他们的问题是 X"能让 interviewer 立刻把你和背题的人区 separate。
> 本章按「read vs write + 记什么 + 面试怎么用」组织。

## 1. 为什么要 read real-world systems

三个 mock 里用得上的理由：
1. **trade-off 的实锤**：你说"我们选 eventually consistent"，real-world systems 给你"Netflix 的播放许可证就是这么丢的"级别的证据
2. **scale 的参照系**：知道 Instagram 用 12 台 Postgres 撑过 1000 万 user，你就不会再对 1M DAU 的题疯狂 sharding
3. **deep dive 的弹药**：interviewer 问"这个 approach 现实中可行吗"，你有第一手故事

## 2. must-read cases（按面试价值 ranking）

### 2.1 Netflix（streaming media + elastic cloud 的标杆）

- **read vs write**：Netflix TechBlog 的 playback architecture、chaos engineering 系列、Evans & OMS（open connect CDN）
- **记住**：
  - 全链路**eventually consistent**，availability > consistency：service degradation 时显示旧推荐、旧许可证也没关系，**播放永远不中断**是北极星
  - 每个 service 配 fallback（静态 response）——"graceful degradation"的工程实现
  - Chaos Monkey 主动杀 instance 验证 redundancy——operational maturity 的天花板 impression
- **interview usage**：任何"availability vs consistency"的讨论，"Netflix 把播放中断当最大 SLA 违约，所以全 system eventually consistent + degradation"一句话顶三段论证

### 2.2 Uber（geolocation + real-time matching）

- **read vs write**：Uber Engineering 的 H3 六边形网格、schemaless storage、marketplace matching
- **记住**：
  - GeoHash 的精度/boundary 问题 → H3 六边形：等面积、邻居等距、无 boundary 拼接缝
  - write-heavy read-heavy 的位置流（司机每 4 秒上报）→ 自研 Schemaless（atop MySQL 的 KV sharding）
  - matching 的 CAP 取舍：供需不平衡时 matching 质量下降但不 denial of service
- **interview usage**：geolocation 题（打车、外卖、附近的人）的降维打击；H3 的"为什么六边形不是四边形"是最漂亮的 trade-off 微型案例

### 3.3 Instagram（小团队扛大 traffic）

- **read vs write**：12 台 machine 撑 1000 万 user 的经典博子、Instagram Engineering blogs
- **记住**：
  - Postgres sharding + 应用层 routing（shard by user_id）；用 Postgres 的特性玩花样（counter 用 Redis INCR，photo id 按 photo_id sharding 按 user_id query 的二级映射）
  - **反模式警示**：他们 2012 年的技术 selection（Cassandra 当时太新）证明"无聊技术 + 好 sharding"足够远
- **interview usage**："scale matching 阶段"的最强论据——over-engineering 的反面教材集

### 2.4 Meta（social graph + cache 哲学）

- **read vs write**：TAO（FB 的 distributed 图 storage）、Facebook 的 cache 层 papers、mcrouter/memcache architecture
- **记住**：
  - TAO：社交图 read 是图遍历（一对多的 fan-out read），通用 DB 做不了 → 图专用 cache 层，hit rate 95%+
  - memcache + mcrouter 的**区域化 cache**（regional pools，跨 DC read local replica、write 走主区域）
  - look-aside cache + lease 机制解决过期竞态
- **interview usage**：Feed/社交图/cache deep dive 三合一的弹药库；TAO papers 20 分钟 read 完，性价比极高

### 2.5 WhatsApp（radical simplicity）

- **read vs write**："1M connections per server" 博文（Erlang, FreeBSD）
- **记住**：50 个工程师 9 亿 user；每 connection memory 压到 KB 级；Erlang actor model 天然映射 long-lived connection session
- **interview usage**：chat system 题的 capacity 论据（"single machine 1M long-lived connection 是被 WhatsApp 证明过的，所以 10M connection 10 台起步"）

### 2.6 classic papers（按性价比 ranking）

| papers | core 思想 | 面试覆盖 |
|------|---------|---------|
| **Amazon Dynamo** | eventually consistent KV、consistent hashing、vector clock、 hinted handoff | 所有 storage 题的母体 |
| **Google Bigtable** | LSM、wide-column、partition tablets、chubby | Cassandra/HBase 的原理 |
| **Google MapReduce / GFS** | batch processing 与 distributed 文件 system 原型 | data pipeline 题 |
| **Kafka papers** | log 即 message system | Q6 的原文出处 |
| **Spanner** | TrueTime、外部 consistency、全球分布 transaction | strongly consistent 多区域题 |
| **Raft** | 可理解的 consistency consensus | 被问"leader election 怎么实现"时 |

> DDIA（Designing Data-Intensive Applications）第 5–6 章几乎覆盖上面全部思想，时间紧就只 read 这两章 + 第 9 章（consistency）。

## 3. 案例怎么"用进"面试

**错误用法**（背书式）："Netflix 的 blogs 说过……"——像背的。

**正确用法**（论证式）：

> "availability 优先在 streaming media 是被验证过的策略——Netflix 全 system eventually consistent，因为播放中断不可逆而许可证 late 10 秒 imperceptible。我们的 scenario 同样是'read 旧 data imperceptible、denial of service fatal'，所以我做同样取舍。"

公式：**他们的约束 → 他们的选择 → 我的约束像不像 → 我的选择**。

## 4. 15 分钟/天的案例习惯

1. High Scalability 或 engineering blog 挑一篇
2. read 的时候只回答三个问题：scale 多大？最难的技术约束是什么？他们牺牲了什么？
3. 一行笔记存进 repo：`cases.md`（自己的案例库，mock 前翻自己的笔记比翻原文快 10 倍）

## Next Module

→ [08 · Company Style Guide](08-company-guides.md)
