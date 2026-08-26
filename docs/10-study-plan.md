# 10 · 6-Week Study Plan

> 按社区 consensus 的 E5 prep 周期（4–6 周，每天 1–2 小时）排布。core 原则：**50% 时间在输出（做题/mock/retrospective），输入只占 30%**。可以压缩到 4 周（每周强度 ×1.5），不建议低于 4 周。

## overview

```
周 1 认知与 framework rubric internalize + framework 练到肌肉记忆 + numbers 表
周 2 building blocks 与 starter problem building blocks 速过 + Q1–Q3（Easy 三连）
周 3 core classic problems Ⅰ Q4–Q5 + 第一次 live mock
周 4 core classic problems Ⅱ Q6–Q8 + 第二次 mock + case studies 启动
周 5 depth 与迁移 archetype induction + cold-solving 新题 + 第三次 mock
周 6 冲刺 company style calibration + 弱点专项 + retrospective 卡片
```

## 第 1 周 · 认知与 framework

**target**：知道考什么、怎么走场，还没碰具体题。

- [ ] read [01](01-what-is-system-design.md)、[02](02-e5-senior-expectations.md)——把 four-dimension rubric 抄在自己桌上
- [ ] read [03](03-delivery-framework.md)，用 1 道简单题（如"设计 Pastebin"）**只练流程**，不看 depth，recording
- [ ] 背 [04](04-estimation-numbers.md) 的三张表（latency / availability / single-machine capacity），每天口算 2 个 scenario
- [ ] 周末自检：不看资料说出 six-phase 时间预算 + 每阶段一句话 target

**weekly output**：一段 45 分钟 cold-solving recording + 一张自己手 write 的 rubric。

## 第 2 周 · building blocks 与 starter problem

- [ ] [05](05-building-blocks.md) 过一遍，每个 building blocks 做一张三列表（approach/alternative/cost）
- [ ] Q1 short URL、Q2 rate limiting、Q3 Top-K，**严格按 06 overview 的 5 步 practice 法**（cold-solving → 对照 → cheat sheet → 3 天后重做 → 讲给别人）
- [ ] 配套看 ByteByteGo 对应 building blocks 的动画（每集 ≤10 分钟，看完必须能复述）

**weekly output**：3 道题的 cheat sheet + 三张 building blocks trade-off 表。

## 第 3 周 · core classic problems Ⅰ + 第一次 mock

- [ ] Q4 聊天、Q5 Feed，同样 5 -step method
- [ ] **第一次 live mock**（interviewing.io 或朋友）——预期会被虐，重点 record：时间失控点、freezing up 点、被 hint 的次数
- [ ] Mock retrospective write 进 `mock-log.md`：每个失误归类（流程 / 知识 / depth）

**weekly output**：2 道题 + 第一份 mock retrospective。

## 第 4 周 · core classic problems Ⅱ + case studies 启动

- [ ] Q6 message queue、Q7 ticketing、Q8 click aggregation
- [ ] DDIA 第 5、6、9 章（每天 30–40 页）
- [ ] 每天 read 1 个案例（[07](07-real-systems-case-studies.md) 的 checklist），一行笔记进 `cases.md`
- [ ] 第二次 mock

**weekly output**：3 道题 + 7 条案例笔记 + 第二份 mock retrospective。

## 第 5 周 · depth 与迁移（E5 的关键一周）

**target**：从"会这 8 道题"变成"会所有题"。

- [ ] **archetype induction**（[06 overview](06-classic-questions-overview.md) §2）：对每道做过的题 annotate archetype，write 出该 archetype 的通用 deep dive 三件套
- [ ] **cold-solving 2 道没准备过的题**（Hello Interview 题库 random 挑，如 Distributed Cache、Dropbox、Typeahead Suggestion）——检验迁移能力，这是本周唯一 KPI
- [ ] 第三次 mock，这次重点观察自己有没有"驱动讨论"
- [ ] 弱点专项：从三份 mock retrospective 里找出现次数 ≥2 的问题，集中补

**weekly output**：archetype 对照表 + 2 道陌生题的完整 cold-solving recording。

## 第 6 周 · 冲刺

- [ ] read [08](08-company-guides.md)，针对 target 公司调整表达重心
- [ ] 每天中午 15 分钟"elevator pitch"practice：random 抽 1 题，60–90 秒讲完 core trade-off
- [ ] 过一遍所有 cheat sheet 和 mock-log，**只复习失败点，不学新东西**
- [ ] 面前 48 小时：不做新题、不看新 resource；只做一次轻松的全流程 walkthrough + 睡够

## 时间不够的 4 周压缩版

砍法：第 2 周并入第 1 周（building blocks 只看 05 章文字版，跳过 videos）；Q6 和 Q8 二选一；mock 从 3 次减到 2 次（**mock 不能再减**，它是 prep 的最大杠杆）。

## Mock Log template

```markdown
## Mock #N — 日期 / 平台 / 题目
- 总体感觉（1–5）：
- 时间控制：哪个阶段 timeout 了？
- 主动驱动了吗？（有几处是我提议方向？）
- 被 hint 的次数和内容：
- freezing up 瞬间（题干 + 我当时的处理）：
- 归类：□流程 □知识 □depth □沟通
- 下次要改的一件事（只选一件）：
```

## 每日节奏建议（在职版）

```
通勤 20 min：口算 practice / YouTube building blocks videos / 案例笔记
晚上 60–90 min：当前周的主任务（classic problems or mock）
周末 2–3 h：cold-solving + recording review + 重做
```

## 最后一页（打印贴墙上）

1. 先问需求再动笔——**需求 write 上 whiteboard**
2. 每个 numbers 后面跟一句"所以……"
3. 每个组件三段式：是什么 / 为什么 / what if it breaks
4. 主动提议 deep-dive spots，和 interviewer negotiate
5. 被纠偏 = 收到礼物，立刻调头
6. 留最后 5 分钟 wrap-up：bottleneck / monitoring / out-of-scope items
7. freezing up 就说出来："我在 A 和 B 之间犹豫……"
8. **你是在做 design review，不是在被审问。**

---
← [09 Resource Map](09-resources.md) ｜ [Back to TOC](../README.md)
