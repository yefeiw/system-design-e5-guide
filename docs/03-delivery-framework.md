# 03 · Delivery Framework: Running the 45 Minutes

> Every framework out there (Hello Interview, Design Gurus, Alex Xu's 4-step method) is fundamentally the same skeleton. This chapter unifies them into a single 6-phase timeline and provides script templates for each phase — the templates are not for memorization; they exist so that under pressure you don't have to reinvent the structure.

## 0. Why you need a framework

The most typical way an interview dies without a framework: the **time sink**. You spend 25 minutes in one step (usually diagramming or a deep dive), then the interviewer drags you away, and everything after is rushing with not a single real deep dive. A framework is fundamentally a **time budget** — you must stay conscious of the timeout on every phase.

## 1. Six-phase overview

```
┌────────────────────────────────────────────────────────────────────┐
│ Phase 1  Requirements clarification   5 min   question checklist + scale estimation │
│ Phase 2  Core feature confirmation    3 min   3–5 features + explicit non-goals     │
│ Phase 3  High-level design           10 min   component diagram + data flow + API skeleton │
│ Phase 4  Data model                   5 min   storage selection + table/KV structure + sharding │
│ Phase 5  Deep dive                   17 min   2–3 subsystems (negotiate with the interviewer) │
│ Phase 6  Wrap-up                      5 min   bottlenecks / monitoring / operations / out-of-scope items │
└────────────────────────────────────────────────────────────────────┘
```

> Note: Phases 3+4 together are the classic "high-level design"; Alex Xu's 4-step method merges 1+2 into "understand the requirements." You may adjust the structure slightly, but **the time discipline is non-negotiable**.

## 2. Phase 1 · Requirements clarification (5 minutes)

### Must-ask checklist (memorize it; run through it every session)

**Scale**
- What are the DAU / MAU? — determines every capacity number that follows
- Read-heavy or write-heavy? What's the read/write ratio? — determines the storage and caching strategy
- What's the peak multiplier (typical daily QPS × 3–5)?

**Feature boundaries**
- What is the core user flow? (have the interviewer describe a user story)
- Which clients must be supported? (Web / iOS / Android / third-party API)
- Do we need offline / flaky-network support?

**Non-functional requirements (raise these proactively — bonus signal)**
- Latency budget: is an order of magnitude like P99 read 200ms acceptable?
- Consistency: eventually consistent (second-level lag is OK) or read-your-writes?
- Availability target: how many nines? Is a degraded mode acceptable?
- Can data be lost, and how much? (logs vs billing are worlds apart)

### Script templates

> "Let me spend a few minutes aligning on requirements first. I'll ask some scale and boundary questions, then run a round of estimation so you can check whether the numbers point in the right direction."

> "The problem doesn't specify the read/write ratio. I'll assume 100:1 — if it's closer to 1:1, my storage choice later would be completely different, so I'd like to confirm this with you first."

**Key move**: **write the interviewer's answers on the whiteboard**. This isn't ceremony — every design decision afterward requires you to point back at them.

## 3. Phase 2 · Core feature confirmation (3 minutes)

**Distill 3–5 features** from the requirements, and state explicitly what you will not build:

> "Based on the requirements so far, I plan to focus on these four features: ①... ②... ③... ④.... I won't expand on multi-region DR or a public third-party API for now — I'll come back to them in the trade-offs at the end. Does that work?"

This is the most signal-dense 30 seconds of the E5 interview: **converging = you know what matters**. The interviewer will almost always say yes, and with that sentence you've defused the "over-engineering" risk in advance.

## 4. Phase 3 · High-level design (10 minutes)

Draw one component diagram that includes: client → gateway/LB → stateless service tier → cache → storage → (async) queue → downstream consumers.

Three disciplines:
1. **Draw-as-you-talk**, one sentence per box for its responsibility — details are saved for the deep-dive phase
2. **Number the arrows on the data flow**: walk through "a user sends a request — which components does it pass through?"
3. **Check in every 2–3 minutes**: "Is this high-level structure OK? If so, I'll move on to the data model." — this prevents you from sprinting 10 minutes in the wrong direction

API skeleton (optional, 30 seconds): list endpoint names + verbs only; skip parameter details unless you're interviewing for a Product Architecture round (see chapter 08).

## 5. Phase 4 · Data model (5 minutes)

- Choose storage: SQL / NoSQL / KV / time-series / search engine — **always with a reason** (see the selection table in chapter 05)
- Draw the table schema or KV structure: primary key, partition key, secondary index
- Proactively state your shard key choice and hot-spot risk: "I'm using user_id as the partition key — celebrity users' partitions will skew; in the deep-dive phase I'll cover how to handle that"

## 6. Phase 5 · Deep dive (17 minutes) — the E5 main battleground

### How to pick deep-dive spots

> "There are three spots worth going deep on: A (write amplification in the fan-out), B (the scaling path of the storage layer), and C (the consistency window). The one I most want to discuss is A, because it's the most likely to fail first. Which would you like to look at?"

That single sentence earns three high-value signals at once: initiative, risk instinct, and collaboration. The interviewer will usually pick A, or name the one they care about — **whatever they name, drill into that**; the direction they choose is usually the one they're scoring.

### The "drill-three-levels" method for deep dives

At each level, answer: **what it is → why → what if it breaks**

```
Level 1: The approach ("message delivery uses long polling")
Level 2: The why ("versus WebSocket: we don't need bidirectional low latency, long polling is simpler to operate and more compatible")
Level 3: The failure ("what about connection saturation? → partition the connection gateway + heartbeat-based eviction + fall back to polling")
```

### When you're pushed into a knowledge blind spot

> "I haven't operated this hands-on. Based on the principles of X and Y, my reasoning is Z — but I'm not certain, so please correct me if I'm wrong."

Honesty + reasoning beats fabrication. E5 interviewers have seen hundreds of candidates; making something up gets caught 100% of the time.

## 7. Phase 6 · Wrap-up (5 minutes)

Reserve the last 5 minutes (watch the clock ahead of time) and cover proactively:

- **SPOF**: which box in the diagram hurts most if it dies? → redundancy approach
- **Bottleneck**: if traffic ×10, what dies first? (usually one of: DB writes, the fan-out, network bandwidth)
- **Monitoring**: 3 SLIs (latency P99 / error rate / backlog)
- **Operations**: how to deploy, how to do a canary rollout, how to roll back
- **Out-of-scope items**: "geo-routing and cost optimization weren't covered today; I'd rank them below everything above."

Wrapping up with an "out-of-scope checklist" is an advanced signal: it shows you know where your design's boundary lies.

## 8. Common process incidents and rescues

| Incident | Rescue script |
|----------|---------------|
| Requirements clarification still going at 10 minutes | "I think we're about there — I'll assume the rest as I go and write them down here." |
| Interviewer changes the requirements mid-flight | Stop writing → "This affects A and B; C is unaffected. I'll change those two spots and continue." (a visible impact analysis = bonus points) |
| Freezing up for 30+ seconds | Say it out loud: "I'm torn between A and B; the difference is... I lean toward A." The silent freeze is what's fatal. |
| Running out of time | "8 minutes left — I want to prioritize finishing X, and cover Y and Z in one sentence each." Interviewers love this kind of time awareness. |

## 9. Practice method

1. **Internalize the templates**: run 3 problems holding the script templates, then throw the templates away
2. **Recording retrospective**: screen-record yourself running the full flow on your phone, watching specifically for — silences >10 seconds, actual time spent per phase, and whether the flow is followable from the interviewer's seat
3. **Live mocks**: only do [Interviewing.io](https://interviewing.io) or friend mocks after the framework is drilled in — don't waste live mocks on practicing the framework

## Next Module

→ [04 · Estimation & Numbers to Know](04-estimation-numbers.md)
