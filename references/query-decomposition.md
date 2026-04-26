# Query 拆解：题目 → 产品候选池 → search query

> Skill 在 Phase 1 使用。

## 核心动作

不要拿题目原文去搜。要先反向枚举 Tencent Cloud 产品名，然后对每个产品独立 grounding。

理由：

- 题目里的术语（"AI agents capabilities"、"sovereignty capabilities"）是分析师视角的功能簇，不是 Tencent 内部产品名
- 搜"Tencent AI agents" 会拿到大量不相关的网页 + 大量 hallucination 风险
- 搜具体产品名（"TI-AGENT"、"Hunyuan"、"TCS"）才能拿到准确的官方文档

## 拆解步骤

### Step 1：从题目抽 functional area

读题目，识别这道题在问什么 functional area：

| 题目类型 | functional area 例子 |
|---|---|
| Security 5 features | 网络安全、主机安全、身份、数据、AI 安全 |
| AI agents 5 features | agent 开发、orchestration、management、AI 安全 |
| Sovereignty 5 features | 数据主权、运营主权、技术主权、合规 |
| Database services | SQL、NoSQL、分布式、缓存、HTAP、向量、图 |
| ... | ... |

### Step 2：枚举 functional area → Tencent 产品

对每个 functional area，列出 Tencent Cloud 实际产品名：

例：Security 题 → 候选产品池

```
- 云防火墙 CFW (Cloud Firewall)
- WAF
- 主机安全 CWP (Cloud Workload Protection)
- 零信任接入
- 腾讯云风控 TCRS (Tencent Cloud Risk Service)
- 数据安全治理 DSGC (Data Security Governance Center)
- 云访问安全 CSC
- 密钥管理服务 KMS
- 凭据管理服务 SSM
- DDoS 防护
- T-Sec 系列
- AI 内容安全
```

例：AI agents 题 → 候选产品池

```
- TI-AGENT
- Hunyuan agent
- ADP (AI Development Platform)
- TI-ONE
- MaaS
- AI 数据增强工具
- ...
```

**这一步必须人工 + skill 协作**：

- skill 给一个起步候选池（基于训练数据的 Tencent 产品知识）
- **请用户/团队补充**：v0.1 不能自动从云知里枚举，所以可能漏产品
- 拿到补充后再进入 Step 3

### Step 3：每个产品 → search query

对每个候选产品，生成 1-2 个 search query：

| 产品 | 主 query（找 overview） | 辅 query（找近期更新）|
|---|---|---|
| 云防火墙 | `cloud.tencent.com/document/product/1132` | `cloud.tencent.com/document/product/1132 2025` 或公众号 "云防火墙" |
| WAF | `cloud.tencent.com/document/product/627` | `cloud.tencent.com/document/product/627 2025` |
| ... | ... | ... |

主 query：抓产品总览页（描述、功能列表、SLA）
辅 query：抓近 12 个月的 launch 文章（公众号最准，v0.3 才能自动；v0.1 用户需手动给 URL）

### Step 4：搜索 + 抓取（进入 Phase 2）

对每个 query 执行 Phase 2 的抓取流程。

---

## 反向映射技巧

### 当题目用了"功能簇"措辞而非产品名

例：题目问 "5 key features in your support for multiple AI models"

不要搜 "support for multiple AI models"，要反推：

- "支持多种 AI 模型" → Tencent 哪个产品做这件事？
- 答案：**MaaS**（模型托管）+ **TI-ONE**（模型训练）+ **Hunyuan API**（自有模型接入）

### 当 functional area 跨多个产品

例：题目问 "5 key features in cloud infrastructure for AI"

→ 跨 compute / storage / network / GPU 集群多个产品。先列：

- 计算：GPU 实例（GN 系列）、HCC（高性能计算集群）
- 存储：CFS Turbo（高性能并行文件系统）、GooseFS
- 网络：RDMA over Converged Ethernet (RoCE)
- 调度：TKE GPU、TI-ACC

每个产品独立 grounding，让团队最后挑 5 个。

### 当题目里有 Tencent 内部代号

例：题目（罕见）直接提到 "Tencent Cloud Native Database (TDSQL)" → 直接以 TDSQL 为 query。

---

## 候选池规模建议

每道 5-feature 题，候选池建议 **8-10 个产品**（不少于 8，不多于 12）：

- 少于 8：团队挑选空间不够，可能错过更好的 feature
- 多于 12：grounding 工作量爆炸，团队也挑得累

如果发现某个 functional area 候选 < 8，明确告诉用户：

```
[REVIEW: product] AI agents 候选池只有 5 个产品（< 推荐 8）
  Reason: v0.1 范围内（公开文档 + skill 训练数据）只能识别 5 个 Tencent AI agent 相关产品。
  Suggested next step: 团队补充候选产品（特别是 2025-2026 内新发布的，可能未进文档主索引）。
```

---

## 不许做的事

1. **不许直接拿题目原文去 search**——召回低，噪音大
2. **不许靠 LLM 知识脑补 Tencent 产品**——必须用 skill training data + 用户补充，编出不存在的产品名是大忌
3. **不许跨题复用候选池**——每道题独立拆，候选池可能有重叠但要单独走 grounding
4. **不许在 Phase 1 就挑 top N**——Phase 1 只产出候选池，挑选是 wording skill / 团队的事
