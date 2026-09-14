# 01 · What Is the System Design Interview

> This chapter answers three questions: what this interview is **actually testing**, what **formats** exist, and what standard the interviewer **scores you against**.

## 1. Authoritative definitions

### Design Gurus (the Grokking series)

> "The system design interview is an open-ended technical interview: you are asked to design a large-scale distributed system from scratch based on requirements. There is no single correct answer — you are evaluated on your ability to **clarify requirements, handle ambiguity, and make key architecture decisions**."

This is the most standard, textbook definition. Note two key phrases: **open-ended** (which means the initiative is yours) and **no right or wrong** (which means the process is what gets scored).

### Hello Interview (a team of former FAANG interviewers, currently the best free resource)

They don't fuss over definitions — they hand you a scorecard directly. Their understanding of the System Design interview:

> A 45-minute collaborative design session in which the interviewer plays the role of a senior colleague / reviewer, watching how you turn a vague product requirement into a defensible technical architecture.

### System Design Primer (GitHub ~280k stars)

> "Learn how to design large-scale systems. Prep for the system design interview, and become a better engineer."

Its role is that of a master resource index: knowledge organized by topic (load balancing, consistency, CDN, caching, ...) with classic problems and reference answers attached. Good as a map, not as a route.

### System Design Handbook

Emphasizes an idea every senior engineer should burn into their brain:

> **There is no perfect design. Every choice has a cost.** Good design is not the "optimal solution" — it is the most reasonable compromise under the given constraints.

A piece of trivia: Meta internally calls its system design round the **"Pirate" round** (the interviewer group that runs it goes by the codename "Pirates"), and in recent years it has split off a **"Pirate X"** variant focused on API / product design (see [08 Company Style Guide](08-company-guides.md)).

## 2. The four interview formats

Per the Design Gurus taxonomy (90% of E5 backend interviews are the first type):

| Type | What it tests | Classic problems | Where it appears |
|------|---------------|------------------|------------------|
| **backend / distributed system design** | scalable architecture, data modeling, bottleneck analysis | short URL, Feed, ticketing system | the main battleground for backend/full-stack E5 |
| **API design** | REST semantics, resource modeling, versioning, pagination | design the Twitter API, payment webhook | a standalone round at some companies |
| **frontend system design** | component architecture, state management, rendering performance | design the Netflix player, a table component | frontend roles only |
| **OO design (OOD)** | class diagrams, design patterns, domain modeling | design a parking lot, a chess game | mostly at small and mid-size companies |

> Meta-specific warning: at E5 you may hit a **Product Architecture round** (API / data modeling / user flow focused) instead of a traditional System Design round (infrastructure / scalability focused). The rubric is the same, but the deep dive direction differs — I've seen too many people prepare as pure backend and then collide with Product Architecture.

## 3. The four-dimension rubric (Hello Interview Rubric)

This is currently the publicly available framework that comes closest to the internal scorecards at big tech — **four dimensions, percentage-weighted**:

### 3.1 Problem Navigation — 30%

- **Proactive clarification**: scale (QPS / DAU / data volume), read/write ratio, consistency requirements, latency budget, client types
- **Scope narrowing**: explicitly saying "I plan to build X first and defer Y to follow-ups" — what interviewers fear most is a candidate who wants to do everything
- **Aligning on definitions**: translating vague words into concrete features (how fast is "fast"? how is an "active user" defined?)

### 3.2 Solution Design — 30%

- A clear high-level architecture: you can draw the component diagram and explain the data flows
- Covers both core features and non-functional requirements (availability, scalability, latency)
- **The approach matches the requirements**: a system with 10k DAU should not open with sharding + multi-region DR (over-engineering is a penalty at E5)

### 3.3 Technical Depth & Differentiation — 20%

- Deep dive on at least one subsystem down to the implementation level: storage engine selection, index design, consistency protocols, failure modes
- Can articulate the trade-off: "I pick A, the cost is X, because in our scenario X is acceptable and Y is not"
- Shows understanding beyond the "standard answers": knows the real pitfalls of common approaches (e.g., Redis cache invalidation storms, Kafka partition skew)

### 3.4 Communication & Collaboration — 20%

- Structured delivery: top-down, always letting the interviewer know which layer you're on
- **Accepting steering**: when the interviewer corrects course, you adjust quickly instead of digging in (an interviewer's hint is a gift)
- Commanding the whiteboard: diagrams that others can actually read, draw-as-you-talk

> Note the signal carried by the 30/30/20/20 structure: **"running the right process" and "a sound approach" are 60 points — more important than how hardcore your tech is**. Many people fail this round not because their tech is weak, but because they lose control of the process — drawing diagrams without asking about requirements, or walking themselves into a dead end they can't escape.

## 4. What the interviewer is actually thinking while scoring

The rubric translated into the interviewer's inner monologue:

| Rubric item | Interviewer's inner monologue |
|-------------|-------------------------------|
| Proactive clarification | "I hadn't given any requirements yet, and they'd already asked everything that needed asking" |
| Scope narrowing | "They know what matters and what doesn't — like someone who has run production systems" |
| Approach matching requirements | "For this scale, this approach is just right — not over-built, not under-built" |
| Deep dive trade-offs | "They haven't just read ByteByteGo — they know the real pitfalls of these components" |
| Accepting steering | "I dropped one hint and they turned on a dime — easy to collaborate with" |

**The E5 dividing line**: at E4 you can pass even if the interviewer drags you through the whole thing; at E5 you must drive the entire session, with the interviewer merely reviewing your design.

## 5. Anatomy of a real 45-minute interview

```
0:00–2:00  Interviewer presents the problem (a deliberately vague one-liner)
2:00–7:00  You clarify requirements, estimate scale, define scope      ← Navigation
7:00–20:00 High-level design: component diagram + API + data model   ← Design
20:00–38:00 Deep dive on 2–3 subsystems (interviewer picks direction) ← Depth
38:00–43:00 Bottlenecks, monitoring, failure, scaling, wrap-up        ← Depth + wrap-up
43:00–45:00 Your questions
```

For the concrete playbook per phase, see [03 delivery framework](03-delivery-framework.md).

## Next Module

→ [02 · E5/Senior Expectations](02-e5-senior-expectations.md)
