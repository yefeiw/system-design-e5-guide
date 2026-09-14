# 05 · Building Blocks

> 每个 building block 按两层组织：**E4 层**（知道是什么）一句话带过；**E5 层**（selection 理由 + trade-off + 真实的坑）是你要练到脱口而出的部分。

## 1. Load Balancer

**E4 层**：L4（TCP）按 connection 分发，L7（HTTP）按 request 分发，algorithm 有 polling/最少 connection/consistent hashing。

**E5 层**：
- **stateful vs stateless**：consistent hashing 解决「同一 user 落到同一 node」——用在 connection 粘滞、cache sharding；cost 是 node 增删时的 rehash jitter（virtual nodes mitigate）
- **健康检查与摘除**：多久探测一次、几次失败摘除、**摘除后 traffic 去哪**（雪崩 risk）
- **global load balancing**：GeoDNS / Anycast 把 user 引到最近区域；「区域挂了 TTL 内切不走」是经典坑
- 现实中通常双层：云 LB（ALB/Envoy）+ client 侧 load balancing（service mesh）

**killer line**："LB 本身要 stateless + 健康检查，否则它自己成了 SPOF——我们用 N+1 个 LB + VRRP/Anycast。"

## 2. Cache

**E4 层**：local cache vs distributed cache（Redis/Memcached），cache-aside 模式。

**E5 层**——按四个问题组织：

### 2.1 放什么模式
| 模式 | 适用 | 坑 |
|------|------|-----|
| Cache-Aside | default 选择 | invalidation window；首次 loading penetration |
| Write-Through | read-heavy、能接受 write latency | write path 变慢；cold data 占 memory |
| Write-Behind | write QPS 高 | **丢 data window**——billing 类不可用 |
| logical expiration | hot key | return stale data 要 business acceptable |

### 2.2 invalidation 怎么办
- **cache penetration**（查不存在的 key）：Bloom filter / cache null-value
- **cache avalanche**（同时 invalidation）：TTL 加 random jitter / logical expiration / multi-tier cache
- **cache stampede**（hot key 过期瞬间）：singleflight（merge 回源）/ 互斥锁回源

### 2.3 consistency 多强
- 「先更新 DB 再删 cache」是主流答案，但**仍有 window**——讲得出这个 window 及「latency 双删/subscription binlog（CDC）」的补法，就是 E5 level
- 追问终极版：「cache 和 DB 永远 strongly consistent 做不到，只能 convergence——你的 business 能接受多长的 convergence window？」

### 2.4 hit rate 决定一切
> "我会在 monitoring 里盯 hit rate。低于 90% 说明 working set 大于 memory 或者 key 设计有问题——这时加 machine 不如改 shard key。"

## 3. database 与 storage selection

**E4 层**：SQL 保一致，NoSQL 好 scaling。

**E5 层**——selection 必须落在「read/write 模式 + consistency 需求 + query 模式」三个轴上：

| storage | 什么时 candidates | 代表 | 经典坑 |
|------|-----------|------|--------|
| 关系型（PostgreSQL/MySQL） | 复杂 query、transaction、strongly consistent default | RDS, Spanner | single machine write 上限 → sharding cost 大 |
| KV（DynamoDB/Cassandra） | 海量 write、key query、eventually consistent acceptable | Dynamo | 无法二级 query；partition key 设计错=灾难 |
| 文档（MongoDB） | schema 灵活、嵌套 read | Mongo | transaction 弱；index 膨胀 |
| wide-column（Cassandra） | time-series、log、write-heavy | Cassandra | LWT 性能差；read amplification |
| 全文 search（Elasticsearch） | search、前缀、aggregation | ES | 近 real-time≠real-time；JVM operations 重 |
| time-series（TimescaleDB/ClickHouse） | monitoring 指标 | ClickHouse | 更新难 |
| object storage（S3） | 大文件、immutable | S3 | 不是 database；列表操作贵 |

**selection script**：
> "write QPS 20K、按 user_id point lookups、eventually consistent acceptable → Cassandra/DynamoDB 合适。如果团队只有 MySQL，分 64 个逻辑库也能扛，cost 是跨行 transaction 没了——这里我选 X 因为……"

### Sharding
- **shard key 选择**是 core 考题：高 cardinality、query 路径覆盖、避免 hot spot（user_id 好于 auto_increment id）
- scope sharding（好查 scope、易 skew）vs hash sharding（均匀、range queries 废）——**没有双赢，只有取舍**
- 扩容路径：consistent hashing / 预分足够多的逻辑片（64→1024）物理迁移
- 跨片 transaction： Saga / two-phase commit / 尽量设计成单片 transaction（**能用单片解决就别跨片**）

## 4. replication 与 consistency

**E4 层**：主从 replication、CAP 定理。

**E5 层**：
- **replication 是 availability 手段，sharding 是 capacity 手段**——别混着讲
- synchronous replication（不丢 data、write 慢）vs async（快、failure 切换丢 data）vs 半 sync（折中）——MySQL `semi-sync`、PostgreSQL `synchronous_commit` 就是这个谱系
- **read-your-writes（read-your-writes）**：user posting 后立刻 refresh 看不到自己的帖子 → session 粘滞到主库 / read timestamp routing。**社交类题 must-know**
- CAP 讲人话版："network partition 必然发生，所以你真正选的是 partition 时 keeps C 还是 keeps A。ticketing system keeps C denial of service；Feed 流 keeps A return 旧 data。"
- consistency 谱系：线性一致 > ordering 一致 > 因果一致 > eventually consistent——**能说清 business 落在哪一档及为什么**

## 5. Message Queue

**E4 层**：Kafka decouple peak shaving。

**E5 层**：
- **delivery semantics 三档**：at-most-once（丢可以）/ at-least-once（不丢但重，**default 档**）/ exactly-once（贵，依赖 idempotent 或 transaction）
- **idempotent 是 at-least-once 的解药**：dedup key + dedup window / database 唯一约束 / 版本号。讲清「consumer 重启后 replay 导致重复扣款怎么防」= E5 signal
- Kafka core 概念：partition 是并行单位、offset 提交 semantics、consumer group rebalancing jitter、**partition count = 并行度上限，提前规划**
- ordering 保证只在**单 partition**内成立——按 user_id partition，同 user 的 message 才 ordered
- peak shaving script："write 50K QPS traffic surge，DB 只能 10K——queue 做缓冲，消费端按能力 pull，cost 是 latency 从 ms 级涨到 second-level，这个 latency business acceptable。"

## 6. CDN & Edge

**E4 层**：静态 resource 放 CDN。

**E5 层**：
- 适合：immutable 内容（图片、videos、JS bundle）；**不适合**：个性化动态内容（API response）
- 中间态：edge cache + 短 TTL（second-level）——"个性化做的只有头部的 user 名，那部分 client 拼"
- invalidation 难题：CDN refresh 是 eventually consistent 的——"改了 config 什么时候全网民生效？"是真实 incident 高发区
- videos：HLS/DASH 切片天然适合 CDN；transcoding pipeline（上传 → raw storage → transcoding queue → 多 bitrate → CDN）

## 7. 其他 high-frequency 小件（一句话 + deep-dive hooks）

- **API Gateway**：鉴权、rate limiting、routing 统一入口——"把横切 follow 点从 service 里拎出来"
- **service 发现**：K8s DNS / Consul——"instance 动态上下线，call 方怎么知道地址"
- **distributed lock**：Redis `SET NX PX` + fencing token（**为什么需要 fencing token** 是经典 deep dive 题）；或 ZooKeeper/etcd（更稳更重）
- **heartbeat 与 failure detection**：timeout 阈值与误判的 trade-off；split-brain 与 quorum
- **Bloom filter**：存在性 approximate 判断，无假阴性有假阳性——"告诉 cache penetration 说不存在的 key"
- **consistent hashing**：见 §1，cache/sharding 通用

## 8. 把 building blocks 串成 script

面试中每个 building blocks 出现时，标准三段式：

> "这里我加一层 Redis 做 cache-aside（**是什么**）。因为 read 17K QPS single database 扛不住（**为什么**）。risk 是 invalidation storm，我用 TTL jitter + logical expiration fallback；hit rate 我会作为 core SLI monitoring（**what if it breaks**）。"

## Next Module

→ [06 · Classic Problems Overview](06-classic-questions-overview.md)
