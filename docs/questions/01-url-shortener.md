# Q1 · URL Shortener

> Difficulty: Easy | archetype: read-heavy + cache
> Looks simple, but it's actually the **best warm-up problem for drilling through three deep-dive spots: ID generation, caching, and the redirect path**.

## 1. Problem Statement

"Design a URL shortener like bit.ly. A user submits a long URL and gets back a short URL; visiting the short URL redirects to the original URL."

## 2. Clarifying Questions

| Question | Why ask it |
|------|---------|
| Does the short URL need a custom alias? (`bit.ly/my-wedding`) | Affects the entire ID generation design |
| Do links expire, or are they permanent? | Storage estimation and cleanup strategy |
| Do we need click counting (an analytics feature)? | **The biggest hidden requirement in this problem**—it decides whether clicks are written into a log pipeline |
| DAU / read and write volume? | Determines whether a cache layer is necessary |
| Read latency requirement? (the meaning of 301 vs 302) | Your opening to get into HTTP details |

**Scope convergence script**: "I'll focus on the three core features—create, read, and redirect—plus the click counter; custom domains and expiration management go to follow-up."

## 3. Estimation Walkthrough (assume 100M DAU, 5 reads and 0.1 writes per user)

```
read QPS ≈ 5,800 (peak ~17K)   write QPS ≈ 115
storage ≈ 500B × 4B records/year ≈ 2TB/year (with index ×2 = 4TB)
bandwidth ≈ 17K × 1KB ≈ 17MB/s (light)
takeaway → the read path must be cached; writes and storage are under no pressure at all → a textbook read-heavy scenario
```

## 4. High-Level Design

```
client ──POST /url──▶ LB ──▶ URL shortener (stateless) ──▶ ID generator
                                   │
                                   ▼
                                MySQL (primary/replica)
client ──GET /abc123──▶ LB ──▶ URL shortener ──▶ Redis cache ──(miss)──▶ MySQL
                                   │
                                   ▼
                              return 301/302 + original URL
(click event ──▶ Kafka ──▶ analytics pipeline, if the problem asks for counting)
```

Data flow, spoken version: "Creation goes through the service tier to get a unique ID, encodes it into a short code, and writes to the primary DB; access goes to the cache first, and on a miss it goes back to the source and backfills."

## 5. Data Model

```sql
-- MySQL is enough (strongly consistent + transactions + simple single-table indexes)
short_url (id BIGINT PK, -- global auto-increment or Snowflake ID
           code VARCHAR(7) UNIQUE,-- base62 encoding
           original_url TEXT,
           created_by BIGINT,
           expires_at TIMESTAMP NULL,
           created_at TIMESTAMP)
INDEX (created_by), INDEX (expires_at)
```

**Why SQL and not DynamoDB**: "Writes are only in the hundreds of QPS, so a relational database is under no pressure at all, and expiration cleanup plus per-user queries both need flexible indexes—bringing in a KV store would actually lose those. At most this scales to a single database with read/write separation."

## 6. Deep Dives

### Deep Dive A · How to generate IDs (the classic part of this problem)

| Approach | Pros | Cons / what an E5 must say out loud |
|------|------|---------------------|
| DB auto-increment ID | Simple, no collisions | SPOF; exposes the total count (crawlers can enumerate your volume); after sharding you need step-based ranges |
| UUID | No coordination | 128 bits is too long and unordered (B+ tree page splits) → ruled out immediately for a short-code scenario |
| **Segment mode (Leaf / Meituan)** | The DB only hands out segments, so it absorbs high write volume | The segment service needs HA; monotonic only within a segment |
| Snowflake | Roughly increasing, decentralized | The **clock rollback** problem; machine bits need planning |

E5 standard answer: "Write QPS is only 115, so I'll take DB auto-increment plus a segment cache and that's enough—introducing Snowflake's timestamp bits for a low-frequency write is pure over-engineering. If the problem changed to billion-scale writes, I'd switch to Snowflake and handle clock rollback (wait, error out, or reserve backup bits)."

### Deep Dive B · Base62 and collisions

- 62^7 ≈ 3.5 trillion, so 7 characters of encoding is plenty
- auto-increment ID → base62 is **collision-free by construction** (this is the hidden dividend of choosing auto-increment, and saying it out loud earns you points)
- If you take the first 7 characters of a hash → you must handle collisions (look up the DB and retry) → being able to say "that's why I don't choose hashing" is how you demonstrate trade-off reasoning

### Deep Dive C · 301 vs 302 and caching

> "A 301 permanent redirect gets cached by the browser—**subsequent clicks never reach our servers, so click counting is dead**. If you want the counts, use a 302 temporary redirect; the cost is that every request comes back to the origin, so all the read pressure lands on us. So the choice comes down to whether the business cares more about counting accuracy or response latency."

This paragraph is the most E5 thirty seconds of this problem.

### Deep Dive D · hot spot

"A popular short URL (going viral) hammers a single key on a single Redis shard → use a local cache (in-process LRU) as the first tier plus logical expiration to prevent a stampede."

## 7. Red Flags

- Opening with "shard it across 128 databases" (sharding what—115 QPS of writes?)
- Using hashing without discussing collision handling
- Only saying "add Redis," without discussing hit rate, invalidation strategy, or hot spots
- No awareness of the 301/302 counting problem (only reacting after the interviewer points it out)

## 8. One-Minute Elevator Pitch

"The read/write ratio is 1000:1. Writes go through DB auto-increment plus base62 encoding, which is collision-free by construction; reads go through Redis cache-aside plus an in-process second-level cache to absorb hot spots. 301 vs 302 depends on whether we need click counting—if we do, we have to use 302 and hit the origin, which makes cache hit rate a core SLI. Click logs go to Kafka asynchronously for analytics. Storage is on the order of 2TB per year, so a single database with a primary and replicas is enough, and I won't introduce any distributed component at this scale."

---
← [06 classic problems overview](../06-classic-questions-overview.md) | [Q2 rate limiter →](02-rate-limiter.md)
