# Source Access Mechanics

> Skill 在 Phase 2（source 抓取）使用。每个 source 怎么访问、什么 v0.x 上线、有什么限制。

## v0.1（当前）—— 公开文档

### `cloud.tencent.com/document` (中文文档)

工具：**Firecrawl** (`mcp__firecrawl__firecrawl_scrape` / `firecrawl_search` / `firecrawl_map`)

抓取策略：

1. 先用 `firecrawl_map` 给定产品根 URL 列出所有子页：
   ```
   firecrawl_map(url="https://cloud.tencent.com/document/product/1132", search="<keyword>")
   ```
2. 对感兴趣的子页用 `firecrawl_scrape` 抓 markdown：
   ```
   firecrawl_scrape(url="<sub-url>", formats=["markdown"], onlyMainContent=true)
   ```
3. 抓 metadata 时包括 `og:updated_time` / `article:modified_time` / `<meta name="lastmod">` 等任何能拿到的时间戳

注意事项：

- 文档结构有产品总览 → 子模块 → 具体特性 / API 三层
- 抓总览 + 特性页足够 grounding；不需要全站爬
- 同一产品中英两版结构一致，可双抓后比对（中文常更详细，英文常更分析师友好）

### `www.tencentcloud.com/document` (国际版英文文档)

工具：**Firecrawl**（同上）

策略：

- 优先抓 EN 版本（节约 wording skill 的翻译工作）
- 国内独有功能（如等保 2.0 / DSL 合规）回落中文版
- EN 版本如果是某产品 stub（"coming soon" 或 thin description），认为 EN 不可用，用中文版

---

## v0.2（roadmap）—— 云知

### 云知（内部知识库）

工具：**Playwright**（浏览器自动化）

为什么不用 Firecrawl：云知需要登录态 + 可能在企业内网 + 可能不是标准 web 而是 SPA。Playwright 能 handle 这些。

待 v0.2 调研的事项：

1. **Auth 机制**：单点登录？session cookie？需要 VPN？
2. **URL 结构**：是否有稳定 URL pattern 让 skill 直接 navigate？还是只能搜索？
3. **搜索 API**：是否有内部 search endpoint？还是只能模拟 UI 搜索？
4. **页面结构**：内容在 DOM 哪里？SPA 加载需要 wait 多久？
5. **Rate limit**：多频抓会不会被 IT 封？

v0.2 实现前必须确认：

- 用户能否在自己的电脑上登录云知（决定 Playwright 能不能跑）
- Tencent IT 是否允许浏览器自动化工具访问云知（合规问题）

---

## v0.3（roadmap）—— 公众号

### 公众号（mp.weixin.qq.com）

两阶段架构：

#### Stage 1: Discovery (找 URL)

公众号反爬严，**单纯靠 Firecrawl 拿不到文章列表**。需要专门服务：

- **WeRSS** (https://werss.app)：商用 RSS 服务，订阅特定公众号，每天发现新文章 → 输出 RSS feed (含 mp.weixin.qq.com URL)。免费版有限制，付费稳定。
- **RSSHub** (https://docs.rsshub.app)：开源，可自部署。`/wechat/mp/<biz>` 路由把公众号转 RSS。需要 ops 维护。
- **第三方公众号搜索引擎**（搜狗微信等）：滞后 + 不稳定，不推荐。

推荐 v0.3 起步：**WeRSS 付费版** 或 **自部署 RSSHub**（看团队偏好）。

discovery 输出 → 本地维护一份 manifest:

```
.local/wechat-manifest.json   (本地 cache，不入 repo)
[
  {
    "url": "https://mp.weixin.qq.com/s/abc123",
    "title": "...",
    "published": "2026-02-15",
    "discovered_via": "WeRSS",
    "discovered_at": "2026-02-16"
  },
  ...
]
```

#### Stage 2: Scrape (抓正文)

工具：**Firecrawl**（拿到 URL 后是可以抓的）

```
firecrawl_scrape(url="https://mp.weixin.qq.com/s/abc123", formats=["markdown"])
```

Firecrawl 拿到 mp.weixin.qq.com URL 抓正文是 work 的（关键障碍是 discovery 不是 scrape）。

注意事项：

- 公众号正文里 published 时间通常在文首，可从 metadata 抓
- 公众号文章常含图（Firecrawl 会忽略），但产品功能描述基本在文字
- 公众号 markdown 转换有时把"作者"/"封面"等当正文 → onlyMainContent=true

待 v0.3 调研事项：

1. WeRSS 价格 / RSSHub 自部署 ops 成本
2. 团队该订阅哪些公众号（核心：「腾讯云」「腾讯云开发者」「腾讯云 AI」「腾讯安全」等官方号）
3. Manifest 多频更新（每天 cron？）
4. 旧文是否需要 backfill（某个公众号过去 12 个月所有文章）

---

## v0.x 优先级与依赖

```
v0.1 (current)
  └── cloud.tencent.com/document         (Firecrawl)
  └── www.tencentcloud.com/document      (Firecrawl)

v0.2 (next)
  └── 云知                               (Playwright + auth)
       └── 依赖：用户能登录云知 + IT 允许浏览器自动化

v0.3 (next next)
  └── 公众号                             (WeRSS/RSSHub discovery + Firecrawl scrape)
       └── 依赖：选定 RSS 服务 + 启动 manifest 维护流程
```

每个 v0.x 上线前都跑 1-2 道真题验证 → 修 bug → 再扩。

---

## 通用抓取规则（所有 source 都适用）

1. **每次抓必须带 timestamp**：`Retrieved` 是抓的时间，`Page-updated` / `Published` 是源里的时间
2. **抓回的 raw content 不直接用**：要进 Phase 3 做 audit-tagged 抽取
3. **不缓存 raw content 长期**：抓完就用，过期就丢；下次需要再抓（除非 v0.1 升级到 hybrid pre-index）
4. **rate limit 友好**：批量抓时间隔 ≥ 200ms（避免对腾讯 CDN / 云知 / 微信 触发风控）
5. **失败要记录**：抓失败的 URL 列入 `[REVIEW: product]`（"该 URL 无法抓取，可能权限/网络问题，请人工核查"）

---

## 不在 grounding skill 范围内的 source

- **客户私下分享的资料**（PRD、销售 deck）—— 走人工，不走 skill
- **历年问卷历史答案**—— 历史口径 review 是另一个 workflow（参考 wording skill 边界声明）
- **公开 web 搜索**（除文档/公众号外）—— v0.x 不做，避免引用低质量第三方内容
- **第三方分析师报告（Gartner/Forrester）历史 PDF**—— 这是 evidence target 的一部分，不是 evidence source
