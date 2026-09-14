# 10 · 6-Week Study Plan

> Laid out according to the community-consensus E5 prep cycle (4–6 weeks, 1–2 hours a day). Core principle: **50% of your time goes to output (problems/mocks/retrospectives), while input takes only 30%**. You can compress it to 4 weeks (multiply weekly intensity by 1.5), but going below 4 weeks isn't recommended.

## overview

```
Week 1    Awareness and framework      internalize the rubric + drill the framework to muscle memory + the numbers tables
Week 2    Building blocks and starter problems      quick pass through building blocks + Q1–Q3 (three easy ones in a row)
Week 3    Core classic problems I      Q4–Q5 + first live mock
Week 4    Core classic problems II      Q6–Q8 + second mock + kick off case studies
Week 5    Depth and transfer      archetype induction + cold-solving new problems + third mock
Week 6    Sprint      company style calibration + targeted weak spots + retrospective cards
```

## Week 1 · Awareness and framework

**Target**: know what's being tested and how the session runs, without having touched a specific problem yet.

- [ ] Read [01](01-what-is-system-design.md) and [02](02-e5-senior-expectations.md) — copy the four-dimension rubric onto your desk
- [ ] Read [03](03-delivery-framework.md), then run 1 easy problem (e.g. "design Pastebin") **practicing the process only**, ignoring depth; record it
- [ ] Memorize the three tables in [04](04-estimation-numbers.md) (latency / availability / single-machine capacity) and do 2 scenarios of mental math a day
- [ ] Weekend self-check: without looking at any material, state the six-phase time budget + a one-sentence target for each phase

**Weekly output**: one 45-minute cold-solving recording + one rubric you wrote by hand.

## Week 2 · Building blocks and starter problems

- [ ] Run through [05](05-building-blocks.md) once, making a three-column table (approach/alternative/cost) for every building block
- [ ] Q1 short URL, Q2 rate limiting, Q3 Top-K, **strictly following the 5-step practice method in the 06 overview** (cold-solving → compare → cheat sheet → redo 3 days later → explain it to someone)
- [ ] Watch the corresponding ByteByteGo animations for each building block (≤10 minutes per episode; you must be able to restate it afterward)

**Weekly output**: cheat sheets for 3 problems + three building-block trade-off tables.

## Week 3 · Core classic problems I + first mock

- [ ] Q4 chat, Q5 Feed, same 5-step method
- [ ] **Your first live mock** (interviewing.io or a friend) — expect to get beaten up; focus on recording: where you lost control of time, where you froze up, how many hints you needed
- [ ] Write the mock retrospective into `mock-log.md`: categorize every mistake (process / knowledge / depth)

**Weekly output**: 2 problems + your first mock retrospective.

## Week 4 · Core classic problems II + kick off case studies

- [ ] Q6 message queue, Q7 ticketing, Q8 click aggregation
- [ ] DDIA chapters 5, 6, and 9 (30–40 pages a day)
- [ ] Read 1 case a day (the checklist in [07](07-real-systems-case-studies.md)), one-line note into `cases.md`
- [ ] Second mock

**Weekly output**: 3 problems + 7 case notes + a second mock retrospective.

## Week 5 · Depth and transfer (the decisive week for E5)

**Target**: go from "I know these 8 problems" to "I know every problem."

- [ ] **Archetype induction** ([06 overview](06-classic-questions-overview.md) §2): annotate the archetype of every problem you've done, and write out that archetype's universal deep dive trio
- [ ] **Cold-solve 2 problems you haven't prepared** (pick randomly from the Hello Interview bank, e.g. Distributed Cache, Dropbox, Typeahead Suggestion) — this tests transfer ability and is the only KPI for the week
- [ ] Third mock, this time watching specifically for whether you "drove the discussion"
- [ ] Targeted weak spots: find the issues that appear ≥2 times across the three mock retrospectives and fix them in a focused block

**Weekly output**: an archetype cross-reference table + full cold-solving recordings for 2 unfamiliar problems.

## Week 6 · Sprint

- [ ] Read [08](08-company-guides.md) and shift your emphasis to match the target company
- [ ] 15 minutes of "elevator pitch" practice every day at noon: pull 1 random problem and cover its core trade-offs in 60–90 seconds
- [ ] Run through all your cheat sheets and the mock log, **reviewing only failure points, learning nothing new**
- [ ] Final 48 hours: no new problems, no new resources; just one relaxed full walkthrough + enough sleep

## The 4-week compressed version when time is short

What to cut: merge week 2 into week 1 (read only the text version of chapter 05 for building blocks, skip the videos); pick either Q6 or Q8; reduce mocks from 3 to 2 (**mocks cannot be cut any further** — they're the single biggest lever in prep).

## Mock Log template

```markdown
## Mock #N — date / platform / problem
- Overall feel (1–5):
- Time control: which phase timed out?
- Did I drive proactively? (how many times did I propose the direction?)
- Number and content of hints received:
- Freezing-up moments (prompt + how I handled it at the time):
- Category: □ process □ knowledge □ depth □ communication
- One thing to change next time (pick only one):
```

## Suggested daily rhythm (working professional edition)

```
Commute 20 min: mental math practice / YouTube building block videos / case notes
Evening 60–90 min: the current week's main task (classic problems or a mock)
Weekend 2–3 h: cold-solving + recording review + redos
```

## The last page (print it and tape it to the wall)

1. Ask about requirements before you draw — **write the requirements on the whiteboard**
2. Follow every number with a sentence that starts with "so..."
3. Every component in three parts: what it is / why / what if it breaks
4. Proactively propose deep-dive spots and negotiate with the interviewer
5. Being corrected = receiving a gift; change course immediately
6. Save the last 5 minutes for wrap-up: bottleneck / monitoring / out-of-scope items
7. If you freeze up, say it out loud: "I'm torn between A and B..."
8. **You're running a design review, not being interrogated.**

---

← [09 Resource Map](09-resources.md) | [Back to TOC](../README.md)
