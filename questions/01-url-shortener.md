# Q1 · URL Shortener

> Difficulty: Easy ｜ archetype：read-heavy + cache
> 看似简单，其实是**把「ID generate」「cache」「redirect 链路」三个 deep-dive spots 打穿**的最佳练手题。

## 1. Problem Statement

"设计一个类似 bit.ly 的 URL shortener。user 提交长 URL，得到一个 short URL 接，访问 short URL 接跳转到原 URL。"

## 2. Clarifying Questions

| 问题 | 为什么问 |
|------|---------|
| short URL 需要自 definitions 别名吗？（`bit.ly/my-wedding`）| 影响 ID generate 的整个设计 |
| 链接过期吗？永久有效？ | storage estimation 和 cleanup strategy |
| 需要 click 统计吗（分析 feature）？ | **这题最大的隐藏需求**——决定要不要 write 入 click log pipeline |
| DAU / read write 字数级？ | 决定 cache 层必要性 |
| read latency requirement？（301 vs 302 的意义）| impression HTTP 细节的机会 |

**scope convergence script**："我聚焦创建/read 取/redirect 三个 core feature + click counter；自 definitions 域名和过期管理放到 follow-up。"

## 3. Estimation Walkthrough（假设 100M DAU，per-user 5 read 0.1 write）

```
read QPS ≈ 5,800（peak ~17K） write QPS ≈ 115
storage ≈ 500B × 4B 条/年 ≈ 2TB/年（含 index ×2 = 4TB）
bandwidth ≈ 17K × 1KB ≈ 17MB/s（轻）
takeaway → read path 必须 cache；write 和 storage 毫无压力 → 单 write-heavy read 教科书 scenario
```

## 4. High-Level Design

```
client ──POST /url──▶ LB ──▶ URL shortener（stateless）──▶ ID generate 器
                                   │
                                   ▼
                                MySQL（主从）
client ──GET /abc123──▶ LB ──▶ URL shortener ──▶ Redis cache ──(miss)──▶ MySQL
                                   │
                                   ▼
                              return 301/302 + 原 URL
（click event ──▶ Kafka ──▶ 分析 pipeline，若题目要统计）
```

data flow 口述版："创建走 service tier 拿唯一 ID、编码成短码、write 主库；访问走 cache 优先，miss 回源并回填。"

## 5. Data Model

```sql
-- MySQL 即可（strongly consistent + transaction + 单表 index 简单）
short_url (id BIGINT PK, -- global auto-increment 或 Snowflake ID
           code VARCHAR(7) UNIQUE,-- base62 编码
           original_url TEXT,
           created_by BIGINT,
           expires_at TIMESTAMP NULL,
           created_at TIMESTAMP)
INDEX (created_by), INDEX (expires_at)
```

**为什么 SQL 不是 DynamoDB**："write 只有百级 QPS，关系型毫无压力，且过期 cleanup、按 user query 都要灵活 index——引入 KV 反而丢失这些。scale 撑死 single database + read write 分离。"

## 6. Deep Dives

### Deep Dive A · ID 怎么 generate（本题经典）

| approach | 优点 | 缺点 / E5 必须说出来 |
|------|------|---------------------|
| DB auto-increment ID | 简单、无冲突 | SPOF；暴露总数（爬虫可枚举你的量）；分库后要步长分段 |
| UUID | 无协调 | 128 位太长、无序（B+ 树页分裂）→ 短码 scenario 直接排除 |
| **号段模式（Leaf/美团）** | DB 只发段，扛高 write | 号段 service 要 HA；段内 monotonic |
| Snowflake Snowflake | 趋势递增、去中心化 | **clock 回拨**问题；machine 位要规划 |

E5 standard answers："write QPS 才 115，我选 DB auto-increment + 号段 cache 就够——为一个低频 write 引入 Snowflake 的时间位纯属 over-engineer。如果题改成十亿级 write，我会换 Snowflake 并处理 clock 回拨（等待/报错/备份位）。"

### Deep Dive B · Base62 与碰撞

- 62^7 ≈ 3.5 万亿，7 位编码足够
- auto-increment ID → base62 **天然无碰撞**（这是选 auto-increment 的隐藏红利，说出来是加分）
- 若 hash 取前 7 位 → 必须处理碰撞（查库 retry）→ 讲得出"所以我不选 hash"是 trade-off 表达

### Deep Dive C · 301 vs 302 与 cache

> "301 永久 redirect 会被浏览器 cache——**后续 click 不再到我们 server，click 统计就废了**。要统计就 302 临时 redirect，cost 是每次都回源，read 压力全在我们这。所以这个选择取决于 business 更在乎统计准确还是 response latency。"

这一段是本题最 E5 的 30 秒。

### Deep Dive D · hot spot

"热门 short URL（病毒传播）单 key 会被打到单个 Redis sharding → local cache（in-process LRU）做第一层 + logical expiration 防 stampede。"

## 7. Red Flags

- 上来就"sharding 128 个库"（write 115 QPS 分什么）
- 用 hash + 不谈碰撞处理
- 只说"加 Redis"，不谈 hit rate/invalidation 策略/hot spot
- 301/302 的统计问题没 awareness（被 interviewer 点破后才反应）

## 8. One-Minute Elevator Pitch

"read/write ratio 1000:1，write 走 DB auto-increment + base62 编码天然无碰撞；read 走 Redis cache-aside + in-process 二级 cache 扛 hot spot；301 vs 302 取决于要不要 click 统计——要统计就得 302 回源，那就把 cache hit rate 做成 core SLI。click log async 进 Kafka 做分析。storage order of magnitude 2TB/年，single database 主从足够，不为这个 scale 引入任何 distributed 组件。"

---
← [06 classic problems overview](../06-classic-questions-overview.md) ｜ [Q2 rate limiter →](02-rate-limiter.md)
