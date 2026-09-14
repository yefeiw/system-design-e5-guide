# 02 · E5 / Senior Expectations

> For the same problem, the "passing answer" at E4 and at E5 is completely different. This chapter pinpoints exactly where the difference lies, and how to train each dimension.

## 1. E5 vs E4: five essential differences

### 1.1 Driving the conversation: who is in the driver's seat

| | E4 | E5 |
|---|---|---|
| Requirements clarification | interviewer prompts "should we talk about scale?" | you proactively list 5–8 clarifying questions and decide yourself which assumptions hold |
| Direction | interviewer says "let's talk about storage" | you say "I think the biggest risk is in the fan-out — I want to go deep there first" |
| Deep-dive spots | wherever you get taken | you propose 2–3 deep dive candidates and **negotiate** the priority with the interviewer |

**In one sentence: at E4 you are being interviewed; at E5 you are running a design review, and the interviewer is the reviewer.**

### 1.2 Depth: punch through at least one subsystem

At E4, drawing the right block diagram and naming the right components is basically enough to pass. The implicit E5 requirement: **at least one subsystem must go deep enough to answer "how is the data actually stored, how is consistency maintained, what if it breaks."**

The litmus test for "punched through" (using cache as the example):
- **What a mid-level answer sounds like:** "add a Redis cache layer to reduce database load"
- **What a senior answer sounds like:** "it's write-heavy, so write-through would create a cold-data problem — I'd choose cache-aside + a short TTL; for invalidation storms, a logical-expiration fallback; for hot keys, a second layer of local cache; and the cache hit rate needs to stay above 95% to justify this complexity — I'd put that number into monitoring"

### 1.3 Trade-off awareness: every decision has a cost

The default sentence pattern of an E5 answer is not "I use X," but:

> "I plan to use X. The alternatives are Y and Z. I chose X because A is a hard constraint in our scenario and X is strongest on A; the cost is that B gets worse, and B is a soft constraint for us, which we can mitigate with C."

A decision presented without its trade-off is, in an E5 interviewer's eyes, no decision at all.

### 1.4 Operational maturity: how the system survives after launch

E5 defaults to covering these proactively (without the interviewer prompting):
- **Monitoring**: the key SLIs (latency P99, error rate, queue backlog, cache hit rate)
- **Rate limiting and degradation**: under overload, what gets dropped first (degrade non-core features, return stale data)
- **Capacity planning**: single-machine capacity × redundancy factor, and the trigger conditions for scaling out
- **Failure drills**: how the system behaves when a given component dies (SPOF, split-brain, data-loss window)

### 1.5 Handling ambiguity

E4 treats ambiguity as an obstacle ("the problem wasn't clear"); E5 treats it as leverage:

> "The problem doesn't specify the read/write ratio, so I'll assume 100:1 — if it's closer to 1:1, my entire storage choice later would change, so I'd like to confirm this with you."

Note the sentence pattern: **state the assumption first, declare its blast radius, then hand the confirmation back to the interviewer**. It advances the session while signaling judgment.

## 2. Each company's E5 signal checklist

Synthesized from Hello Interview's Meta E5 guide, System Design Handbook, and community interview reports, here are positive E5 signals (in the style of what interviewers write in their feedback):

**Navigation**
- "Drove requirements gathering without prompting" (led requirements collection unprompted)
- "Scoped the problem well — explicitly deferred Y to follow-ups" (good scope convergence)

**Design**
- "Design matched the stated scale — no over-engineering" (approach matched the scale, not over-designed)
- "Considered failure modes before I asked" (addressed failure modes without waiting for my prompt)

**Depth**
- "Went deeper than the standard answer on X" (went deeper than the standard answers on X)
- "Knew the real-world pitfalls of the tech they chose" (knew the real pitfalls of the technologies they picked)

**Communication**
- "Structured, easy to follow" (structured, easy to follow)
- "Took my hint and adjusted quickly" (caught the hint and adjusted fast)

**Negative signals (the biggest causes of failure)**:
- needing 3 hints from the interviewer before changing course
- the words "trade-off / cost / alternative" never once appearing in the whole session
- answering "I haven't studied that" more than 2 times during deep dives
- over-engineering: discussing multi-region active-active for a system with a million DAU

## 3. How to train each dimension

| Dimension | Practice method | Self-check standard |
|-----------|-----------------|---------------------|
| Driving the conversation | open a problem yourself (no walkthroughs), run the full flow on a whiteboard, timed at 45 minutes | recording review: how many times was I "waiting for the interviewer" |
| Depth | for each problem pick one subsystem and write a 1-page deep-dive note | can you talk about it for 10 minutes without notes |
| Trade-offs | for every decision add a 3-column table: approach / alternative / cost | at least 1 table per component |
| Operations | for every design answer: "it's 3 a.m. and it's down — what fires first in monitoring? How do I recover?" | an SLI checklist exists |
| Ambiguity | drill the "assumption-first" sentence pattern; memorize 20 common assumptions as templates | see the script templates in chapter 03 |

## 4. Timeline consensus

The North American community consensus (Reddit r/ExperiencedDevs, Hello Interview, interviewing.io blogs) on the E5 prep cycle: **4–6 weeks**, 1–2 hours per day, split roughly as:

- 30% of the time reading material (these notes, ByteByteGo, classic problem walkthroughs)
- **50% of the time doing problems + mocks** (the biggest lever, and the part most people skip)
- 20% of the time retrospecting (record your own mocks, review them for structure problems)

For the detailed schedule, see [10 · 6-Week Study Plan](10-study-plan.md).

## Next Module

→ [03 · Delivery Framework: Running the 45 Minutes](03-delivery-framework.md)
