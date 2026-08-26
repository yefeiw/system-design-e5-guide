# 04 · Estimation & Numbers to Know (Back-of-Envelope)

> estimation 的作用不是算准，是**用 numbers 框定设计空间**。E5 面试里一句「按 100M DAU、read/write ratio 100:1，read QPS peak 约 100K，single database MySQL 扛不住，所以这里必须 sharding」值 10 句形容词。

## 1. Numbers to Know（先背这张表）

### 1.1 latency numbers（Jeff Dean 经典表，2020s 更新版）

| 操作 | order of magnitude | 记忆锚点 |
|------|------|---------|
| L1 cache 命中 | ~1 ns | — |
| 互斥锁加锁 | ~15 ns | — |
| memory random read | ~100 ns | 1 KB data |
| **SSD random read** | **~150 µs** | 比 memory 慢 1000× |
| 同机房 RTT | ~0.5 ms | — |
| **跨可用区 RTT** | **~1–2 ms** | 同区域 |
| **跨区域 RTT（美东↔美西）** | **~60–80 ms** | 光速物理极限 |
| HDD 寻道 | ~10 ms | — |
| sequential reads 1 MB（SSD） | ~1 ms | — |

**面试 killer line**："跨区域一次 RTT 够 memory read 60 万次——这就是为什么多区域 synchronous replication 是最后手段。"

### 1.2 availability conversion

| 9s | 年停机 | 用法 |
|----|--------|------|
| 99% | 3.65 天 | 不能叫 HA |
| 99.9% | 8.7 小时 | 一般线上 service 底线 |
| 99.99% | 52 分钟 | 需要自动 failover |
| 99.999% | 5 分钟 | 需要消除一切人工介入 |

**killer line**："我们说 99.95%，意味着月度预算 22 分钟——每次发布耗 5 分钟的话，发布 failure 最多容忍 4 次。"

### 1.3 single-machine capacity 直觉（2020s 硬件）

| resource | single machine order of magnitude |
|------|---------|
| memory | 64–256 GB |
| SSD | 4–16 TB |
| 网卡 | 10–25 Gbps ≈ 1–3 GB/s |
| Web server concurrency connection | ~10K–65K（端口/memory 限制）|
| 单 MySQL instance 安全 write QPS | ~5K–10K（普通硬件、简单 write）|
| 单 Redis instance | ~100K ops/s |
| 单 Kafka broker | ~100K+ msg/s（小 message）|

**这些 numbers 决定「什么时候必须上 distributed」**——比如 write 入 QPS 估出来 50K，single database 必挂，你就有了 sharding/queue peak shaving 的 numbers 依据。

## 2. estimation workflow（4 步，2–3 分钟讲完）

1. **user scale** → DAU（题目给的或假设）
2. **read write QPS** → per-user 操作次数 × DAU ÷ 86400，再 × peak multiplier ×3–5
3. **storage** → 单条 record 大小 × 日增条数 × retention period（**别忘了 index 和 replica 的放大系数，通常 ×2–3**）
4. **bandwidth** → QPS × average request/response 大小

### 完整示例：URL shortener（题目：100M DAU，per-user 5 次 read 0.1 次 write）

```
read QPS = 100M × 5 / 86400 ≈ 5,800 QPS，peak ×3 ≈ 17K QPS
write QPS = 100M × 0.1 / 86400 ≈ 115 QPS，peak ≈ 350 QPS
storage = 500 bytes/条 × 4B 条/年 ≈ 2 TB/年（含 index ×2 ≈ 4 TB）
bandwidth = 17K × 1 KB ≈ 17 MB/s read 出 —— 很轻
takeaway → read 需要 cache（17K QPS single database + 每次回源扛不住），write 完全无压力，
         storage 单 sharding 扛得住 → 这是"read 重 write 轻"的教科书 scenario
```

注意最后一句：**estimation 必须落到设计 takeaway**。算完不说话等于白算。

## 3. E5 级别的 estimation bonus signal

### 3.1 主动声明假设的 error margin

> "DAU 我假设 100M，哪怕差一个 order of magnitude 到 10M，takeaway 不变——因为 bottleneck 在 cache 而不是 database 行数。"（impression estimation 的稳健性思维）

### 3.2 算「钱」

> "storage 4 TB/年，S3 是 $0.023/GB/月，一年 storage cost 不到 $1,200——cost 不是约束。但如果有图片，object storage + CDN 才是大头。"

### 3.3 算「人」

大厂 interviewer 喜欢听 capacity planning 落到 operations：
> "17K QPS peak，单 instance 扛 3K，需要 6 台 + N+2 redundancy ≈ 8 台，一个 ASG 就够了，不需要多区域。"

## 4. 常见坑

| 坑 | 修正 |
|----|------|
| 忘 × peak multiplier（用 average QPS 定 capacity） | average ×3–5 才是 capacity target |
| 忘 replica 和 index 放大 | storage 直接 ×2–3 |
| 单位错乱（GB/Gbps/GB/s 混用） | whiteboard 顶上 write checklist 位再动笔 |
| 估完不落地 | 每个 numbers 后面跟一句"所以……" |
| exact 到小数 | keep 1–2 significant digits，impression 的是 order of magnitude 思维 |

## 5. practice checklist（每天 10 分钟，练两周）

给这些 scenario 口算 QPS / storage / bandwidth，限时 3 分钟：

1. Twitter Feed：200M DAU，per-user 刷 20 次
2. chat system：50M DAU，per-user 40 条 message，message retain 1 年
3. videos 站：100M DAU，per-user 30 分钟，average bitrate 2 Mbps
4. log system：10K server × 每台 100 logs/s × 500 B/log
5. ad click：1B impressions/day，CTR 1%，click record 200 B

答案自己算，重点是把「numbers → 设计 takeaway」的最后一句话练 cost 能。

## Next Module

→ [05 · Building Blocks](05-building-blocks.md)
