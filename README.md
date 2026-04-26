<div align="center">

# Analyst Grounding

**从 Tencent Cloud 自有源抓 audit-tagged evidence 给 [analyst-wording](https://github.com/AOMJ2PMP/analyst-wording-skill) 用**

每条 fact 都带 provenance：可回查、可审计、可分配 reviewer

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.ai/code)

</div>

---

## 这个 skill 干什么

[analyst-wording](https://github.com/AOMJ2PMP/analyst-wording-skill) 解决了"怎么写让分析师喜欢的回答"。但写之前还有一步：**事实从哪来**。

这个 skill 干的就是 **grounding** —— 从 Tencent Cloud 自有源（公开文档、云知、公众号）抓事实，每条 fact 带 audit chain，handoff 给 wording skill 用。

```
原题
  ↓
[analyst-grounding]  ← 这个 skill
  搜：cloud.tencent.com/document
       www.tencentcloud.com/document
       云知（v0.2）
       公众号（v0.3）
  输出：evidence package + audit chain
  ↓
[analyst-wording]
  按题型 + 分析师 voice 写中文 + 英文
  ↓
团队 review → 提交
```

## 核心特性：Audit Chain

每条 evidence 都带 provenance tag，三类之一：

```
[CITED] CFW 支持三层部署形态
  Source: cloud.tencent.com/document/product/1132/47570
  Quote: "云防火墙（CFW）支持互联网边界、内部边界（VPC 间）和主机边界三种部署形态..."
  Retrieved: 2026-04-26
  Page-updated: 2025-11-03

[INFERRED] 新发现的恶意 IP 在数分钟内可被全网封禁
  Reasoning: 5 分钟同步窗口 + 三层覆盖 = 网络范围内 ≤ 5 分钟可达
  Based on:
    - [CITED] 5 分钟级同步
    - [CITED] 三层部署形态

[REVIEW: product] [X] B 恶意请求/天
  Reason: 占位符；公开文档无对外披露的请求量

[REVIEW: Kevin] Framing: "single-pane policy" vs "real-time threat sharing"
  Reason: 两个都符合 source 事实，需要 AR 视角判断哪个更打动 Lydia Leong
```

四种 tag：

| Tag | 含义 |
|---|---|
| `[CITED]` | 直接 quote source 原文（必带 URL + Quote + 时间戳） |
| `[INFERRED]` | 跨 ≥2 个 cited 推理（必带 reasoning） |
| `[REVIEW: product]` | 待产品专家 review（技术准确性、SLA、占位数字、冲突仲裁）|
| `[REVIEW: Kevin]` | 待 Kevin (AR / ex-Gartner VP) review（战略 framing、分析师评分透镜） |

下游 reviewer 扫一遍输出就知道**自己负责确认什么**——audit chain + 责任分配一次完成。

## 工作流（5 phase）

```
原题 + 字数限制
        ↓
Phase 1: query 拆解（题目 → Tencent 产品候选池 → search queries）
        ↓
Phase 2: source 抓取（Firecrawl / Playwright，带 timestamp）
        ↓
Phase 3: audit-tagged 抽取（每条 fact 打 CITED/INFERRED/REVIEW tag）
        ↓
Phase 4: 冲突检测 + 缺失项标记（不 merge，全 surface）
        ↓
Phase 5: 输出 evidence package（schema 对齐 wording skill input）
```

详见 [SKILL.md](SKILL.md)。

## 设计原则

1. **每条 fact 必须带 audit tag**。无 tag 不出输出。
2. **CITED 必须带原文 quote**。能逐字 grep 回 source。
3. **INFERRED 必须显式推理**。"based on industry knowledge" = 编。
4. **冲突 surface，不 merge**。两个 source 说不同的，全列出 + [REVIEW: product]。
5. **timestamp 强制**：每个 source 必须带 `Retrieved` + `Page-updated` 或 `Published`。
6. **不挑 top N**。grounding 只产出候选池，挑选是 wording skill / 团队的事。
7. **不写答案**。grounding 只产出 evidence；写答案是 wording skill 的职责。

## Roadmap

```
v0.1 (current)
  cloud.tencent.com/document          via Firecrawl
  www.tencentcloud.com/document       via Firecrawl

v0.2 (next)
  云知 (内部知识库)                    via Playwright (browser automation)
    依赖：用户能登录云知 + Tencent IT 允许浏览器自动化

v0.3 (next next)
  公众号 (mp.weixin.qq.com)           via WeRSS / RSSHub (discovery) + Firecrawl (scrape)
    依赖：选定 RSS 服务（WeRSS 商用 / RSSHub 自部署）+ 维护订阅 manifest
```

每个 v0.x 上线前跑 1-2 道真题验证 → 修 bug → 再扩。

## 安装

### 方法 A：克隆到 Claude skills 目录（推荐）

```bash
git clone https://github.com/AOMJ2PMP/analyst-grounding-skill ~/.claude/skills/analyst-grounding
```

### 方法 B：克隆到工作目录 + symlink（迭代友好）

```bash
git clone https://github.com/AOMJ2PMP/analyst-grounding-skill ~/code/analyst-grounding-skill
ln -s ~/code/analyst-grounding-skill ~/.claude/skills/analyst-grounding
```

同时安装 [analyst-wording](https://github.com/AOMJ2PMP/analyst-wording-skill) 才能跑完整流程：

```bash
git clone https://github.com/AOMJ2PMP/analyst-wording-skill ~/.claude/skills/analyst-wording
```

## 触发词

- 「找事实」「grounding」
- 「查文档」「问卷取证」
- 「analyst evidence」「fetch evidence」
- 「帮我 ground」「ground 这道题」

## 文件结构

```
analyst-grounding-skill/
├── SKILL.md                              # 主入口（5-phase 协议）
├── references/                           # Skill 运行时加载
│   ├── audit-chain-format.md             # CITED / INFERRED / REVIEW 格式（Phase 3）
│   ├── source-priority.md                # 冲突解决 + freshness（Phase 4）
│   ├── source-access-mechanics.md        # 各 source 怎么访问 + roadmap（Phase 2）
│   ├── query-decomposition.md            # 题目 → 产品候选池（Phase 1）
│   └── handoff-to-wording.md             # 输出 schema，对齐 wording input（Phase 5）
├── README.md
└── LICENSE
```

## 后续怎么迭代

迭代主要在 `references/`，少动 SKILL.md。

| 想改什么 | 改哪个文件 |
|---|---|
| 加新 source 类型 | `references/source-access-mechanics.md` |
| 改冲突 / 优先级规则 | `references/source-priority.md` |
| 改 audit tag 格式 | `references/audit-chain-format.md` |
| 改题目拆解策略 | `references/query-decomposition.md` |
| 改输出 schema | `references/handoff-to-wording.md` |
| 改协议本身 | `SKILL.md`（少做） |

### 最常见的迭代

**1. 发现某 source 抓不到关键信息**：

  - 是 source 范围问题（v0.1 不覆盖云知）→ 加 [REVIEW: product]，等 v0.2 解决
  - 是抓取规则问题（漏了某个子页）→ 改 `source-access-mechanics.md` 的抓取策略

**2. 发现 [INFERRED] 推理太弱**：

  - 改 `audit-chain-format.md` 加严约束（如要求至少 3 个 [CITED]）
  - 或改 `query-decomposition.md` 让候选池更全，减少 inference 需要

**3. 发现 reviewer 经常分错（product/Kevin）**：

  - 改 `audit-chain-format.md` 的"触发场景"清单

**4. 发现 wording skill 接不住 evidence**：

  - 改 `handoff-to-wording.md` 的 schema 对齐 wording skill 实际期望

## 配套使用

```
原题 → analyst-grounding → evidence package → analyst-wording → CN draft → EN final
```

或通过 wrapper skill「跑问卷」自动编排（roadmap）。

## 起源

- 灵感来自 [nuwa-skill](https://github.com/alchaincyf/nuwa-skill) 的 protocol 结构
- 与 [analyst-wording](https://github.com/AOMJ2PMP/analyst-wording-skill) 是 parallel 关系，串联跑
- 为腾讯云分析师关系工作流量身设计，但 skill 通用，任何 vendor 都可以 fork 后调整 source 列表

## License

MIT — 见 [LICENSE](LICENSE)
