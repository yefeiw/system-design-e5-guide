# 05 · Building Blocks

> Each building block is organized in two layers: the **E4 layer** (knowing what it is) gets one sentence; the **E5 layer** (why you'd select it + trade-offs + the real pitfalls) is the part you need to drill until it rolls off your tongue.

## 1. Load Balancer

**E4 layer**: L4 (TCP) distributes by connection, L7 (HTTP) distributes by request; algorithms include round-robin / least-connections / consistent hashing.

**E5 layer**:
- **stateful vs stateless**: consistent hashing solves "the same user lands on the same node" — used for connection stickiness and cache sharding; the cost is rehash jitter when nodes are added or removed (virtual nodes mitigate it)
- **Health checks and ejection**: how often to probe, how many failures trigger ejection, and **where the traffic goes after ejection** (avalanche risk)
- **global load balancing**: GeoDNS / Anycast steer users to the nearest region; "the region is down but we can't fail away within the TTL" is the classic pitfall
- In practice it's usually two layers: the cloud LB (ALB/Envoy) plus client-side load balancing (service mesh)

**killer line**: "The LB itself has to be stateless and health-checked, otherwise it becomes the SPOF — so we run N+1 LBs with VRRP/Anycast."

## 2. Cache

**E4 layer**: local cache vs distributed cache (Redis/Memcached), the cache-aside pattern.

**E5 layer** — organized around four questions:

### 2.1 Which Caching Pattern
| Pattern | When it fits | Pitfall |
|------|------|-----|
| Cache-Aside | the default choice | invalidation window; penetration on the first load |
| Write-Through | read-heavy, and you can tolerate write latency | slower write path; cold data occupies memory |
| Write-Behind | high write QPS | **data loss window** — unusable for billing-type workloads |
| logical expiration | hot keys | returning stale data has to be acceptable to the business |

### 2.2 What to Do About invalidation
- **cache penetration** (queries for keys that don't exist): Bloom filter / cache null-value
- **cache avalanche** (many keys invalidated at once): TTL with random jitter / logical expiration / multi-tier cache
- **cache stampede** (the instant a hot key expires): singleflight (merge the origin reads) / mutual exclusion lock around the origin read

### 2.3 How Strong Does consistency Need to Be
- "Update the DB first, then delete the cache" is the mainstream answer, but **there is still a window** — being able to describe that window and the fixes (delayed double delete / subscribing to the binlog (CDC)) is E5 level
- The follow-up endgame: "Cache and DB can never be strongly consistent, only convergent — how long a convergence window can your business tolerate?"

### 2.4 hit rate Decides Everything
> "I'd watch hit rate in monitoring. Below 90% means the working set is bigger than memory or the key design is wrong — and in that case adding machines helps less than fixing the shard key."

## 3. database and storage selection

**E4 layer**: SQL for consistency, NoSQL for scaling.

**E5 layer** — selection has to land on three axes: "read/write pattern + consistency requirements + query pattern":

| storage | When it's a candidate | Representative | Classic pitfall |
|------|-----------|------|--------|
| Relational (PostgreSQL/MySQL) | complex queries, transactions, strongly consistent by default | RDS, Spanner | single-machine write ceiling → sharding is expensive |
| KV (DynamoDB/Cassandra) | massive writes, key lookups, eventually consistent is acceptable | Dynamo | no secondary queries; a wrong partition key design is a disaster |
| Document (MongoDB) | flexible schema, nested reads | Mongo | weak transactions; index bloat |
| wide-column (Cassandra) | time-series, logs, write-heavy | Cassandra | poor LWT performance; read amplification |
| Full-text search (Elasticsearch) | search, prefixes, aggregations | ES | near real-time ≠ real-time; JVM operations are heavy |
| Time-series (TimescaleDB/ClickHouse) | monitoring metrics | ClickHouse | updates are hard |
| Object storage (S3) | large files, immutable data | S3 | it isn't a database; list operations are expensive |

**selection script**:
> "20K write QPS, point lookups by user_id, eventually consistent is acceptable → Cassandra/DynamoDB fits. If the team only has MySQL, 64 logical shards can carry it too, at the cost of losing cross-row transactions — here I'd pick X because..."

### Sharding
- **Choosing the shard key** is the core test: high cardinality, coverage of the query paths, avoiding hot spots (user_id beats auto_increment id)
- scope sharding (easy scope queries, prone to skew) vs hash sharding (even distribution, range queries are dead) — **there's no win-win, only trade-offs**
- Scaling path: consistent hashing / pre-split into enough logical shards (64→1024) and migrate physically
- Cross-shard transactions: Saga / two-phase commit / design so the transaction stays on a single shard whenever possible (**if a single shard can do it, don't go cross-shard**)

## 4. replication and consistency

**E4 layer**: primary-replica replication, the CAP theorem.

**E5 layer**:
- **replication is an availability tool, sharding is a capacity tool** — don't conflate them
- synchronous replication (no data loss, slow writes) vs async (fast, loses data on failover) vs semi-sync (the middle ground) — MySQL's `semi-sync` and PostgreSQL's `synchronous_commit` both sit on this spectrum
- **read-your-writes (read-your-writes)**: a user posts and immediately refreshes but can't see their own post → sticky the session to the primary / read-timestamp routing. **Must-know for social problems**
- CAP in plain English: "Network partitions are inevitable, so what you're really choosing is whether you keep C or keep A during a partition. A ticketing system keeps C and denies service; a feed keeps A and returns stale data."
- Consistency spectrum: linearizable > ordered > causal > eventually consistent — **be able to say which notch the business sits at and why**

## 5. Message Queue

**E4 layer**: Kafka decouples and smooths peaks.

**E5 layer**:
- **Three tiers of delivery semantics**: at-most-once (losing messages is fine) / at-least-once (no loss but duplicates, **the default tier**) / exactly-once (expensive, depends on idempotency or transactions)
- **Idempotency is the antidote to at-least-once**: a dedup key + dedup window / a unique constraint in the database / version numbers. Explaining clearly "how do you prevent a consumer restart and replay from double-charging" is an E5 signal
- Kafka core concepts: partitions are the unit of parallelism, offset commit semantics, consumer group rebalancing jitter, and **partition count = the ceiling on parallelism, so plan it up front**
- Ordering is only guaranteed **within a single partition** — partition by user_id if you need messages from the same user to stay ordered
- Peak shaving script: "A write traffic surge of 50K QPS and the DB can only take 10K — the queue buffers it and consumers pull at their own capacity; the cost is that latency goes from milliseconds to seconds, and that latency is acceptable to the business."

## 6. CDN & Edge

**E4 layer**: put static resources on a CDN.

**E5 layer**:
- Good fit: immutable content (images, videos, JS bundles); **bad fit**: personalized dynamic content (API responses)
- The middle ground: edge cache + a short TTL (seconds) — "the only personalization is the user name in the header, so the client stitches that in"
- The invalidation problem: CDN refreshes are eventually consistent — "I changed the config, when does it take effect across the whole network?" is where real incidents cluster
- Video: HLS/DASH segmentation is a natural fit for CDN; the transcoding pipeline (upload → raw storage → transcoding queue → multiple bitrates → CDN)

## 7. Other High-Frequency Odds and Ends (One Sentence + Deep-Dive Hooks)

- **API Gateway**: auth, rate limiting, routing behind a single entry point — "pull the cross-cutting concerns out of the services"
- **Service discovery**: K8s DNS / Consul — "instances come and go, so how does the caller know the address"
- **distributed lock**: Redis `SET NX PX` + fencing token (**why you need a fencing token** is a classic deep-dive question); or ZooKeeper/etcd (more robust, heavier)
- **heartbeat and failure detection**: the trade-off between the timeout threshold and false positives; split-brain and quorum
- **Bloom filter**: approximate membership testing, no false negatives but possible false positives — "it tells cache penetration that this key doesn't exist"
- **consistent hashing**: see §1, applies to both caching and sharding

## 8. Stitching the Building Blocks Into a Script

Whenever a building block comes up in an interview, use the standard three-part form:

> "Here I'd add a Redis tier for cache-aside (**what it is**). Because 17K read QPS won't fit on a single database (**why**). The risk is an invalidation storm, so I'd use TTL jitter plus a logical expiration fallback; and I'd monitor hit rate as a core SLI (**what if it breaks**)."

## Next Module

→ [06 · Classic Problems Overview](06-classic-questions-overview.md)
