# System Design @ E5 — Senior Engineer Interview Lecture Series

> A complete prep lecture series for the System Design interview at the North American E5 / Senior Engineer level.
> It synthesizes the definitions, rubrics, and frameworks from mainstream North American resources — Hello Interview, ByteByteGo, System Design Primer, Design Gurus (Grokking), System Design Handbook — organized through the lens of a FAANG interviewer.

## What This Series Is

In one sentence: **going from "solving problems" to "driving a design discussion like a Senior."**

The E5 System Design interview is not testing whether you have memorized a few architecture diagrams. It tests whether, in a 45-minute open-ended problem, you can **proactively converge the requirements, propose an approach, go deep on the key subsystems, and articulate the trade-off behind every decision**. This series is organized as five progressive layers:

```
Cognition (what the interview tests) → Fundamentals (estimation / building blocks) → Design Skill (classic problem deep dives) → Real-World Systems → Communication and Pacing
```

## Table of Contents

| # | Module | Content | Time Investment |
|---|--------|---------|-----------------|
| 01 | [What Is System Design](docs/01-what-is-system-design.md) | definitions, interview types, four-dimension rubric | 1 day |
| 02 | [E5/Senior Expectations](docs/02-e5-senior-expectations.md) | the essential E5 vs E4 differences, what signal interviewers look for | 1 day |
| 03 | [Delivery Framework: Running the 45 Minutes](docs/03-delivery-framework.md) | six-phase timeline + script templates per phase | 2 days (drill until it's muscle memory) |
| 04 | [Estimation & Numbers to Know](docs/04-estimation-numbers.md) | back-of-envelope, Numbers to Know cheat sheet | 2 days |
| 05 | [Building Blocks](docs/05-building-blocks.md) | LB / cache / database / sharding / consistency / message queue / CDN | 1–2 weeks |
| 06 | [Classic Problems Overview](docs/06-classic-questions-overview.md) | problem grading, E5 high-frequency checklist, practice method | — |
| Q1 | [URL Shortener](docs/questions/01-url-shortener.md) | starter problem + ID generation deep dive | 3 hours |
| Q2 | [Rate Limiter](docs/questions/02-rate-limiter.md) | algorithmic deep dives + distributed consistency | 3 hours |
| Q3 | [Top-K / Heavy Hitters](docs/questions/03-top-k-heavy-hitters.md) | streaming algorithms + accuracy trade-offs | 3 hours |
| Q4 | [Chat System](docs/questions/04-chat-system.md) | real-time implementation: transport choice (MQTT/WebSocket) / connection lifecycle & session resume / delivery paths for the three tick types / ephemeral signals / E2EE | 3 hours |
| Q5 | [News Feed](docs/questions/05-news-feed.md) | Push vs Pull / Fan-out / consistency | 3 hours |
| Q6 | [Distributed Message Queue](docs/questions/06-distributed-message-queue.md) | storage model / delivery guarantees / Kafka comparison | 3 hours |
| Q7 | [Ticket Booking System](docs/questions/07-ticket-booking.md) | concurrency control / strong consistency / queueing | 3 hours |
| Q8 | [Ad Click Aggregator](docs/questions/08-ad-click-aggregator.md) | stream processing / Exactly-once / billing accuracy | 3 hours |
| 07 | [Real-World System Case Studies](docs/07-real-systems-case-studies.md) | Netflix / Uber / classic paper breakdowns | ongoing |
| 08 | [Company Style Guide](docs/08-company-guides.md) | Meta Pirate / Google / Amazon differences | 0.5 day |
| 09 | [Resource Map](docs/09-resources.md) | websites / videos / books / papers, categorized by use | — |
| 10 | [6-Week Study Plan](docs/10-study-plan.md) | weekly plan + mock schedule | — |

## How to Use This

1. **Week 1**: read 01–04 and internalize the rubric and the framework. This is not knowledge; these are the exam rules.
2. **Weeks 2–5**: go through the building blocks in 05, then move into 06 + the Q1–Q8 classic problems. For every problem, **you must run a 45-minute mock yourself first**, then compare against the lecture notes.
3. **Throughout**: at least 1–2 mocks per week (live humans preferred — [Interviewing.io](https://interviewing.io) / friends / AI all work).
4. **Final week before the interview**: review 08 company styles + retrospect on the failure points from all your mocks.

## Core Beliefs (the consensus across all mainstream resources)

- **There is no single correct answer** — you are evaluated on the process of weighing trade-offs, not on the takeaway.
- **Communication is 20% of the score** — half your brain designs, half your brain explains.
- **Depth > Breadth** — E5 requires going deep on at least one subsystem down to the data model / consistency / bottleneck level.
- **Mocks are the biggest lever** — watching 100 videos is worth less than 1 mock that beats you up.

## License

MIT — use it freely, and PRs adding questions you were asked are welcome.
