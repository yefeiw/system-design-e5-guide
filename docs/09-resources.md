# 09 · Resource Map

> Categorized by use, not by fame. Principles: **free first, few but sharp, and every resource bound to a specific interview purpose**.

## 1. websites (used by prep phase)

### Tier 1 (core, all free)

| Resource | What It Is | Where to Use It |
|------|--------|--------|
| [Hello Interview](https://www.hellointerview.com) | A free guide from a team of former FAANG interviewers + a classic problems library + scoring rubrics | **Your main resource.** The Delivery Framework, the four-dimension rubric, classic problem walkthroughs tiered by difficulty (E5 high-frequency: Ticket Booking / Ad Click / Top-K / Rate Limiter / Distributed Cache), and a dedicated Meta E5 guide |
| [System Design Primer](https://github.com/donnemartin/system-design-primer) | A ~280k-star GitHub knowledge outline | A map for filling gaps: every building block (LB/cache/consistency/CDN) has an explanation plus a pros/cons table |
| [ByteByteGo](https://bytebytego.com) | Alex Xu's team: animated diagrams + a weekly newsletter | Building **intuition for building blocks**: the visualized principles behind every storage/queue/cache pattern |

### Tier 2 (targeted reinforcement)

| Resource | Use |
|------|------|
| [Design Gurus](https://designgurus.io) (the team behind Grokking) | Paid courses + a free blog; the source of the four interview categories (backend/API/frontend/OOD) |
| [interviewing.io](https://interviewing.io) | **Live mocks**, the best reputation in North America; the recording/playback mechanism is excellent |
| [Codemia](https://www.codemia.io) | Bills itself as LeetCode for system design, with AI scoring for practice |
| [Exponent](https://www.tryexponent.com) | Paid courses + a mock platform; consistent quality in its question bank and explanations |
| [Tech Interview Handbook](https://www.techinterviewhandbook.org) | A free overview, stronger on the behavioral and algorithm sides |

## 2. YouTube channels (for filling concept gaps, not the main path)

| Channel | Style | When to Watch It |
|------|------|-----------|
| **ByteByteGo / Alex Xu** | Animated diagrams, one concept in 10 minutes | Your first stop for learning building blocks (pairs with the websites) |
| **Gaurav Sen** | Enthusiastic explanation + interview-style walkthroughs | Early on, to build a feel for "what an interview looks like" |
| **Jordan Has No Life** | 2-hour hardcore deep dives | When you want to hear the 8th layer of detail on a problem |
| **Tech Dummies (Narendra L)** | Whiteboards a classic problem step by step | Draw along with Kafka/short URL/YouTube |
| **Hussein Nasser** | The low-level principles of databases/networks | **Not tested directly in interviews**, but it drills your foundations; listen on your commute |
| **codeKarle** | Real big-tech case study style | To fill in case studies |
| **sudo code** | Frontend system design | For frontend roles specifically |

> Community consensus (discussed repeatedly on Reddit's r/ExperiencedDevs): **YouTube can only build intuition, it can't replace hands-on practice.** Every minute you watch a video you should ask, "could I whiteboard this from memory?"

## 3. Books (read only three, in priority order)

| Book | What to Read | Priority |
|----|-----------|--------|
| **Alex Xu, System Design Interview Vol. 1** | All 12 chapters, one problem per chapter | The mainstay of E5 prep; read this first |
| **System Design Interview Vol. 2** | Pick problems you haven't seen + internally scaled company cases | After Vol. 1 |
| **DDIA (Designing Data-Intensive Applications)** | Chapter 5 (replication), 6 (partitioning), 9 (consistency and consensus) | Read only these three chapters; finishing the whole book is Staff+ homework and can be skipped when time is tight |

## 4. classic papers (see [07 case studies](07-real-systems-case-studies.md))

Dynamo → Bigtable → the Kafka papers → TAO → Raft (the visualized version at raft.github.io). 30–60 minutes each; extract only "what did they sacrifice."

## 5. Engineering Blogs

| Blog | High-Value Content |
|------|-----------|
| Netflix TechBlog | Elasticity, degradation, playback architecture |
| Uber Engineering | H3, marketplace, schemaless |
| Meta Engineering | TAO, memcache, cache consistency |
| Cloudflare Blog | Rate limiting (their sliding window implementation), edge network |
| High Scalability (aggregator) | A quick tour of various companies' architectures, 10 minutes a day |

## 6. resource × prep phase cross-reference

```
Week 1    All Hello Interview guides + chapters 01–04 of this guide      (awareness layer)
Week 2–3  ByteByteGo building block videos + Primer for gap-filling + Q1–Q4 classic problems      (building blocks layer)
Week 4–5  Q5–Q8 classic problems + the three DDIA chapters + interviewing.io mocks ×2      (depth layer)
Week 6    Run through both Alex Xu volumes + company styles (08) + retrospective      (wrap-up layer)
```

## 7. one-line principle

> More resources ≠ better prepared. **There's only one test: with no materials in front of you, can you whiteboard a brand-new problem for a full 40 minutes with trade-offs?** If not, go back to the classic problems and mocks instead of opening another resource.

## Next Module

→ [10 · 6-Week Study Plan](10-study-plan.md)
