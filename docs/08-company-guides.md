# 08 · 公司风格指南

> 同一个「过」，每家公司的口味不同。临考前一周把这份读了，把你的表达微调成目标公司的风格。

## 1. Meta（含 Pirate / Pirate X 辨析）

### 轮次结构
- 系统设计轮内部代号 **"Pirate"**；E5 常见变体 **"Pirate X"**（Product Architecture）
- **必须问清楚你面到的是哪种**（ recruiter 会告诉你；不确定就当 Pirate X 准备，因为它更常见于 E5）

| | Pirate（System Design）| Pirate X（Product Architecture）|
|---|---|---|
| 重心 | 基础设施、扩展性、瓶颈 | API 设计、数据建模、产品功能 |
| 典型题 | 设计 WhatsApp 后端、News Feed、Top-K | 设计 Instagram Reels 的后端 API、设计 Marketplace |
| 深挖方向 | 存储选型、一致性、容量 | API 契约、表结构、用户流程边界 |
| 相同点 | **评分标准完全一样（四维 rubric）**，只是深挖地形不同 |

### Meta 风格要点（来自 Hello Interview 的 Meta 指南）
- Meta 面试官期待你**非常快地进入深挖**——高层设计压缩到 10 分钟以内，留 25 分钟给深度
- "Move fast" 文化映射到面试：**宁可边画边说，不要沉默地想**
- Meta 特别爱问**监控/指标**（"这个设计上线后你怎么知道它健不健康？"）——Pirate 轮收尾必被问，提前准备 3 个 SLI
- 喜欢具体的数字假设（Meta 内部文化是数据驱动），估算环节做扎实

## 2. Google

- 传统的 **Googleyness + 工程严谨**风格：比 Meta 更看重**方案的正确性与边界讨论**，节奏可以稍慢
- 爱问**谷歌特色的题**：搜索、YouTube、Maps、Google Docs（协作编辑/CRDT）
- 期待你主动讨论**多区域/全球部署**（Google 的默认世界观是行星尺度）
- 面试官可能在**具体技术上钻得很深**（一个数据结构、一个协议细节）——诚实 + 推理 > 装懂
- L5 对应 E5：同样要求你驱动讨论；Google 更能容忍你花 2 分钟想清楚再说（沉默思考不是扣分项，但要说"让我想 30 秒"）

## 3. Amazon

- **Leadership Principles 埋伏在每一轮**——系统设计轮尤其考 Customer Obsession / Dive Deep / Bias for Action
- 题面常带业务味（"设计 Amazon Fresh 库存系统"）——**先花 1 分钟把需求和客户体验讲清楚再动笔**，这本身就是 LP 信号
- Amazon 面试官爱问 **"你怎么处理故障/这个设计的运营成本"**（Dive Deep + Frugality）
- 单向书面反馈文化：**面试官不太和你互动**，别等 hint——你自己推进全程（这和 Meta 的快节奏异曲同工）
- 数据一切：估算做扎实，Amazon 明确把 "bar raiser" 标准写在公开文档里——深度不足是 E5 挂法的重灾区

## 4. 其他常面目标（速记）

| 公司 | 风格要点 |
|------|---------|
| **Apple** | 务实 + 隐私：设计里主动提数据最小化/端侧处理是加分项 |
| **Netflix** | 可用性 > 一致性的哲学；聊 chaos/resilience 会被记住 |
| **Uber / Lyft** | 地理 + 实时匹配高频；H3/GeoHash 必须熟 |
| **Stripe** | API 设计 + 正确性（金额、幂等 key、对账）——支付语义必须无懈可击 |
| **Airbnb** | 偏 product architecture，数据建模权重高 |
| **LinkedIn** | 社交图谱 + 搜索推荐，风格接近 Google |
| **TikTok/Bytedance** | 节奏快、题量大，可能 2 道题 45 分钟——框架熟练度要求最高 |

## 5. 通用临场校准法

开考 3 分钟内，从面试官的前两个反应判断口味，实时调整：

- 面试官**频繁打断追问细节** → 收窄：少画框、快进深挖（Meta/字节风格）
- 面试官**安静让你讲** → 保持框架完整 + 主动交代 trade-off（Google 风格）
- 面试官**问运营/成本/故障** → 收尾阶段加码运维章节（Amazon 风格）

**但注意：先按 03 章的标准框架走，微调是修饰不是重构。**

## 下一章

→ [09 · 资源地图](09-resources.md)
