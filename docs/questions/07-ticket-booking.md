# Q7 · Ticket Booking (Flash Sales)

> Difficulty: Medium | archetype: concurrency + resource contention (consistency vs availability)
> Hello Interview flagged this as an **E5 high-frequency problem**. The essence: the moment of ticket-grabbing is a **strong-consistency problem**; everything else (search, impressions, payment) is a distractor.

## 1. Problem Statement

"Design a Ticketmaster: users search events, select seats (or grab tickets), place an order, and pay. A Taylor Swift-scale onsale: 500K people simultaneously fighting for 50K tickets."

## 2. Clarifying Questions

| Question | Intent |
|------|------|
| Seat-selection mode (specific seat) or seat-grab mode (first-come, any seat)? | The two concurrency models are completely different |
| Is overselling allowed? (airlines deliberately sell 105%) | **The anchor for the consistency requirement** |
| How long is the hold after placing the order? (shopping-cart holding period) | Lock granularity and duration |
| Is payment async? | Order state machine complexity |
| Do we need a waitlist? | The architecture's fourth answer |

## 3. Estimation Walkthrough

```
500K people flood in at the onsale moment; peak requests: 500K users × 1 refresh/s = 500K QPS
Sellable inventory: 50K tickets — the writes that truly need "strongly consistent contention" number ≤ 50K
takeaway → 99% of the traffic (browsing, refreshing, querying remaining tickets) never touches inventory.
       Separate reads from writes and the write conflict is actually tiny — this insight determines the entire architecture
```

## 4. High-Level Design

```
Before the onsale: static event pages + CDN (search/detail 100% cached, zero origin traffic)
During the onsale:
  user ──▶ queueing gateway (Virtual Waiting Room: ID issuance + token + heartbeat)
              │ (admit in batches, e.g. 2,000 people every 5 seconds)
              ▼
          Remaining-ticket query (Redis inventory snapshot cache, second-level refresh)
              │ tickets available
              ▼
          Order service ──▶ inventory decrement (atomic operation, see Deep Dive A)
              │ success        │ failure
              ▼                ▼
          Order created (15-min payment window)   re-queue / waitlist
              ▼
          Payment (async callback) ──▶ ticket issuance (seat/ticket code, written to the store, eventually consistent)
```

**The architecture narrative**: "I use a **queueing room to turn an uncontrollable traffic surge into a controllable, steady flow** — users take a number and enter, instead of all of them hammering the business layer; only admitted requests touch inventory. This one decision solves both overload and fairness (first-come, first-served) at the same time."

## 5. Data Model

```sql
events (event_id, venue, onsale_at, total_seats)
seats (seat_id, event_id, section, row, num, status) -- core table for seat-selection mode
       status: AVAILABLE → HELD → SOLD / back to AVAILABLE
orders (order_id, user_id, event_id, status, expires_at)
       status: PENDING → PAID → ISSUED / EXPIRED/CANCELLED
inventory counter: Redis inventory:{event_id} (a counter for seat-grab mode; per-seat row locks / optimistic locking for seat-selection mode)
```

## 6. Deep Dives

### Deep Dive A · Concurrency correctness of the inventory decrement (the main course)

**The approach spectrum, from low to high**:

1. **Database optimistic locking**: `UPDATE seats SET status='HELD' WHERE seat_id=? AND status='AVAILABLE'` (CAS semantics; rows affected = 1 means success)
   - Simple and correct; a single hot row may reach a few thousand QPS on the DB — combined with queueing rate limiting, **this is often enough**
2. **Redis atomic decrement**: `DECR` / Lua (check + decrement atomically) — sustains 100K QPS
   - **The key pitfall: what if Redis succeeds but the DB write fails** → async persistence + reconciliation; what if Redis dies (AOF + snapshots; worst case you lose second-level inventory state → fully rebuild from the DB on recovery)
3. **Pre-deduct inventory into in-memory segments**: split the inventory into 100 segments, each deducted independently — disperses the hot spot; the last segment merges
   - High complexity; only justified when a single row is the proven bottleneck

**The E5 closing**: "First compute the real contention volume: 50K tickets means at most 50K successful decrements, and after queueing, concurrent writes are only a few thousand QPS — **single-row optimistic locking is the right answer**, and pre-deducting in Redis is premature optimization. What's actually hard isn't surviving the load, it's **the correctness of the state machine**." (This shows the "match the approach to the scale" judgment interviewers look for.)

### Deep Dive B · Locks and holding periods (seat-selection mode)

- HELD state + `expires_at`: no payment within 15 minutes → auto-release
- Release path: scheduled scan (slow) → **delayed queue** (Redis ZSet/LevelDB ordered by expiry time) → expiry event drives the release
- "Expiry release must be the system's proactive behavior, not something waiting on user action — otherwise a scalper script can lock up the entire venue with seats it never pays for."

### Deep Dive C · Payment's eventual consistency

- Order state machine: PENDING → PAID → ISSUED, every transition idempotent (payment callbacks will retry)
- Payment succeeded but issuance failed: a reconciliation job scans PENDING_PAID → compensation (refund or manual intervention)
- "The payment gateway delivers its callback at least once, so my state transitions must be idempotent: `UPDATE orders SET status='PAID' WHERE order_id=? AND status='PENDING'` — CAS again."

### Deep Dive D · The formal argument for no overselling

When asked "how do you guarantee absolutely no overselling":
> "Every path converges on the same serialization point: a CAS on each seat row. The Redis snapshot is allowed to be stale (it shows tickets available, the order fails → prompt a retry; that's a UX problem, not a correctness problem); **the impression layer stays optimistic, the transaction layer stays absolutely pessimistic**. The proof of no overselling = every sale passed through that one CAS UPDATE."
("Optimistic at the impression layer, pessimistic at the transaction layer" — this sentence alone is E5-level synthesis.)

### Deep Dive E · Anti-scalper (bonus points)

- Device fingerprinting + account risk checks up front (filtered before entering the queueing room)
- Purchase limits (per-user quota, validated in the order service's memory + reconciled against the store)
- SMS verification codes to stretch out the human response rhythm

## 7. Red Flags

- Opening with search architecture (misreading the problem — this tests consistency, not search)
- Using a distributed lock on the entire event (granularity disaster) without discussing row-level CAS
- Decrementing inventory in Redis without addressing DB consistency
- No payment-timeout release path
- Pitching ZooKeeper/etcd leader election (solves the wrong problem)

## 8. One-Minute Elevator Pitch

"First, separate reads from writes: static pages fully on CDN, remaining-ticket reads from a cache snapshot (the impression layer is allowed to be optimistic); a virtual queueing room turns the traffic surge into a steady flow before orders are placed. The transaction layer's correctness rests on a per-seat CAS UPDATE (the AVAILABLE→HELD→SOLD state machine + delayed-queue expiry release + idempotent payment callbacks); Redis pre-deduction is added only when a single hot row is measurably insufficient, and always with a reconciliation fallback. The real contention volume is only 50K successful writes — match the approach to the scale; the proof of no overselling is 'every sale converges on a single serialized point.' Search and recommendations earn no points on this problem."

---
← [Q6 message queue](06-distributed-message-queue.md) | [Q8 ad click aggregation →](08-ad-click-aggregator.md)
