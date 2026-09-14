# 04 · Estimation & Numbers to Know (Back-of-Envelope)

> The point of estimation isn't to be precise, it's to **use numbers to bound the design space**. In an E5 interview, one sentence — "At 100M DAU and a 100:1 read/write ratio, peak read QPS is about 100K, a single MySQL database can't take that, so this has to be sharded" — is worth ten sentences of adjectives.

## 1. Numbers to Know (Memorize This Table First)

### 1.1 latency numbers (Jeff Dean's Classic Table, 2020s Update)

| Operation | order of magnitude | Memory anchor |
|------|------|---------|
| L1 cache hit | ~1 ns | — |
| Mutex lock acquisition | ~15 ns | — |
| memory random read | ~100 ns | 1 KB of data |
| **SSD random read** | **~150 µs** | 1000× slower than memory |
| RTT within the same datacenter | ~0.5 ms | — |
| **Cross-AZ RTT** | **~1–2 ms** | Same region |
| **Cross-region RTT (US East ↔ US West)** | **~60–80 ms** | The speed-of-light physical limit |
| HDD seek | ~10 ms | — |
| sequential reads of 1 MB (SSD) | ~1 ms | — |

**Interview killer line**: "One cross-region RTT is long enough for 600,000 memory reads — that's why multi-region synchronous replication is a last resort."

### 1.2 availability conversion

| 9s | Annual downtime | How to use it |
|----|--------|------|
| 99% | 3.65 days | You can't call this HA |
| 99.9% | 8.7 hours | The baseline for an ordinary production service |
| 99.99% | 52 minutes | Requires automatic failover |
| 99.999% | 5 minutes | Requires eliminating all manual intervention |

**killer line**: "When we say 99.95%, we mean a monthly budget of 22 minutes — if each deploy burns 5 minutes of it, we can tolerate at most 4 failed deploys."

### 1.3 single-machine capacity Intuition (2020s Hardware)

| resource | single machine order of magnitude |
|------|---------|
| memory | 64–256 GB |
| SSD | 4–16 TB |
| NIC | 10–25 Gbps ≈ 1–3 GB/s |
| Web server concurrent connections | ~10K–65K (port/memory limits) |
| Safe write QPS on a single MySQL instance | ~5K–10K (commodity hardware, simple writes) |
| Single Redis instance | ~100K ops/s |
| Single Kafka broker | ~100K+ msg/s (small messages) |

**These numbers are what decide when you have to go distributed** — if your write QPS estimate comes out at 50K, a single database is guaranteed to fall over, and now you have the numbers to justify sharding or queue-based peak shaving.

## 2. estimation workflow (4 Steps, Told in 2–3 Minutes)

1. **user scale** → DAU (given in the prompt or assumed)
2. **read write QPS** → per-user operation count × DAU ÷ 86400, then × peak multiplier ×3–5
3. **storage** → size of one record × records added per day × retention period (**don't forget the index and replica amplification factor, usually ×2–3**)
4. **bandwidth** → QPS × average request/response size

### Full Example: URL Shortener (Prompt: 100M DAU, 5 reads and 0.1 writes per user)

```
read QPS = 100M × 5 / 86400 ≈ 5,800 QPS, peak ×3 ≈ 17K QPS
write QPS = 100M × 0.1 / 86400 ≈ 115 QPS, peak ≈ 350 QPS
storage = 500 bytes/record × 4B records/year ≈ 2 TB/year (with index ×2 ≈ 4 TB)
bandwidth = 17K × 1 KB ≈ 17 MB/s read out — that's light
takeaway → reads need a cache (17K QPS won't fit on a single database that also serves every miss), writes are
           under no pressure at all, storage fits in a single sharding → this is the textbook "read-heavy, write-light" scenario
```

Note that last line: **an estimation has to land on a design takeaway**. Crunching the numbers and saying nothing is the same as not doing them at all.

## 3. E5-Level estimation bonus signal

### 3.1 State the Error Margin on Your Assumptions

> "I'm assuming 100M DAU, and even if I'm off by an order of magnitude at 10M, the takeaway doesn't change — because the bottleneck is the cache, not the number of database rows." (the robustness mindset of back-of-the-envelope estimation)

### 3.2 Do the Money Math

> "Storage is 4 TB/year, S3 is $0.023/GB/month, so a year of storage costs under $1,200 — cost isn't the constraint. But if there are images, object storage + CDN is where the real money goes."

### 3.3 Do the "People" Math

Big-tech interviewers like hearing capacity planning land on operations:
> "17K QPS at peak, one instance carries 3K, so I need 6 instances plus N+2 redundancy ≈ 8 instances, one ASG is enough, no multi-region needed."

## 4. Common Pitfalls

| Pitfall | Fix |
|----|------|
| Forgetting the peak multiplier (sizing capacity off average QPS) | average ×3–5 is the real capacity target |
| Forgetting replica and index amplification | multiply storage by 2–3 up front |
| Mixing up units (GB / Gbps / GB/s) | write the bit-vs-byte checklist at the top of the whiteboard before you touch the marker |
| Estimating without landing it | follow every number with a "so therefore..." |
| Chasing exact decimals | keep 1–2 significant digits; back-of-the-envelope is order-of-magnitude thinking |

## 5. practice checklist (10 Minutes a Day for Two Weeks)

Do QPS / storage / bandwidth in your head for these scenarios, three minutes each:

1. Twitter Feed: 200M DAU, 20 refreshes per user
2. chat system: 50M DAU, 40 messages per user, messages retained for 1 year
3. video site: 100M DAU, 30 minutes per user, average bitrate 2 Mbps
4. log system: 10K servers × 100 logs/s per server × 500 B/log
5. ad click: 1B impressions/day, 1% CTR, 200 B per click record

Work the answers out yourself. The point is to drill the closing sentence — "numbers → design takeaway" — until it's instinct.

## Next Module

→ [05 · Building Blocks](05-building-blocks.md)
