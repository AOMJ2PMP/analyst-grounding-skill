# Source 优先级 + 冲突 + 时间戳规则

> Skill 在 Phase 4（冲突检测）使用。

## 默认优先级

```
云知 (internal authoritative)
  > 文档 (cloud.tencent.com/document, official public)
  > 公众号 (mp.weixin.qq.com, PR-shaped, latest)
  > web search (3rd party, fallback)
```

含义：

- 上层源 silent 时 → 直接用下层源（不需要冲突标记）
- 上层源有内容时 → 上层为主，**但下层仍记录在 candidate 里**
- 上下层 disagree → 不 merge，全部 surface + `[REVIEW: product]`

## 时间戳让 priority 动态调整

**优先级是默认，不是绝对**。如果下层源比上层源**新很多**，可能反映上层未同步的更新。这种情况要 surface：

例：

```
Conflict on SLA number:

[CITED] "99.99% SLA"
  Source: cloud.tencent.com/document/product/cfw/sla
  Quote: "云防火墙服务可用性承诺达 99.99%..."
  Retrieved: 2026-04-26
  Page-updated: 2024-09-12              ← 18 个月前

[CITED] "99.995% SLA"
  Source: mp.weixin.qq.com/s/...（公众号《云防火墙能力升级》）
  Quote: "新版本 SLA 提升至 99.995%..."
  Published: 2026-02-15                  ← 2 个月前
  Retrieved: 2026-04-26

[REVIEW: product] 当前对外 SLA 口径
  Reason: 文档源（默认更高优先级）比公众号源旧 17 个月，可能反映文档未同步的升级。两个数字都需要 product 团队确认当前对外口径。
```

**规则**：

- 如果上层源 vs 下层源时间差 < 3 个月 → 按默认优先级（上层为准）
- 如果时间差 3-12 个月 → surface 两个 + `[REVIEW: product]`
- 如果时间差 > 12 个月 → surface 两个 + `[REVIEW: product]`，且在 Reason 里明确指出"上层源已经过时，建议优先采用下层源数据"

## 时间戳获取

每个 source 都要尝试抓 timestamp：

| Source 类型 | 时间戳字段 | 抓取位置 |
|---|---|---|
| cloud.tencent.com/document | `Page-updated` | 页面 footer / `<meta>` `dcterms:modified` / sitemap lastmod |
| www.tencentcloud.com/document | `Page-updated` | 同上（国际版结构一致）|
| 公众号 (mp.weixin.qq.com) | `Published` | 文章顶部发布时间 + meta `og:article:published_time` |
| 云知（v0.2）| `Page-updated` | 取决于云知 UI 结构（待 v0.2 调研）|
| web search | `Published` | 视具体源 |

**抓不到时间戳怎么办**：

- 留空字段，**不要瞎填**
- 在 Phase 4 触发额外 `[REVIEW: product]`，注明"未抓到时间戳，可能影响冲突判断"

## 冲突检测的具体动作

Phase 4 的算法：

```
对每个 fact 类型（如 SLA / 规模 / feature 列表 / 客户引用）：
  1. 收集所有 [CITED] 项
  2. 比较 claim 是否一致
     - 一致 → 不冲突，保留所有 [CITED]（多源支撑反而加分）
     - 不一致 → 进入冲突处理
  3. 冲突处理：
     a. 检查时间差
     b. 按上面"时间戳让 priority 动态调整"规则决定
     c. 一律 surface 所有版本 + 添加 [REVIEW: product] 项
     d. Reason 里说明冲突来源 + 建议（如果时间差明显）
```

## 缺失项的标记

对照 wording skill `references/evidence-by-type.md` 里的 schema，**每个题型有 required 和 optional 项**。grounding skill 应主动检查：

```
对照题型 schema 检查 required 项：
  for required_item in schema:
      if not found in any source:
          emit [REVIEW: product] with Reason 解释为什么没找到
          + 建议哪些非自动化的途径可以补（如"客户引用：走客户成功团队对外授权流程"）
```

例（产品能力 × 列举题）：

```
Required by wording skill but not auto-grounded:

[REVIEW: product] 客户引用
  Reason: v0.1 范围只覆盖公开文档；公开文档无产品级 case study。
  Suggested next step: 查云知（v0.2 后 grounding 可自动覆盖）；或走客户成功团队对外授权流程。

[REVIEW: product] 量化指标 [X] B requests/day
  Reason: 文档 SLA 页未披露请求量；公众号查到的最近 launch 文章里也无该量级数据。
  Suggested next step: product 团队提供对外可披露版本，或同意省略此量化（用 "operating at scale" 代替）。
```

## 不许做的事

1. **不许仲裁冲突**。两个 source 说不同的，skill 只 surface，不裁判。
2. **不许 hide 低优先级源的数据**。即使上层源覆盖了，下层源数据也要列在 candidate 里（团队可能想知道公众号是怎么对外说的）。
3. **不许靠时间戳就自动选**。即使下层源比上层新很多，也要标 [REVIEW: product]，不许自动 promote。
4. **不许编时间戳**。抓不到就留空，不要猜。
