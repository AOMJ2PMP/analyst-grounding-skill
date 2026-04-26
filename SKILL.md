---
name: analyst-grounding
description: |
  从 Tencent Cloud 自有源（文档 / 云知 / 公众号）抓事实给 analyst-wording skill 用的 grounding skill。
  v0.1 只支持 cloud.tencent.com/document（公开文档，Firecrawl）。
  v0.2 加云知（Playwright 浏览器自动化）。v0.3 加公众号（WeRSS / RSSHub 发现 URL + Firecrawl 抓正文）。
  每条 evidence 都带 audit chain：[CITED] / [INFERRED] / [REVIEW: product] / [REVIEW: Kevin]。
  输出格式 = analyst-wording 的 Phase 2 evidence input 格式，handoff 干净。
  触发词：「找事实」「grounding」「查文档」「analyst evidence」「问卷取证」「帮我 ground」「fetch evidence」
---

# Analyst Grounding · 分析师问卷取证术

> 「每一句话都要能回查到 source 原文。」
>
> 写不出 source 的事实，宁可标 `[REVIEW: product]` 让人补，也不许编。

## 核心理念

1. **每条 fact 必须带 audit tag**。三种之一：`[CITED]`（直引）/ `[INFERRED]`（跨源推理）/ `[REVIEW: <reviewer>]`（待人确认）。无 tag 的 fact 不许出现在输出里。
2. **CITED 必须带原文 quote**。不能写"我从 X 文档抓的"——必须能逐字 grep 回 source。
3. **INFERRED 必须显式推理链**。"based on industry knowledge" 这类 hand-wave 等于编。至少 2 个 [CITED] 才能 [INFERRED]。
4. **REVIEW 只有 2 类 reviewer**：`product`（技术准确性）和 `Kevin`（战略 framing / 是否打动分析师）。其他流程（客户授权、PR 合规）走外部，不进 skill。
5. **冲突 surface，不 merge**。两个源说不同的数字 → 都列出 + 标 `[REVIEW: product]`。
6. **timestamp 强制**：每个 source 必须带 `Retrieved` 日期 + `Page-updated` / `Published` 日期（如可获取）。
7. **输出 = wording skill 的 input**。两个 skill 串联跑，evidence package 的 schema 必须对齐。

## 关键区分

捕捉的是 **fact + provenance**，不是 **claim**。
- ❌ "腾讯 CFW 行业领先" — 没 source、没 provenance、没 audit chain，废。
- ✅ "[CITED] '云防火墙支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态' / Source: cloud.tencent.com/document/product/1132/47570 / Quote: 原文片段..." — 可回查、可审计。

捕捉的是 **可验证的事实片段**，不是 **完整答案**。
- Grounding skill 不写答案，只产出 evidence package
- Wording skill 拿到 evidence package 后写答案
- 两个 skill 都要严格遵守自己的边界

---

## 执行流程

### Phase 0: 入口确认

收到用户输入后，判断场景：

| 输入 | 路径 |
|---|---|
| 粘贴了完整问卷题目 + 字数限制 | → Phase 1 |
| 只说"帮我查 X 产品的资料" | → Phase 0B 反问：是哪道问卷题在用这些资料？ |
| 想批量跑多道题 | → 提示一道一道做；evidence 容易 cross-contaminate |

#### Phase 0B: 模糊请求澄清

只追问：

1. **是哪道分析师问卷题？**（必须是有 deadline 的真题，不接受"准备一下产品介绍"这种泛需求）
2. **题目原文是什么？**（中英都要，按原表）
3. **wording skill 的 evidence schema 适用哪一条？**（如：产品能力 × 列举 → 需要 5 个 candidate feature；详见 wording skill 的 [evidence-by-type.md](../analyst-wording-skill/references/evidence-by-type.md)）

得到这些后进入 Phase 1。

---

### Phase 1: Query 拆解

详见 [references/query-decomposition.md](references/query-decomposition.md)。

**核心动作**：把题目反向映射到 Tencent Cloud 产品名候选池。

例：题目 "5 key features in Security capabilities" → 不要直接拿题目去搜，要先反向枚举产品名：

```
Security 候选产品池：
- 云防火墙 CFW
- WAF
- 主机安全 CWP
- 零信任接入
- 腾讯云风控 TCRS
- 数据安全治理 DSGC
- 云访问安全 CSC
- 密钥管理 KMS
- ...（团队再补充）
```

然后**对每个候选产品独立 grounding**——产品名是抓文档的入口词。

输出：候选产品池清单 + 每个的官方文档入口 URL（待 Phase 2 抓内容）。

---

### Phase 2: Source 抓取

详见 [references/source-access-mechanics.md](references/source-access-mechanics.md)。

**v0.1 范围**：

- ✅ `cloud.tencent.com/document/...`（中文文档，via Firecrawl）
- ✅ `www.tencentcloud.com/document/...`（国际版英文文档，via Firecrawl）
- ⏳ 云知（v0.2，via Playwright）
- ⏳ 公众号（v0.3，via WeRSS / RSSHub 发现 + Firecrawl 抓）

**v0.1 抓取规则**：

1. 对每个候选产品，先抓 product overview 页（`.../product/<id>` 或 `.../product/<id>/<overview-section>`）
2. 抓时记录 `Retrieved` 时间戳（now）+ 尝试从页面 metadata 抓 `Page-updated`（看 `<meta>` 或 footer）
3. 国际版优先（如果 EN 版本完整覆盖）；国内独有功能回落中文版
4. 同步抓子页（feature list、SLA、pricing）—— 通常 sitemap 直接给

抓回来的内容**不直接用**——要进 Phase 3 做 audit-tagged 抽取。

---

### Phase 3: Audit-tagged 抽取

详见 [references/audit-chain-format.md](references/audit-chain-format.md)。

把抓回来的 raw content 拆成三类 fact：

#### `[CITED]` — 直接 quote

某句话在 source 里有对应原文。格式：

```
[CITED] <claim>
  Source: <URL>
  Quote: "<原文片段，必须能 grep 到>"
  Retrieved: 2026-04-26
  Page-updated: 2025-11-03
```

要求：
- Quote 是 source 里的**字面原文**，不许 paraphrase
- Source 是中文 → quote 中文；source 是英文 → quote 英文
- Quote 长度 1-3 句，包含支撑 claim 的关键信息

#### `[INFERRED]` — 跨源推理

某 claim 不直接在任何一个 source 里，但能从 ≥2 个 [CITED] 推出。格式：

```
[INFERRED] <claim>
  Reasoning: <显式推理链>
  Based on:
    - [CITED] reference 1
    - [CITED] reference 2
```

要求：
- Reasoning 必须显式，不许 hand-wave
- 至少 2 个 [CITED] 项支撑
- 单源推论不算 INFERRED，应直接 [CITED]
- 推理跨度大或不显然 → 同时标 [REVIEW: Kevin]

#### `[REVIEW: <reviewer>]` — 需要人确认

两类 reviewer：

**`[REVIEW: product]`** — 技术 / 数据 / 占位符问题
```
[REVIEW: product] <claim 或 占位符>
  Reason: <为什么需要 product team review>
```
触发场景：占位数字（[X] B requests/day）/ 文档与公众号冲突的 SLA / 架构准确性存疑 / 不确定能不能这样描述产品。

**`[REVIEW: Kevin]`** — 战略 framing / 分析师评分透镜
```
[REVIEW: Kevin] <claim 或 framing 选择>
  Reason: <为什么需要 Kevin review>
```
触发场景：differentiation framing 选择 / 是不是分析师评分眼 / 跨多个 [CITED] 后的 INFERRED 推论强度不够 / "vs 竞品"的角度选择。

---

### Phase 4: 冲突检测 + 缺失项标记

详见 [references/source-priority.md](references/source-priority.md)。

**冲突检测**：

如果两个 source 对同一 fact 给出不同值，**不要 merge，全部 surface**：

```
Conflict on SLA number:

[CITED] "99.99% SLA"
  Source: cloud.tencent.com/document/product/cfw/sla
  Quote: "云防火墙服务可用性承诺达 99.99%..."
  Retrieved: 2026-04-26
  Page-updated: 2024-09-12

[CITED] "99.995% SLA"
  Source: mp.weixin.qq.com/s/...（公众号《云防火墙能力升级》）
  Quote: "新版本 SLA 提升至 99.995%..."
  Published: 2026-02-15
  Retrieved: 2026-04-26

[REVIEW: product] 当前对外 SLA 口径
  Reason: 公众号源比文档新 17 个月，可能反映文档未同步的升级。两个数字都需要 product 团队确认。
```

**缺失项标记**：

对照 wording skill 的 evidence schema，明确告诉用户哪些 input 没找到：

```
Required by wording skill but not found:

[REVIEW: product] 客户引用
  Reason: 公开文档未提供客户名 case study；云知未在 v0.1 范围内（v0.2 加）。
  
[REVIEW: product] 量化指标 [X] B requests/day
  Reason: 文档 SLA 页未披露请求量数据。
```

---

### Phase 5: 输出 evidence package

详见 [references/handoff-to-wording.md](references/handoff-to-wording.md)。

输出格式必须严格对齐 wording skill 的 Phase 2 input schema。

骨架：

```
[Evidence package]
Question: <原题>
Question type tag: <按 wording skill taxonomy>
Sources searched: cloud.tencent.com/document, www.tencentcloud.com/document
Generation date: 2026-04-26

Candidate pool（团队从中挑 top N）：

Candidate 1: <产品名>
  Description (draft):
    <技术描述，每个 sentence 后附 audit tag 引用>
  
  Provenance:
    [CITED] ...
    [CITED] ...
    [INFERRED] ...
    [REVIEW: product] ...
    [REVIEW: Kevin] ...

Candidate 2: ...
...

Conflicts detected: <列表>
Missing required input: <列表>

Hand-off note for wording skill:
  - 候选产品池 N 个，团队需从中挑 top <X>（不在 grounding skill 范围内）
  - 待 product review 的 N 项
  - 待 Kevin review 的 N 项
```

---

## 反模式（绝对不能做的事）

1. **不许出 fact 没有 audit tag**。每行 fact 必须以 `[CITED]` / `[INFERRED]` / `[REVIEW: ...]` 之一开头。
2. **不许 [CITED] 没有 Quote**。无原文 = 等于编。
3. **不许 [INFERRED] 单 source**。单 source 应是 [CITED]。如果觉得"推理"了，重新检查——大概率是改写而不是推理。
4. **不许 [REVIEW] 不指定 reviewer**。`[REVIEW]` 不行；必须 `[REVIEW: product]` 或 `[REVIEW: Kevin]`。
5. **不许 merge 冲突 source**。surface 所有版本 + 标 [REVIEW: product]。
6. **不许 emoji**。
7. **不许 markdown 链接**。URL 明文。
8. **不许 industry knowledge 兜底**。"based on what I know about cloud security" = 编。源里没有就说源里没有。
9. **不许预设答案**。grounding skill 只产出 evidence；写答案是 wording skill 的事。
10. **不许替团队挑 top N**。Phase 5 输出候选池，不输出 top N。

---

## Reference 文件

加载顺序：

1. [references/audit-chain-format.md](references/audit-chain-format.md) — `[CITED]` / `[INFERRED]` / `[REVIEW]` 格式规范（Phase 3）
2. [references/source-priority.md](references/source-priority.md) — 冲突解决 + freshness 规则（Phase 4）
3. [references/source-access-mechanics.md](references/source-access-mechanics.md) — 各 source 的访问方式 + v0.1 / v0.2 / v0.3 roadmap（Phase 2）
4. [references/query-decomposition.md](references/query-decomposition.md) — 题目 → 产品候选池 → search query 的拆解方法（Phase 1）
5. [references/handoff-to-wording.md](references/handoff-to-wording.md) — 输出 schema，对齐 wording skill input（Phase 5）

---

## 边界（skill 做不到的事）

1. **挑 top N feature 做不到**。这是 wording skill 已经声明的边界——grounding skill 同样不挑，只产出候选池。
2. **客户引用授权做不到**。NDA 客户能不能引用是外部审批流程，不在 skill 范围。
3. **未发布 / 未公开数据做不到**。公开文档 + 云知 + 公众号都没有的数据，标 [REVIEW: product]，不许编。
4. **冲突仲裁做不到**。两个 source 说不同的数字，skill 只 surface，不裁判。仲裁必须 [REVIEW: product]。
5. **战略 framing 取舍做不到**。多个角度都对的时候选哪个，必须 [REVIEW: Kevin]。

---

## 配套使用

这个 skill 设计成与 [analyst-wording](https://github.com/AOMJ2PMP/analyst-wording-skill) 串联跑：

```
原题 → analyst-grounding → evidence package → analyst-wording → CN draft → EN final
```

或者通过 wrapper skill「跑问卷」自动编排。两者独立 install，独立 iterate。
