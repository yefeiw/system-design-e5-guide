# 06 · Classic Problems Overview

> You don't need to grind 50 problems to prep for E5. **Taking 8–10 problems to the point where you can deep dive one layer down beats skimming 30 problems where you've "drawn the diagram" for all of them.**

## 1. problem grading (Aligned with the Hello Interview Difficulty Framework)

### Easy (Must Be Flawless)
- URL shortener ([Q1](questions/01-url-shortener.md)) — tests ID generation, caching, redirects
- rate limiter ([Q2](questions/02-rate-limiter.md)) — tests algorithmic deep dives + the distributed clock
- Top-K / Heavy Hitters ([Q3](questions/03-top-k-heavy-hitters.md)) — tests streaming algorithms + the accuracy trade-off

### Medium (The E5 Core Strike Zone)
- chat system ([Q4](questions/04-chat-system.md))
- News Feed ([Q5](questions/05-news-feed.md))
- distributed message queue ([Q6](questions/06-distributed-message-queue.md))
- Ticket Booking ([Q7](questions/07-ticket-booking.md)) — flagged by Hello Interview as E5 high-frequency
- ad click aggregation ([Q8](questions/08-ad-click-aggregator.md)) — E5 high-frequency, the whole stream-processing bucket

### Hard (Only If You Have Room Left)
- Google Drive / Dropbox (file storage + chunking + sync)
- YouTube / Netflix (video transcoding pipeline + CDN)
- Google Maps (tiled index + route planning)
- search engine (crawl → index → ranking)
- Uber (geolocation index via GeoHash + matching and dispatch)

## 2. The archetype Behind Every Problem

Almost every problem is a variant of these five archetypes. Recognize the archetype and you've recognized where the interviewer is scoring:

| archetype | Core tension | Problems it covers |
|------|---------|---------|
| **read-heavy + cache** | hit rate vs consistency | short URL, Feed, profile page |
| **write-heavy + peak shaving** | throughput vs latency vs data loss | logs, click aggregation, message queue |
| **state sync (long-lived connection)** | connection scale vs push latency | chat, collaborative editing, gaming |
| **concurrency resource contention** | consistency vs availability | ticketing, flash sale, money transfer |
| **Fan-out write amplification** | push-on-write fan-out vs aggregate-on-read | Feed, notifications, social graph |

**In the later stage of practice, drill "archetype transfer" specifically**: when you get an unfamiliar problem, spend 30 seconds classifying its archetype, then apply your deep-dive repertoire directly.

## 3. The practice workflow for Every Problem (Don't Do It Backwards!)

```
1. cold-solving (45 min, timed, whiteboard, recorded) ← the most important step, never read the walkthrough first
2. Compare against the walkthrough (30 min) and flag three kinds of gaps:
   □ Process gaps (forgot to ask X / lost control of the clock)
   □ Knowledge gaps (don't understand how a component works)
   □ Depth gaps (never dug down to the trade-off layer)
3. Write a one-page personal cheat sheet (a decision checklist, not a knowledge summary)
4. Redo the same problem 3 days later (without the cheat sheet) and record it for comparison
5. Present a 15-minute version to a friend a week later
```

## 4. deep-dive repertoire: The "Pick One of Three Deep Dives" Every Problem Needs

In the interview's deep-dive phase (Phase 5), the interviewer almost always picks one of these three directions, so have all three ready:

- **Data layer**: storage selection / shard key / index / consistency
- **Failure layer**: SPOF / overload / partial invalidation (split-brain, network partition)
- **Evolution layer**: how the architecture changes once traffic goes ×10 / ×100

## 5. This Repo's Problem Walkthrough Structure

Every problem follows the same structure (fully aligned with [03 delivery framework](03-delivery-framework.md)):

1. **problem statement** (the vague card the interviewer reads from)
2. **clarifying-questions checklist** (what you should ask and the intent behind each question)
3. **estimation walkthrough** (numbers → takeaway)
4. **high-level design** (ASCII component diagram + data flow)
5. **data model**
6. **deep dive 1/2/3** (the three most common deep-dive directions for this problem and the E5-level answers)
7. **red-flag answers** (answers that immediately reveal you're not E5)
8. **one-minute elevator pitch** (your review card before a mock)

## Getting Into the Problems

→ [Q1 · URL shortener](questions/01-url-shortener.md)
