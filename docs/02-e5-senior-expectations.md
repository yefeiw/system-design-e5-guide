# 02 · E5 / Senior Expectations

> 同一道题，E4 和 E5 的「过线答案」完全不同。本章讲清楚差在哪，以及每个 dimension 怎么练。

## 1. E5 vs E4：五个本质区别

### 1.1 driving the conversation：谁在开车

| | E4 | E5 |
|---|---|---|
| requirements clarification | interviewer 提示「要不要问问我 scale？」 | 你主动列出 5–8 个 clarifying questions，自己决定哪些假设成立 |
| 方向选择 | interviewer 说「我们聊聊 storage 吧」 | 你说「我认为最关键的 risk 在 fan-out，我想先深入这里」 |
| deep-dive spots | 被带到哪算哪 | 你提议 2–3 个 deep dive candidates，和 interviewer**negotiate**priority |

**一句话：E4 是被面试的，E5 是在做 design review，interviewer 是 reviewer。**

### 1.2 depth：至少一个 subsystem 打穿

E4 画对框图、组件名字叫得对，基本就能过。E5 的隐性 requirement：**至少一个 subsystem 深到能回答「data 具体怎么存、怎么保持一致、what if it breaks」**。

「打穿」的检验标准（以 cache 为例）：
- ❌ E4 level：「加一层 Redis cache 减轻 database 压力」
- ✅ E5 level：「write-heavy 所以用 write-through 会有 cold data 问题，我选 cache-aside + 短 TTL；invalidation storm 用 logical expiration fallback；hot key 加 local cache 二级；cache hit rate 要在 95%+ 才值得这个复杂度——这个 numbers 我会放进 monitoring」

### 1.3 trade-off awareness：每个决策都有 cost

E5 答案的 default 句式不是「我用 X」，而是：

> 「我打算用 X。alternative 是 Y 和 Z。选 X 是因为我们的 scenario 里 A 是硬约束，X 在 A 上最强；cost 是 B 变差，而 B 在我们这里是 soft constraint，可以靠 C mitigate。」

没有讲出 trade-off 的决策，在 E5 interviewer 眼里等于没有做过决策。

### 1.4 operational maturity：上线之后怎么活下来

E5 default 要主动覆盖（不用 interviewer 提醒）：
- **monitoring**：关键 SLI（latency P99、error rate、queue backlog、cache hit rate）
- **rate limiting degradation**：overload 时先丢什么（degradation 非 core feature、return stale data）
- **capacity planning**：single-machine capacity × redundancy 系数，扩容触发条件
- **failure 演练**：某个组件挂了 system 表现如何（SPOF、split-brain、data 丢失 window）

### 1.5 handling ambiguity

E4 把模糊当阻碍（「题目没说清楚」），E5 把模糊当杠杆：

> 「题目没说 read/write ratio，我假设 100:1——如果是 1:1 我后面整个 storage selection 会换，所以这点我想和你确认一下。」

注意这个句式：**先给假设、声明假设的影响面、再把确认权交给 interviewer**。既推进了进度，又 impression 了判断力。

## 2. 各家的 E5 signal checklist

综合 Hello Interview 的 Meta E5 指南、System Design Handbook 与社区面经，E5 正向 signal（interviewer write 进反馈的原文 style）：

**导航类**
- "Drove requirements gathering without prompting"（主动引导需求收集）
- "Scoped the problem well — explicitly deferred Y to follow-ups"（scope convergence 好）

**设计类**
- "Design matched the stated scale — no over-engineering"（approach 与 scale matching，不过度设计）
- "Considered failure modes before I asked"（没等我问就考虑了 failure 模式）

**depth 类**
- "Went deeper than the standard answer on X"（在 X 上比 standard answers 深）
- "Knew the real-world pitfalls of the tech they chose"（知道自己选的技术的真实坑点）

**沟通类**
- "Structured, easy to follow"（有 structure，容易跟上）
- "Took my hint and adjusted quickly"（接住提示快速调整）

**负向 signal（挂人重灾区）**：
- interviewer 给了 3 次 hint 才拐弯
- 全程没有出现过「trade-off / cost / alternative」这三个词
- deep dive 时答「这个没研究过」超过 2 次
- over-engineering：日活百万的 system 讨论了跨区域多活

## 3. 每个 dimension 怎么练

| dimension | practice method | 自检标准 |
|------|---------|---------|
| driving the conversation | 自己开题（不看任何 walkthrough），timed 45 分钟 whiteboard 走全流程 | recording review：有多少次是「我在等 interviewer」 |
| depth | 每道题选一个 subsystem，write 1 页 deep-dive 笔记 | 能不能不看资料讲 10 分钟 |
| trade-off | 给每个决策补一张 3 列表：approach / alternative / cost | 每个组件至少 1 张 |
| operations | 对每个设计回答：「半夜 3 点它挂了，monitoring 先报什么？我怎么恢复？」 | 有 SLI checklist |
| ambiguity | practice「假设驱动」句式，把 20 个常见假设背成 template | 见 03 章 script templates |

## 4. timeline consensus

北美社区（Reddit r/ExperiencedDevs、Hello Interview、interviewing.io blogs）对 E5 prep 周期的 consensus：**4–6 周**，每天 1–2 小时，其中：

- 30% 时间 read 材料（讲义、ByteByteGo、classic problems walkthrough）
- **50% 时间做题 + mock**（最大的杠杆，也是大多数人最省掉的部分）
- 20% 时间 retrospective（录自己的模拟，回看 structure 问题）

详细排期见 [10 · 6-Week Study Plan](10-study-plan.md)。

## Next Module

→ [03 · Delivery Framework: Running the 45 Minutes](03-delivery-framework.md)
