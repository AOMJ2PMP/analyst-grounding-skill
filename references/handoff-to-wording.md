# Handoff to Wording Skill

> Skill 在 Phase 5 使用。输出格式必须严格对齐 [analyst-wording](https://github.com/AOMJ2PMP/analyst-wording-skill) Phase 2 的 evidence input schema。

## 总骨架

```
[Evidence Package]

Question source: <机构 + 报告名 + 题号>
Question (original):
<原题完整文本，中英都贴>
Question type tag (per wording skill taxonomy):
  - 大类: <产品能力 / Strategy>
  - 形式: <简答 / 列举 / 是非 / 选择 / 数量 / 对比 / 时间锚定>
  - 修饰符: <量化, 客户引用, URL, 近 12 个月, 诚实模式, NDA 模式>
Char limit: <如 Forrester EN 1500 char / Gartner Description 500 char × 5>
Generation date: YYYY-MM-DD
Sources searched: <列表>
Sources NOT searched (out of v0.x scope): <列表>

---

Candidate pool (团队从中挑 top N):

Candidate 1: <产品名 / feature 名>
  
  Description (draft):
    <技术描述，每个 sentence 都有对应 audit tag 在下方>
  
  Provenance:
    [CITED] ...
    [CITED] ...
    [INFERRED] ...
    [REVIEW: product] ...
    [REVIEW: Kevin] ...

Candidate 2: ...

Candidate 3: ...

...

---

Conflicts detected:

<冲突列表，每个含两个 [CITED] + 一个 [REVIEW: product]>

---

Missing required input (per wording skill schema):

[REVIEW: product] <什么没找到>
  Reason: ...
  Suggested next step: ...

[REVIEW: product] ...

---

Hand-off note for wording skill:

  - 候选产品池 N 个，团队需从中挑 top <X> 进入 wording Phase 3
  - 待 product team review: N 项（详见 Provenance 列表）
  - 待 Kevin (AR) review: N 项（详见 Provenance 列表）
  - Conflicts: N 项需要团队仲裁
  - Missing: N 项需团队补充或决定省略
```

---

## 字段说明

### Question source / original / type tag

直接对齐 wording skill Phase 1 输出。如果 wording skill 已经跑过 Phase 1，直接复用它的 type tag；如果是 grounding-first 路径，由 grounding skill 自己识别。

### Sources searched / NOT searched

明确告诉用户**这次 grounding 覆盖了哪些源、还有哪些没覆盖**。例：

```
Sources searched:
  - cloud.tencent.com/document
  - www.tencentcloud.com/document

Sources NOT searched (out of v0.1 scope):
  - 云知（v0.2 加）
  - 公众号（v0.3 加）
```

让用户知道这次 evidence 的 boundary——避免误以为已经穷尽所有内部资料。

### Candidate pool

候选产品池**全部列出**，每个带：

1. **产品名**（中英都给，便于后续翻译）
2. **Description (draft)**：1 段技术描述（中文版；如果国际版有英文官方描述，附在后面）
3. **Provenance**：完整 audit chain（详见 [audit-chain-format.md](audit-chain-format.md)）

候选池规模建议 8-10 个（详见 [query-decomposition.md](query-decomposition.md)）。

**重要**：候选池**不排序、不挑选**。让团队 / wording skill 来决定 top N。

### Conflicts detected

按 fact 类别（SLA / 规模 / 客户名 / etc.）分组列出冲突。每个冲突带 ≥2 个 [CITED] + 1 个 [REVIEW: product]。详见 [source-priority.md](source-priority.md)。

### Missing required input

对照 wording skill `references/evidence-by-type.md` 的 schema，明确列出**应该有但没找到**的项。每项带 [REVIEW: product] + Reason + Suggested next step。

### Hand-off note

精炼摘要，让用户/wording skill 一眼看清：

- 候选池规模
- 总共多少项待 review
- 总共多少冲突
- 总共多少缺失

如果 hand-off note 里 "待 review" 数量很大（比如 > 候选池数量的 50%），意味着**这次 grounding 覆盖度不够**，建议团队优先解决 review 项再启动 wording skill。

---

## 完整示例（短版）

```
[Evidence Package]

Question source: 2026 Gartner Strategic Cloud / Q5
Question (original):
  EN: What are the five key features in your Security capabilities that distinguish your value proposition from competitors?
  CN: 贵公司安全能力中有哪五项关键特性使您的价值主张区别于竞争对手？
Question type tag (per wording skill taxonomy):
  - 大类: 产品能力
  - 形式: 列举（强制 5 个）
  - 修饰符: 量化, 客户引用, URL, 近 12 个月
Char limit: Gartner Description 500 char × 5 (+ URL + Comments columns)
Generation date: 2026-04-26
Sources searched: cloud.tencent.com/document, www.tencentcloud.com/document
Sources NOT searched (out of v0.1 scope): 云知 (v0.2), 公众号 (v0.3)

---

Candidate pool (10 个，团队挑 top 5):

Candidate 1: 云防火墙 (Cloud Firewall, CFW)

  Description (draft):
    腾讯云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态。
    [REVIEW: product] 跨客户群威胁情报同步窗口、日均拦截量等数据 v0.1 未覆盖。
  
  Provenance:
    [CITED] CFW 三种部署形态
      Source: cloud.tencent.com/document/product/1132/47570
      Quote: "云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态..."
      Retrieved: 2026-04-26
      Page-updated: 2025-11-03
    
    [REVIEW: product] 5-min 威胁情报同步 / 日均 [X] B 拦截量
      Reason: v0.1 公开文档无此量化数据；公众号 (v0.3 才覆盖) 可能有。需要 product 团队提供对外口径。
    
    [REVIEW: Kevin] Framing 选择: "single-pane policy across all three tiers" vs "real-time threat intel sharing"
      Reason: 两个 framing 都符合 source 事实。Lydia Leong 评分透镜下哪个更优？

Candidate 2: WAF

  Description (draft):
    ...

Candidate 3-10: ... (省略)

---

Conflicts detected:
(本次 grounding 未发现冲突；v0.1 source 单一。)

---

Missing required input (per wording skill 列举题 schema):

[REVIEW: product] 客户引用（每个 candidate）
  Reason: 公开文档无产品级 case study。
  Suggested next step: 走客户成功团队对外授权流程 (NDA OK 但需明确标记)；v0.2 后云知可能自动覆盖部分。

[REVIEW: product] 近 12 个月新发布点（每个 candidate）
  Reason: v0.1 不覆盖公众号；公开文档更新滞后。
  Suggested next step: v0.3 启用公众号 grounding 后自动覆盖；v0.2 期间团队人工提供。

---

Hand-off note for wording skill:

  - 候选池: 10 个安全产品，团队挑 top 5 (per Gartner Q5)
  - 待 product review: 12 项 (主要是缺失的量化指标 + 客户引用)
  - 待 Kevin review: 4 项 (5 个 candidate 的 differentiation framing 选择 + 整体的 5-feature 排序)
  - Conflicts: 0
  - Missing: 客户引用 + 近 12 个月新发布点（v0.x 范围限制，需补充）
  
  建议：先解决 product review 中的"量化指标"和"客户引用" 2 项 missing，再启动 wording skill Phase 2。
```

---

## 与 wording skill 的接口契约

Grounding skill 的输出必须满足以下契约（让 wording skill 能直接消费）：

1. **schema 字段名一致**：`Question type tag` / `Candidate pool` / `Provenance` / `Conflicts` / `Missing` / `Hand-off note` 这些字段名是 wording skill 期望看到的
2. **audit tag 严格**：[CITED] / [INFERRED] / [REVIEW: product] / [REVIEW: Kevin] 四种之一，不许变种
3. **timestamp 格式**：`YYYY-MM-DD`
4. **URL 明文**：不包 markdown link
5. **候选池不排序**：让 wording skill / 团队挑

如果 grounding skill 输出不满足这些契约，**wording skill 会拒收并报错**（理论上；实际行为视 wording skill 实现）。

---

## 何时 grounding skill 应该停止 + 让 wording skill 接手

Grounding skill 完成 Phase 5 后**立即停止**。不要：

- 替团队挑 top N
- 起草中文/英文答案
- 给 framing 推荐 (除非标 [REVIEW: Kevin])
- 仲裁冲突 (除非标 [REVIEW: product])

这些都是 wording skill 或团队的工作。grounding skill 的职责是**provenance-tagged evidence package**，到此为止。
