# 08 · Company Style Guide

> 同一个「过」，每家公司的口味不同。临考前一周把这份 read 了，把你的表达微调成 target 公司的 style。

## 1. Meta（含 Pirate / Pirate X 辨析）

### round structure
- system 设计轮内部代号 **"Pirate"**；E5 常见 variant **"Pirate X"**（Product Architecture）
- **必须问清楚你面到的是哪种**（recruiter 会告诉你；不确定就当 Pirate X 准备，因为它更常见于 E5）

| | Pirate（System Design）| Pirate X（Product Architecture）|
|---|---|---|
| 重心 | 基础设施、scalability、bottleneck | API 设计、data modeling、产品 feature |
| classic problems | 设计 WhatsApp backend、News Feed、Top-K | 设计 Instagram Reels 的 backend API、设计 Marketplace |
| deep dive 方向 | storage selection、consistency、capacity | API 契约、table schema、user flow boundary |
| 相同点 | **rubric 完全一样（four-dimension rubric）**，只是 deep dive 地形不同 |

### Meta style essentials（来自 Hello Interview 的 Meta 指南）
- Meta interviewer 期待你**非常快地进入 deep dive**——high-level design 压缩到 10 分钟以内，留 25 分钟给 depth
- "Move fast" 文化映射到面试：**宁可边画边说，不要沉默地想**
- Meta 特别爱问**monitoring/指标**（"这个设计上线后你怎么知道它健不健康？"）——Pirate 轮 wrap-up 必被问，提前准备 3 个 SLI
- 喜欢具体的 numbers 假设（Meta 内部文化是 data-driven），estimation 环节做扎实

## 2. Google

- 传统的 **Googleyness + 工程严谨**style：比 Meta 更看重**approach 的 correctness 与 boundary 讨论**，节奏可以稍慢
- 爱问**谷歌特色的题**：search、YouTube、Maps、Google Docs（协作编辑/CRDT）
- 期待你主动讨论**多区域/全球 deployment**（Google 的 default 世界观是行星尺度）
- interviewer 可能在**具体技术上钻得很深**（一个 data structure、一个协议细节）——诚实 + 推理 > 装懂
- L5 对应 E5：同样 requirement 你驱动讨论；Google 更能容忍你花 2 分钟想清楚再说（沉默思考不是扣分项，但要说"让我想 30 秒"）

## 3. Amazon

- **Leadership Principles 埋伏在每一轮**——system 设计轮尤其考 Customer Obsession / Dive Deep / Bias for Action
- 题面常带 business 味（"设计 Amazon Fresh inventory system"）——**先花 1 分钟把需求和客户体验讲清楚再动笔**，这本身就是 LP signal
- Amazon interviewer 爱问 **"你怎么处理 failure/这个设计的 ops cost"**（Dive Deep + Frugality）
- one-way 书面反馈文化：**interviewer 不太和你互动**，别等 hint——你自己推进全程（这和 Meta 的快节奏异曲同工）
- data 一切：estimation 做扎实，Amazon 明确把 "bar raiser" 标准 write 在公开文档里——depth 不足是 E5 挂法的重灾区

## 4. 其他常面 target（速记）

| 公司 | style essentials |
|------|---------|
| **Apple** | 务实 + 隐私：设计里主动提 data 最小化/端侧处理是 bonus signal |
| **Netflix** | availability > consistency 的哲学；聊 chaos/resilience 会被记住 |
| **Uber / Lyft** | 地理 + real-time matching high-frequency；H3/GeoHash 必须熟 |
| **Stripe** | API 设计 + correctness（金额、idempotent key、reconciliation）——payment semantics 必须无懈可击 |
| **Airbnb** | 偏 product architecture，data modeling 权重高 |
| **LinkedIn** | social graph + search 推荐，style 接近 Google |
| **TikTok/Bytedance** | 节奏快、题量大，可能 2 道题 45 分钟——framework 熟练度 requirement 最高 |

## 5. 通用 in-the-moment calibration 法

开考 3 分钟内，从 interviewer 的前两个反应判断口味，real-time 调整：

- interviewer**频繁打断追问细节** → 收窄：少画框、快进 deep dive（Meta/字节 style）
- interviewer**安静让你讲** → 保持 framework 完整 + 主动交代 trade-off（Google style）
- interviewer**问 ops/cost/failure** → wrap-up 阶段加码 operations 章节（Amazon style）

**但注意：先按 03 章的标准 framework 走，微调是修饰不是重构。**

## Next Module

→ [09 · Resource Map](09-resources.md)
