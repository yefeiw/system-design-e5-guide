# System Design @ E5 — Senior Engineer 面试讲义

> 面向北美 E5 / Senior Engineer 职级的 System Design 面试完整 prep 讲义。
> 综合了 Hello Interview、ByteByteGo、System Design Primer、Design Gurus (Grokking)、System Design Handbook 等北美主流 resource 的 definitions、rubric 与 framework，并结合 FAANG interviewer 视角整理而成。

## 这套讲义是什么

一句话：**从「会做题」到「像 Senior 一样驱动一场设计讨论」**。

E5 的 System Design 面试不是考你背过几个 architecture 图，而是考你能否在一个 45 分钟的开放问题里，**主动 convergence 需求、提出 approach、深入关键 subsystem、并把每个决策的 trade-off 讲清楚**。这套讲义按五层递进组织：

```
认知（面试考什么）→ 基础（estimation/building blocks）→ 设计能力（classic problems deep dive）→ real-world systems → 表达与节奏
```

## 目录

| # | 模块 | 内容 | 时间投入 |
|---|------|------|---------|
| 01 | [What Is System Design](docs/01-what-is-system-design.md) | definitions、面试 types、four-dimension rubric | 1 天 |
| 02 | [E5/Senior 的 Expectations](docs/02-e5-senior-expectations.md) | E5 vs E4 本质区别、interviewer 在找什么 signal | 1 天 |
| 03 | [Delivery Framework: Running the 45 Minutes](docs/03-delivery-framework.md) | six-phase timeline + 每阶段 script templates | 2 天（练到肌肉记忆）|
| 04 | [Estimation & Numbers to Know](docs/04-estimation-numbers.md) | Back-of-envelope、Numbers to Know cheat sheet | 2 天 |
| 05 | [Building Blocks](docs/05-building-blocks.md) | LB / cache / database / sharding / consistency / message queue / CDN | 1–2 周 |
| 06 | [classic problems overview](docs/06-classic-questions-overview.md) | problem grading、E5 high-frequency checklist、practice method | — |
| Q1 | [URL shortener](docs/questions/01-url-shortener.md) | starter problem + ID generate deep dive | 3 小时 |
| Q2 | [rate limiter](docs/questions/02-rate-limiter.md) | algorithmic deep dives + distributed consistency | 3 小时 |
| Q3 | [Top-K / Heavy Hitters](docs/questions/03-top-k-heavy-hitters.md) | streaming algorithm + accuracy trade-off | 3 小时 |
| Q4 | [chat system](docs/questions/04-chat-system.md) | long-lived connection / message delivery semantics / group chat scaling | 3 小时 |
| Q5 | [News Feed](docs/questions/05-news-feed.md) | Push vs Pull / Fan-out / consistency | 3 小时 |
| Q6 | [distributed message queue](docs/questions/06-distributed-message-queue.md) | storage model / delivery guarantees / Kafka comparison | 3 小时 |
| Q7 | [Ticket Booking system](docs/questions/07-ticket-booking.md) | concurrency 控制 / strongly consistent / queueing | 3 小时 |
| Q8 | [Ad Click Aggregator](docs/questions/08-ad-click-aggregator.md) | 流处理 / Exactly-once / billing accuracy | 3 小时 |
| 07 | [real-world systems case studies](docs/07-real-systems-case-studies.md) | Netflix / Uber / classic papers breakdown | 持续 |
| 08 | [Company Style Guide](docs/08-company-guides.md) | Meta Pirate / Google / Amazon 差异 | 0.5 天 |
| 09 | [Resource Map](docs/09-resources.md) | websites / videos / books / papers，categorized by use | — |
| 10 | [6-Week Study Plan](docs/10-study-plan.md) | weekly plan + Mock 安排 | — |

## 使用方式

1. **第 1 周**：read 01–04，把 rubric 和 framework internalize。这不是知识，是「考试 rule」。
2. **第 2–5 周**：05 过一遍 building blocks，然后进入 06 + Q1–Q8 classic problems 训练。每道题**必须自己先做 45 分钟模拟**，再看讲义对照。
3. **全程**：每周至少 1–2 次 mock（真人优先，[Interviewing.io](https://interviewing.io) / 朋友 / AI 都行）。
4. **临考前 1 周**：过 08 company style + retrospective 所有 mock 的失败点。

## core 信念（来自所有主流 resource 的 consensus）

- **没有唯一正确答案**——评估的是你 trade-off 取舍的过程，不是 takeaway。
- **沟通占比 20%**——一半脑子设计，一半脑子讲清楚。
- **Depth > Breadth**——E5 要在至少一个 subsystem 深入到 data model / consistency / bottleneck 层面。
- **Mock 是最大杠杆**——看 100 个 videos 不如做 1 次被虐的 mock。

## License

MIT — 随便用，欢迎 PR 补充你被面到的题。
