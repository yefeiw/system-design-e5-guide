# 07 · 真实系统案例研究（Case Studies）

> 真题练的是「设计过程」，案例研究补的是「现实约束」。面试里一句"Netflix 实际上是这么做的，但他们的问题是 X"能让面试官立刻把你和背题的人区分开。
> 本章按「读什么 + 记什么 + 面试怎么用」组织。

## 1. 为什么要读真实系统

三个 mock 里用得上的理由：
1. **trade-off 的实锤**：你说"我们选最终一致"，真实系统给你"Netflix 的播放许可证就是这么丢的"级别的证据
2. **规模的参照系**：知道 Instagram 用 12 台 Postgres 撑过 1000 万用户，你就不会再对 1M DAU 的题疯狂分库分表
3. **深挖的弹药**：面试官问"这个方案现实中可行吗"，你有第一手故事

## 2. 必读案例（按面试价值排序）

### 2.1 Netflix（流媒体 + 弹性云的标杆）

- **读什么**：Netflix TechBlog 的 playback architecture、chaos engineering 系列、Evans & OMS（open connect CDN）
- **记住**：
  - 全链路**最终一致**，可用性 > 一致性：服务降级时显示旧推荐、旧许可证也没关系，**播放永远不中断**是北极星
  - 每个服务配 fallback（静态响应）——"优雅降级"的工程实现
  - Chaos Monkey 主动杀实例验证冗余——运维成熟度的天花板展示
- **面试用法**：任何"可用性 vs 一致性"的讨论，"Netflix 把播放中断当最大 SLA 违约，所以全系统最终一致 + 降级"一句话顶三段论证

### 2.2 Uber（地理位置 + 实时匹配）

- **读什么**：Uber Engineering 的 H3 六边形网格、schemaless 存储、marketplace 匹配
- **记住**：
  - GeoHash 的精度/边界问题 → H3 六边形：等面积、邻居等距、无边界拼接缝
  - 写多读多的位置流（司机每 4 秒上报）→ 自研 Schemaless（ atop MySQL 的 KV 分片）
  - 匹配的 CAP 取舍：供需不平衡时匹配质量下降但不拒绝服务
- **面试用法**：地理位置题（打车、外卖、附近的人）的降维打击；H3 的"为什么六边形不是四边形"是最漂亮的 trade-off 微型案例

### 3.3 Instagram（小团队扛大流量）

- **读什么**：12 台机器撑 1000 万用户的经典博子、Instagram Engineering 博客
- **记住**：
  - Postgres 分片 + 应用层路由（shard by user_id）；用 Postgres 的特性玩花样（计数用 Redis INCR，photo id 按 photo_id 分片按 user_id 查询的二级映射）
  - **反模式警示**：他们 2012 年的技术选型（Cassandra 当时太新）证明"无聊技术 + 好分片"足够远
- **面试用法**："scale 匹配阶段"的最强论据——over-engineering 的反面教材集

### 2.4 Meta（社交图谱 + 缓存哲学）

- **读什么**：TAO（FB 的分布式图存储）、Facebook 的 cache 层论文、mcrouter/memcache 架构
- **记住**：
  - TAO：社交图读是图遍历（一对多的 fan-out 读），通用 DB 做不了 → 图专用 cache 层，命中率 95%+
  - memcache + mcrouter 的**区域化缓存**（regional pools，跨 DC 读本地副本、写走主区域）
  - look-aside cache + lease 机制解决过期竞态
- **面试用法**：Feed/社交图/缓存深挖三合一的弹药库；TAO 论文 20 分钟读完，性价比极高

### 2.5 WhatsApp（极致精简）

- **读什么**："1M connections per server" 博文（Erlang, FreeBSD）
- **记住**：50 个工程师 9 亿用户；每连接内存压到 KB 级；Erlang actor 模型天然映射长连接会话
- **面试用法**：聊天系统题的容量论据（"单机 1M 长连接是被 WhatsApp 证明过的，所以 10M 连接 10 台起步"）

### 2.6 经典论文（按性价比排序）

| 论文 | 核心思想 | 面试覆盖 |
|------|---------|---------|
| **Amazon Dynamo** | 最终一致 KV、一致性哈希、vector clock、 hinted handoff | 所有存储题的母体 |
| **Google Bigtable** | LSM、宽列、分区 tablets、chubby | Cassandra/HBase 的原理 |
| **Google MapReduce / GFS** | 批处理与分布式文件系统原型 | 数据管道题 |
| **Kafka 论文** | 日志即消息系统 | Q6 的原文出处 |
| **Spanner** | TrueTime、外部一致性、全球分布事务 | 强一致多区域题 |
| **Raft** | 可理解的一致性共识 | 被问"选主怎么实现"时 |

> DDIA（Designing Data-Intensive Applications）第 5–6 章几乎覆盖上面全部思想，时间紧就只读这两章 + 第 9 章（一致性）。

## 3. 案例怎么"用进"面试

**错误用法**（背书式）："Netflix 的博客说过……"——像背的。

**正确用法**（论证式）：

> "可用性优先在流媒体是被验证过的策略——Netflix 全系统最终一致，因为播放中断不可逆而许可证晚到 10 秒无感。我们的场景同样是'读旧数据无感、拒绝服务致命'，所以我做同样取舍。"

公式：**他们的约束 → 他们的选择 → 我的约束像不像 → 我的选择**。

## 4. 15 分钟/天的案例习惯

1. High Scalability 或 engineering blog 挑一篇
2. 读的时候只回答三个问题：规模多大？最难的技术约束是什么？他们牺牲了什么？
3. 一行笔记存进 repo：`cases.md`（自己的案例库，mock 前翻自己的笔记比翻原文快 10 倍）

## 下一章

→ [08 · 公司风格指南](08-company-guides.md)
