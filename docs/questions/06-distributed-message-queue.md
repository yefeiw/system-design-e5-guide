# Q6 · Distributed Message Queue (Design a Kafka)

> Difficulty: Medium–Hard | archetype: write-heavy + peak shaving (the build-your-own-wheel edition)
> The inverted problem — not "design X using Kafka," but "design Kafka itself." It probes your fundamentals on **storage engines + delivery semantics** more than anything else.

## 1. Problem Statement

"Design a distributed message queue: producers send messages, consumers subscribe and process them. It needs high throughput, no message loss, and support for parallel consumption within a consumer group."

## 2. Clarifying Questions

| Question | Intent |
|------|------|
| Throughput target? Latency target? (1M+ QPS + millisecond level?) | Input to the storage model choice |
| Message size? (KB-scale vs MB-scale) | Append-only log vs per-message index |
| Delivery semantics requirement? At-least-once or exactly-once? | Sets the deep dive direction for the whole problem |
| Do messages need a retention period? (consume-and-delete vs retention-based) | **The dividing line between Kafka and traditional MQs** |
| Do we need transactional messages / delayed messages? | Scope control |

## 3. Estimation (used here to justify design decisions)

```
1M msg/s × 1KB = 1 GB/s ingress; 7-day retention = 600TB
A single NVMe node sustains ~2 GB/s sequential writes → an ordered append-only log is the only viable write model
Random writes only get ~100 MB/s → any "one record per message" design dies on the spot
takeaway → the append-only log isn't a choice, it's a law of physics
```

## 4. High-Level Design

```
Producer ──▶ Broker cluster
              ┌───────────────────────────────────────┐
              │ Topic: orders │
              │ Partition 0 (broker 1) [log: seg0 seg1 seg2...] │
              │ Partition 1 (broker 2) [log: ...] │
              │ Partition 2 (broker 3) [log: ...] │
              └───────────────────────────────────────┘
              Each partition = Leader (writes) + ISR replicas (synchronous replication)
Metadata/leader election: Controller / ZooKeeper / Raft
Consumer Group: within each group, each partition is assigned to exactly one consumer instance
              offsets committed to an internal topic (__consumer_offsets)
```

## 5. Data Model (where the core insight lives)

```
Partition = a sequence of append-only log files (segments) on disk
Each message has only an 8-byte "primary key": the offset (a monotonically increasing sequence number within the partition)
index = sparse index (one offset→file position entry per 4KB), binary search locates the segment
```

**The three-hit script**:
1. "Messages are never deleted or modified, only appended — **turn random writes into sequential writes**, and throughput improves by two orders of magnitude. That's the first principle of the entire design."
2. "Consuming doesn't delete a message, it just moves the offset — **consumption is read + pointer**, so any number of consumer groups can replay the same topic (the essential difference from RabbitMQ's consume-and-delete model)."
3. "The partition is the atomic unit of parallelism: producers hash by key to a partition (same key = ordered), and consumption parallelism ≤ partition count — **figure out what level of ordering the business actually needs** before fixing the partition count."

## 6. Deep Dives

### Deep Dive A · The three-segment syllogism of no message loss (must-know)

Walk through each segment; each one has pitfalls:

**① Producer → Broker**:
- acks=0 (fire and forget) / acks=1 (leader persisted) / **acks=all (full ISR persisted)**
- "acks=all + min.insync.replicas=2 + replication.factor=3 is the complete 'no loss' configuration — saying acks=all without mentioning ISR shrinkage (writes rejected when min.insync isn't satisfied) means you haven't configured anything"
- Retry + idempotent producer (the broker dedupes by PID + sequence number)

**② Inside the Broker**:
- The ISR (in-sync replicas) mechanism: replicas that fall too far behind get kicked out of the ISR
- "unclean.leader.election: allowing a lagging replica to become leader = data loss; forbidding it = reduced availability — **this is CAP made concrete inside an MQ**"

**③ Broker → Consumer**:
- Consume first, commit the offset after: at-least-once (crash → replay, possible duplicates)
- Commit first, consume after: at-most-once (crash → lost messages)
- "When the offset gets committed *is* the delivery semantics selector."

### Deep Dive B · How exactly-once actually works

- The idempotent producer only covers a single partition within a single session → going cross-partition requires **transactions** (two-phase commit to an internal topic)
- End-to-end exactly-once = transactions + the consumer side **reading only committed data** (read_committed)
- "Kafka Streams / Flink two-phase-commit checkpoints bring the sink into the transaction too — the cost is a significant throughput drop. **First ask whether the business can live with at-least-once + idempotent consumption**; most of the time it can, so don't pay for exactly-once." (This sentence itself is an E5 signal.)

### Deep Dive C · Consumer group rebalancing

- Consumers going on/offline → partition reassignment → **the entire group pauses consumption during rebalancing** (stop-the-world)
- The group coordinator probes with heartbeats/session timeouts
- Pitfall: GC pauses mistaken for death → jitter storms; tune session.timeout + heartbeat frequency + rebalancing protocol upgrades (incremental/cooperative rebalancing)
- "Long-tail GC on one consumer amplifies into consumption latency for the entire queue — **the queue's stability is held hostage by its slowest consumer**."

### Deep Dive D · Comparison wrap-up vs traditional MQ

| | Kafka model | RabbitMQ model |
|---|---|---|
| Storage | Log retention, offset pointer | Queue, consume-and-delete |
| Throughput | Millions-level | Tens-of-thousands-level |
| Routing | Simple (topic + key) | Flexible (exchange routing) |
| Repeated consumption | Natively supported | Requires DLX workarounds |
| Delayed messages | Not native | Plugins/native |

"Selection comes down to three questions: do you need throughput? Do you need replay? Do you need complex routing?"

## 7. Red Flags

- One database row / one Redis record per message (no awareness of the physical advantage of sequential writes)
- Saying "acks=all means no loss" (no mention of ISR shrinkage)
- Conflating exactly-once with at-least-once + idempotent
- No discussion of partitions and ordering
- Not knowing that rebalancing is stop-the-world

## 8. One-Minute Elevator Pitch

"The first principle is turning random writes into sequential writes: a partition = an append-only log + a sparse index, messages are immutable, and consuming just moves the offset — which is what enables multiple groups and replay. No-loss is a three-segment configuration: producer acks=all + idempotent retry, broker ISR synchronous replication + min.insync.replicas rejecting writes, consumer consumes first and commits after (at-least-once); when exactly-once is required, add transactions + read_committed, but I'd first ask whether idempotent consumption can substitute. The partition is the atomic unit of both parallelism and ordering — same key, same partition, that's the ordering guarantee; partition count caps consumption parallelism. Rebalancing is a stop-the-world storm, and consumer-side stability (GC, long tasks) is the queue's hidden constraint."

---
← [Q5 News Feed](05-news-feed.md) | [Q7 Ticket Booking →](07-ticket-booking.md)
