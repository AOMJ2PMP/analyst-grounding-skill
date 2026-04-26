# Audit Chain 格式规范

> Grounding skill 输出的每一条 evidence 必须带 audit tag。下游 wording skill 和 reviewer（产品专家 / Kevin）都依赖这个格式做 audit。

## 三种 tag

### `[CITED]` — 直接引用

某条 fact 在 source 里有原文支撑，可以逐字回查。

**格式**：

```
[CITED] <claim 一句话>
  Source: <URL>
  Quote: "<source 里的原文片段>"
  Retrieved: YYYY-MM-DD
  Page-updated: YYYY-MM-DD       (如果可获取)
  Published: YYYY-MM-DD           (如果是公众号 / blog 类源)
```

**要求**：

- Quote 必须是 source 里的**字面原文**，不许 paraphrase
- 如果 source 是中文，quote 是中文；source 是英文，quote 是英文
- Quote 长度 1-3 句话，包含支撑 claim 的关键信息
- 如果 source 页面没有 last-updated metadata，留空 Page-updated 字段（不要瞎填）
- 公众号必带 Published；文档必带 Page-updated（如果页面有）

**禁止**：

- claim 没有对应 quote
- quote 是 paraphrase 不是原文（哪怕意思一样）
- 没有 Retrieved 时间戳
- Quote 跨多个段落拼接（容易丢上下文，应该分成多个 [CITED]）

---

### `[INFERRED]` — 跨源推理

某条 fact 不直接来自任何一个 source，但可以从 2+ cited fact 逻辑推出。

**格式**：

```
[INFERRED] <claim 一句话>
  Reasoning: <显式推理链条>
  Based on:
    - [CITED] reference 1（重复源信息或直接引用上文 cited 项）
    - [CITED] reference 2
```

**要求**：

- Reasoning 必须显式，不许 hand-wave
- 至少基于 2 个 [CITED] 项（单 source 应是 [CITED]）
- 推理是逻辑组合，不是 leap of faith
- 如果推理跨度大或不显然，**同时标 `[REVIEW: Kevin]` 让 AR 专家校核 framing**

**禁止**：

- 单 source 推理（直接 [CITED]）
- "based on industry knowledge" / "common in the industry" 类无源推理
- 推理链条说不清楚（如 "combining A and B → C" 但没说怎么 combine）
- 把改写当推理（"文档说 X，所以 Y" 当 Y 实际就是 X 的同义改写）

---

### `[REVIEW: <reviewer>]` — 待 review

某条 fact 不能从 source 取得 / 有冲突 / 涉及战略判断。

**两类 reviewer**：

#### `[REVIEW: product]` — 技术 / 数据准确性

格式：

```
[REVIEW: product] <claim 或 占位符>
  Reason: <为什么需要 product team review>
```

触发场景：

- 占位数字未填（如 `[X] B requests/day`）
- 多个 source 给出不同数字（SLA、规模、价格等）
- 架构描述准确性存疑（如"是否真的是控制平面统一"）
- 是否能这样描述产品（PRD 表述 vs 对外口径）
- 文档没找到但 product team 应该知道的

#### `[REVIEW: Kevin]` — 战略 framing / 分析师评分透镜

Kevin = AR 专家 / ex-Gartner VP analyst。判断"分析师会不会买账"。

格式：

```
[REVIEW: Kevin] <claim 或 framing 选择>
  Reason: <为什么需要 Kevin review>
```

触发场景：

- 多个 differentiation 角度都对，选哪个最打动分析师（如"unified IAM" vs "real-time threat sharing" 哪个更值得作为 5 features 之一）
- INFERRED 推论强度不够，需要 AR 视角校核
- Vision / Strategy 题里的 thesis 选择
- "vs 竞品"题里的 framing 选择
- 是不是符合该机构当年的评分透镜（Gartner 今年权重在哪？Forrester 这一版关心什么？）

**禁止**：

- 技术问题标 Kevin（应该 product）
- 战略问题标 product（应该 Kevin）
- 不分人，"反正都需要 review" 一锅端

---

## 完整示例

Question: "What are 5 key features in your Security capabilities that distinguish your value proposition from competitors?"

Candidate 1: Cloud Firewall (CFW)

Description (draft):

腾讯云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态。威胁情报跨客户群 5 分钟级同步，新发现的恶意 IP 在被任一租户首次识别后数分钟内即可被全网封禁。日均拦截恶意请求 [X] B 次，三层策略统一管理。

Provenance:

```
[CITED] CFW 支持三层部署形态
  Source: cloud.tencent.com/document/product/1132/47570
  Quote: "云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态，统一管理出入站流量与跨 VPC 流量..."
  Retrieved: 2026-04-26
  Page-updated: 2025-11-03

[CITED] 威胁情报 5 分钟级同步
  Source: mp.weixin.qq.com/s/abc123（公众号《云防火墙能力升级》）
  Quote: "新版本将威胁情报同步窗口从 30 分钟缩短至 5 分钟，全平台租户共享..."
  Published: 2026-02-15
  Retrieved: 2026-04-26

[INFERRED] 新发现的恶意 IP 在数分钟内即可被全网封禁
  Reasoning: 5 分钟同步窗口 + 三层覆盖（VPC + internet + east-west）= 网络范围内 ≤ 5 分钟可达
  Based on:
    - [CITED] CFW 支持三层部署形态
    - [CITED] 威胁情报 5 分钟级同步

[REVIEW: product] [X] B 恶意请求/天
  Reason: 占位符。公开文档未披露请求量数据；v0.1 范围未覆盖云知。需要 product 团队提供对外可披露的量级。

[REVIEW: Kevin] Framing: "single-pane policy across all three tiers" 是否作为 differentiation 的关键卖点
  Reason: 候选 framing 还有 "real-time threat intel sharing"。两个都符合 source 事实，但 Lydia Leong 在 cloudpundit 上多次提到 "unified control plane" 是她的评分透镜重点。Kevin 判断哪个更值得放在 5 features 里。
```

---

## 输出格式 hard rules

1. 每条 evidence 行首必须是 `[CITED]` / `[INFERRED]` / `[REVIEW: product]` / `[REVIEW: Kevin]` 之一
2. 不用 emoji
3. 不用 markdown bold / italic / heading
4. URL 明文，不要包成 markdown 链接 `[text](url)`
5. Quote 必须双引号包起来
6. Source 是 URL 不是描述（"公众号文章 abc" 不行，必须给完整 mp.weixin.qq.com URL）
7. 时间戳格式 `YYYY-MM-DD`
8. 字段顺序：Source → Quote → Retrieved → Page-updated/Published（按这个顺序，便于 grep）

## 输出后自检 checklist

```
[ ] 每条 fact 都有 audit tag（grep 验证：每行不是空行/标题就应以 [ 开头）
[ ] 所有 [CITED] 都有 Quote
[ ] 所有 [INFERRED] 都至少 2 个 Based on
[ ] 所有 [REVIEW] 都指定了 reviewer (product / Kevin)
[ ] 所有 source 都是 URL，不是描述
[ ] 所有 timestamp 是 YYYY-MM-DD 格式
[ ] 没有 emoji
[ ] 没有 markdown link
```
