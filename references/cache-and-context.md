# Cache + Context Window 策略

> Skill 在 Phase 2（source 抓取）和 Phase 3（audit-tagged 抽取）使用。
>
> 解决两个问题：
> 1. 跨题目重复抓同一 URL → 慢 + 浪费
> 2. 1M context 不等于"什么都塞进去" → lost-in-the-middle 性能衰减

---

## 三层 Cache 架构

```
L1: session 内 in-memory
    └── 一道题 / 一次 skill run 内已抓的 URL 不重抓

L2: disk cache (cache/ 目录, .gitignored)
    └── 跨 session / 跨题目重用

L3: source fetch (Firecrawl / Playwright / WeRSS)
    └── 只在 L1 + L2 都 miss 时才执行
```

---

## L2 Cache 结构

### 目录布局

```
cache/                                   # 整个目录被 .gitignore
├── _index.json                          # URL → 元数据 索引
├── tencent-doc/                         # cloud.tencent.com/document
│   ├── product-1132-47570.md
│   └── product-1132-47571.md
├── tencentcloud-doc/                    # www.tencentcloud.com/document (国际版)
│   └── ...
├── wechat/                              # v0.3
│   └── s-abc123.md
└── yunzhi/                              # v0.2
    └── ...
```

### 单个 cache 文件格式

每个文件以 YAML frontmatter 开头，便于 grep 和审计：

```markdown
---
url: https://cloud.tencent.com/document/product/1132/47570
retrieved: 2026-04-26T17:30:00+08:00
page_updated: 2025-11-03
published:                                 # 公众号专用，文档留空
content_hash: sha256:abc123def456...
fetch_method: firecrawl
fetch_status: ok
---

# 云防火墙

云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态...

[此处是 Firecrawl 返回的完整 markdown 正文]
```

### Cache 索引

`cache/_index.json`：

```json
{
  "https://cloud.tencent.com/document/product/1132/47570": {
    "cache_file": "tencent-doc/product-1132-47570.md",
    "retrieved": "2026-04-26T17:30:00+08:00",
    "page_updated": "2025-11-03",
    "ttl_until": "2026-05-26T17:30:00+08:00",
    "content_hash": "sha256:abc123def456...",
    "fetch_method": "firecrawl",
    "questions_using": ["Gartner-Q5", "Gartner-Q11", "Forrester-Q23"]
  }
}
```

`questions_using` 数组帮助：

- 跨题目去重统计（CFW 被多少道题引用）
- 需要 invalidate 时知道影响范围
- 团队 review 时知道某个 source 的覆盖度

---

## Cache 命中流程

```
对每个候选 URL：
  1. L1 检查（本次 skill run 已抓过？）
       hit  → 直接用 in-memory copy
       miss → 进 L2

  2. L2 检查（cache/_index.json 里有 + 未过 TTL？）
       hit  → 读 cache 文件
       stale (TTL 过) → 重新 fetch (走 L3)
       miss → 进 L3

  3. L3 fetch（Firecrawl / Playwright / WeRSS+Firecrawl）
       success → 写入 L2 + 更新索引
       fail    → 标 [REVIEW: product] "URL 抓取失败"
```

---

## TTL 规则

默认 TTL：

| Source | TTL | 理由 |
|---|---|---|
| cloud.tencent.com/document | 30 天 | 文档更新慢，30 天足够 |
| www.tencentcloud.com/document | 30 天 | 同上 |
| 公众号 (v0.3) | 永久（按 published 不变）| 文章发布后不修改 |
| 云知 (v0.2) | 7 天 | 内部 wiki 更新更频，缩短 TTL |

**手动 refresh**：

用户可指令 `refresh CFW` / `refresh tencent-doc` → 把对应 cache 标 stale，下次访问重抓。

**项目级 refresh**：

新一期问卷开始时，user 可选 `refresh-cache --all` 全清。建议每个新季度的 Wave/MQ 项目开始时跑一次。

**Page-updated 检查（advanced）**：

更高级的策略：每次 cache hit 时，先 HEAD 请求看 source 的 `Last-Modified` header，如果变了就 invalidate。但增加 1 次 HTTP roundtrip。v0.1 不做，v0.2 视情况加。

---

## 1M Context 性能优化

**1M context 不是"全塞进去"**。Anthropic 自己也确认：超长 context 会触发 lost-in-the-middle 衰减——recall 在中部下降。

Grounding skill 用以下 4 条规则保持 context 紧凑：

### 规则 1：Lazy load

Cache 文件**存在硬盘，不进 context**。skill 只在确认某个 URL 与当前 question 相关时才把它读进 context。

❌ 错误：skill 启动时 load 所有 cache 进 context
✅ 正确：每个 URL 按需 load

### 规则 2：Grep before load

判断"哪些 cache 文件与当前 question 相关"时，用 **Bash + ripgrep**（几乎零 AI token），不用 LLM 阅读：

```bash
# 找出所有 cache 文件里提到 "VPC 边界" 的
rg -l "VPC 边界" cache/

# 找出 last 30 days fetched 的
rg -l "retrieved: 2026-0[34]" cache/
```

只把 grep 命中的文件 load 进 context。

### 规则 3：Snippet extract，不 dump 全文

[CITED] 的 Quote 字段只截 **1-3 句话**支撑 claim，不要把整个 doc 抄进去。

❌ 错误：
```
[CITED] CFW 三层部署
  Quote: "<整篇 5000-token 的产品文档>"
```

✅ 正确：
```
[CITED] CFW 三层部署
  Quote: "云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态。"
```

snippet 长度规则：

- 1-3 句话
- 包含 claim 的关键 keyword
- 留 ±1 句上下文（让审计者能理解语境）

### 规则 4：Per-question isolation

每道题的 evidence package **独立**。wrapper 跑 30 道题时，**不要把前 N 道题的全部 evidence 累计进当前 context**。

实现上：

- 每道题开始时清空 grounding 的"working memory"
- 仅保留 cross-question 的轻量元数据（如 `_index.json` 里的 questions_using）
- evidence package 输出后写入硬盘，不留在 context

---

## Context 预算（per question）

按上述规则，单题 grounding 的 context 大致：

| 内容 | 预估 token |
|---|---|
| 题目原文 + 字数限制 | ~500 |
| Question type tag + schema | ~500 |
| 候选产品池（10 个产品名 + 1 行描述）| ~500 |
| Cache index 相关条目 | ~500 |
| Grep 命中的 cache 文件（snippet 模式）| ~3,000 |
| Audit-tagged 输出（中间产物）| ~2,000 |
| **总计** | **~7,000** |

7K tokens 远低于 1M 上限。有头部空间是正常的——1M context 是**长尾保护**而非"常态使用"。

如果某次 grounding 单题超 30K，触发警告：

```
[REVIEW: product] 单题 context 超 30K，可能 over-fetched
  Reason: 检查候选池规模 / snippet 长度 / 是否漏掉 grep 过滤
```

---

## Wrapper 跑 30 道题时的 context 策略

Wrapper skill「跑问卷」编排 30 题时：

- 每道题独立调 grounding skill（grounding skill 自己 isolate）
- Wrapper 自己**不累计 evidence**，只保留：
  - 题目状态（done / pending / blocked）
  - cross-question 一致性 check 用的轻量元数据（如同一产品在多题里的引用 SLA 数字）
  - reviewer 待办 master list

Wrapper 的 context 不应超过 50K——超了说明它在做太多事，应该把工作 delegate 给 sub-skills。

---

## 不许做的事

1. **不许把 cache 全部 load 进 context** —— 用 grep 过滤
2. **不许 [CITED] 的 Quote 字段塞整个 doc** —— 1-3 句话
3. **不许跨题累计 evidence** —— per-question isolation
4. **不许信任 cache 永远 fresh** —— TTL 必须检查
5. **不许 cache miss 时 silent fail** —— 抓失败要标 [REVIEW: product]
6. **不许把 cache/ 目录入 git** —— `.gitignore` 已加

---

## v0.x 演进

| 版本 | Cache 能力 |
|---|---|
| v0.1（current）| 文档源 cache + TTL 30 天 + 手动 refresh |
| v0.2 | 加云知 cache（TTL 7 天）+ HEAD 请求做 freshness 检查 |
| v0.3 | 加公众号 cache（永久 TTL，按 published）+ WeRSS manifest 维护 |
| v1.0+（如需）| Pre-index + embedding 检索（如果 grep 召回不够） |
