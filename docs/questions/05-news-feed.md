# Q5 · News Feed

> Difficulty: Medium | archetype: Fan-out write amplification + read-heavy
> A Meta favorite. **The hybrid push vs pull approach** is the main axis, and the consistency details are what differentiate you at E5.

## 1. Problem Statement

"Design the news feed for Facebook/Twitter: users post, follow other users, and scroll through a feed of posts from the people they follow."

## 2. Clarifying Questions

| Question | Intent |
|------|------|
| Feed ranking: time order or algorithmic ranking? | Decides whether you need a ranking service (this problem assumes time order; mention ranking in one line) |
| What is the follow limit? (300 follows vs 1 million followers) | **The dividing line between push and pull** |
| Are there celebrity users in the follower graph (millions of followers)? | Same as above — the root cause of the hot spot problem |
| What are the refresh semantics for a new feed? | The read-your-writes vs eventual consistency choice |
| Do we need lightweight interactions like view counts? | Adds a write-side storm variant |

## 3. Estimation Walkthrough

```
100M DAU; each user follows 300 people, posts 2 per day, refreshes 20 times
write: 200M posts/day ≈ 2.3K posts/s, peak 10K/s
read: 2B feed requests/day ≈ 23K QPS average, peak 100K+ QPS ← reads are 20x writes
write fan-out cost: 1 post × average 300 followers = average 600K feed-item writes/s (peak 3M/s)
takeaway → write fan-out's write QPS is ~300x the posting QPS; this is the source of every tension in this problem
```

## 4. High-Level Design

```
posting: client ──▶ Post API ──▶ Post Service ──▶ Post Store (Cassandra/sharding)
                                        │
                                        ▼
                              Kafka (fan-out event)
                                        │
                              ┌─────────┴─────────┐
                              ▼ ▼
                     Fanout Worker (push path)   Follow Graph Service
                       append post_id to each    (Graph)
                       follower's feed cache     │
                              │ │
                              ▼ ▼
Feed refresh: Feed API ──▶ User Feed Cache (Redis List, the pushed timeline)
                       │ (cache miss / hybrid path)
                       └──▶ On-the-fly feed assembly (pull): look up follow list → batch-fetch recent posts → merge
```

## 5. Data Model

```
posts (post_id PK, user_id, content, media_refs, created_at) -- sharding: post_id
follows (follower_id, followee_id, created_at) -- edge table, sharding: follower_id
            (inverted list followees_of:{user} for pull; forward list followers_of:{user} for push)
feed_cache: Redis List per user (holds post_ids, capped at e.g. 800)
inbox (user_id, last_read_post_id) -- cursor
```

## 6. Deep Dives

### Deep Dive A · Push vs Pull vs Hybrid (the main course)

| | Push (fan-out on write) | Pull (fan-out on read) |
|---|---|---|
| Read latency | Extremely low (cache is pre-built) | High (on-the-fly aggregation of 300 people's posts) |
| Write amplification | Enormous (× average follower count) | None |
| Storage | A feed replica per user | No replica |
| New post visibility | Async latency (seconds) | Immediate |
| Celebrity users | **Disaster** (one post = 1 million writes) | Naturally immune |

The E5 answer — **hybrid**:
- Regular users (followers below a threshold, e.g. 10K): push into followers' caches within budget
- Celebrity users: no fan-out; **merge on the fly at read time** (feed API response = regular posts from cache + celebrity posts pulled online, merged by time)
- "The threshold is a tunable parameter, adjusted by monitoring the fanout workers' consumption latency — **tying system behavior to operational metrics** is what makes a complete design."

### Deep Dive B · Consistency: you post, refresh, and don't see it (Meta loves this)

Scenario: user A posts → the message is still in Kafka → A immediately refreshes the feed.
- Approach 1: **read-your-own-writes** — the Feed API checks `inbox.last_read`; if the user's own latest post hasn't landed in the cache yet, splice it in directly from the post store
- Approach 2: the posting API synchronously writes the user's own feed cache (you are always your own follower... no you aren't) — at minimum, synchronously write your own "profile timeline"
- Script: "For everyone else, eventually consistent second-level convergence is perfectly acceptable; for the author themselves, humans have zero tolerance for 'what I just posted isn't visible' — **allocate your consistency budget per user relationship**. That's product psychology, not just technology."

### Deep Dive C · Feed cache details

- Structure: Redis List of post_ids; LPUSH new posts + LTRIM to cap the list (800 entries; deep pagination falls back to the post store)
- Pagination cursor: the `(last_post_id, offset)` dual parameter — "a pure offset skips or duplicates items when new posts are written in; the cursor must be anchored to a post_id"
- Cache miss (cold user): assemble on the fly via pull, then backfill
- Hot-spot user feed cache sharding: consistent hashing on user_id

### Deep Dive D · Propagating deletions and edits

The push model's hidden debt: **deleting a post means removing it from every follower's cache**.
- The feed cache stores post_ids, not content → deleting a post only requires a tombstone in the post store; the read side filters out invalidated posts at render time (lazy deletion)
- "Store IDs, not content — converge content to a single source of truth, and only then can the cache afford to be big." That's the precondition that keeps the push model alive.

### Deep Dive E · Ranking (one sentence)

"Real products use algorithmic ranking (predicted engagement score), which turns feed assembly into a two-stage **candidate retrieval + scoring** pipeline (recall a few hundred → a lightweight model scores them down to a few dozen). The core architecture doesn't change; you add a ranking service. Today I'll build the chronological timeline."

## 7. Red Flags

- Pure push with no discussion of celebrity-user write amplification
- Pure pull with no math on the read latency of on-the-fly aggregation across 300 people
- Feed cache stores full post content (deletion/edit disaster)
- No awareness of read-your-own-writes (only reacts when the interviewer asks "your own post doesn't show up")
- Pagination uses a pure offset

## 8. One-Minute Elevator Pitch

"Reads are 20x writes, and the core tension is fan-out write amplification. Hybrid approach: regular users get push — the post flows through Kafka into fanout workers that write into each follower's Redis feed cache (storing post_ids, not content, so deletions go through lazy tombstone filtering); celebrity users get no fan-out and are pulled and merged on the fly at read time, with the threshold tuned by fanout consumption latency. Consistency is tiered: eventual consistency for everyone else, read-your-own-writes spliced in synchronously for the author. Pagination uses a cursor anchored on post_id. Algorithmic ranking replaces the assembly layer with a two-stage recall + scoring pipeline; the architecture skeleton doesn't change."

---
← [Q4 chat system](04-chat-system.md) | [Q6 message queue →](06-distributed-message-queue.md)
