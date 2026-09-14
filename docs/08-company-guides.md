# 08 · Company Style Guide

> Everybody wants the same "pass," but every company has different taste. Read this in the week before your interview and fine-tune your delivery to the target company's style.

## 1. Meta (Pirate vs. Pirate X)

### round structure
- The system design round's internal codename is **"Pirate"**; the common E5 variant is **"Pirate X"** (Product Architecture)
- **You must confirm which one you're getting** (your recruiter will tell you; if you're unsure, prepare for Pirate X, since it's more common at E5)

| | Pirate (System Design) | Pirate X (Product Architecture) |
|---|---|---|
| Focus | Infrastructure, scalability, bottlenecks | API design, data modeling, product features |
| Classic problems | Design the WhatsApp backend, News Feed, Top-K | Design the backend API for Instagram Reels, design Marketplace |
| Deep dive direction | Storage selection, consistency, capacity | API contracts, table schemas, user flow boundaries |
| In common | **The rubric is identical (four-dimension rubric)**; only the deep dive terrain differs |

### Meta style essentials (from Hello Interview's Meta guide)
- Meta interviewers expect you to **get into the deep dive very fast** — compress the high-level design to under 10 minutes and leave 25 minutes for depth
- "Move fast" culture mapped onto the interview: **rather talk while drawing than think in silence**
- Meta especially loves **monitoring/metrics** questions ("once this design ships, how do you know it's healthy?") — guaranteed in the Pirate wrap-up, so prepare three SLIs in advance
- They like concrete numeric assumptions (Meta's internal culture is data-driven), so make the estimation section solid

## 2. Google

- The traditional **Googleyness + engineering rigor** style: compared with Meta, it weighs **the correctness of your approach and your discussion of edge cases** more, and the pace can be a little slower
- Loves **Google-flavored problems**: search, YouTube, Maps, Google Docs (collaborative editing/CRDT)
- Expects you to proactively discuss **multi-region/global deployment** (Google's default worldview is planetary scale)
- The interviewer may **drill very deep into a specific technology** (one data structure, one protocol detail) — honesty + reasoning > pretending to know
- L5 maps to E5: you drive the discussion at the same bar; Google is more tolerant of you taking 2 minutes to think before speaking (silent thinking isn't a penalty, but say "let me think for 30 seconds")

## 3. Amazon

- **Leadership Principles are buried in every round** — the system design round especially tests Customer Obsession / Dive Deep / Bias for Action
- Prompts often have a business flavor ("design Amazon Fresh's inventory system") — **spend the first minute nailing down the requirements and the customer experience before you start drawing**; that itself is an LP signal
- Amazon interviewers love asking **"how do you handle failure / what's the ops cost of this design?"** (Dive Deep + Frugality)
- One-way written feedback culture: **the interviewer won't interact with you much**, so don't wait for hints — drive the whole session yourself (similar in spirit to Meta's fast pace)
- Everything is data: make estimation solid; Amazon explicitly writes the "bar raiser" standard in public docs — insufficient depth is the biggest failure mode for E5

## 4. Other common targets (quick reference)

| Company | Style Essentials |
|------|---------|
| **Apple** | Pragmatic + privacy: proactively raising data minimization/on-device processing is a bonus signal |
| **Netflix** | The availability > consistency philosophy; talking about chaos/resilience gets remembered |
| **Uber / Lyft** | Geography + real-time matching are high-frequency; H3/GeoHash must be second nature |
| **Stripe** | API design + correctness (amounts, idempotency keys, reconciliation) — payment semantics must be flawless |
| **Airbnb** | Leans product architecture, with heavy weight on data modeling |
| **LinkedIn** | Social graph + search/recommendation, stylistically close to Google |
| **TikTok/Bytedance** | Fast pace, high volume, possibly 2 problems in 45 minutes — the highest demand on framework fluency |

## 5. The universal in-the-moment calibration method

Within the first 3 minutes, read the interviewer's taste from their first two reactions and adjust in real time:

- The interviewer **interrupts often to push on details** → narrow down: draw fewer boxes, jump to the deep dive (Meta/ByteDance style)
- The interviewer **stays quiet and lets you talk** → keep the framework complete + proactively lay out trade-offs (Google style)
- The interviewer **asks about ops/cost/failure** → load up the operations section during wrap-up (Amazon style)

**But note: follow the standard framework from chapter 03 first; fine-tuning is decoration, not a rewrite.**

## Next Module

→ [09 · Resource Map](09-resources.md)
