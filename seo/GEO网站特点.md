# GEO 检查手册（生成式引擎优化审计）

> GEO = Generative Engine Optimization，让内容被 ChatGPT / Perplexity / Gemini / Google AI Overviews / Claude / 豆包 等**答案引擎抓取、引用、推荐**。
> 本手册是「检查过程文档」：下次给任意站点做 GEO 体检时，按下面流程逐项走，最后套用报告模板输出详细建议。
> 参考实现：MkFast（TanStarter）模板 `G:\Projects\seo+geo模板项目\mkfast-template`；实测案例见文末 SkuMotion。

---

## 〇、使用说明

- 每一条检查项都按固定格式：**怎么查（命令）→ 通过标准 → 常见问题 → 修复建议**。
- 建议顺序从 A 到 F 走，因为 A/B 是「门禁和入口」，没开就是零，先查。
- 全部查完，套用「五、检查报告模板」输出，再对照「六、案例」看差距。

---

## 一、检查总流程（6 步）

1. **A. 抓取与门禁**——AI 爬虫能不能进来、训练爬虫有没有拦住
2. **B. 机器入口**——有没有 llms.txt / pricing.txt / RSS / Markdown 原文
3. **C. 结构化数据**——JSON-LD 实体全不全、对不对
4. **D. 内容可引用性**——内容好不好被 AI 整段引用
5. **E. 新鲜度与可信度**——AI 愿不愿意信、愿不愿意引
6. **F. 度量与归因**——知不知道 AI 流量来自哪、有没有效果

---

## 二、详细检查清单

### A. 抓取与门禁（robots.txt）

**怎么查**
```bash
curl -s https://你的站点/robots.txt
```

**通过标准**
- [ ] `User-agent: *` 下 `Allow: /`，私有路径（/api /auth /admin /settings 等）正确 `Disallow`
- [ ] 显式允许 AI 发现/引用爬虫：`OAI-SearchBot`、`ChatGPT-User`、`Claude-SearchBot`、`Claude-User`、`PerplexityBot`、`Googlebot`、`Bingbot`、`DeepSeekBot`、`Kimi-User`、`TavilyBot` 等
- [ ] 显式拦截 AI 训练爬虫：`GPTBot`、`ClaudeBot`、`Google-Extended`、`CCBot`、`Applebot-Extended` 等（`Disallow: /`）
- [ ] 声明 `Sitemap:` 和 `# llms.txt:`

**常见问题**
- 只写了 `User-agent: * Allow: /`，没单独给 AI 爬虫开门 → 部分答案引擎不敢抓。
- 训练爬虫没拦 → 内容被白嫖去训练，版权风险。
- 私有路径没 Disallow → 后台/接口被爬。

**修复建议**
- 复制一份标准分段模板：发现/引用爬虫一组 `Allow`，训练爬虫一组 `Disallow`，私有路径统一 Disallow。
- 高级：加 Cloudflare `Content-Signal`（见 B-5），用一行声明版权立场。

---

### B. 机器入口（llms.txt / pricing.txt / RSS / Markdown）

#### B-1. llms.txt（站点目录，最重要）

**怎么查**
```bash
curl -s https://你的站点/llms.txt
```

**通过标准**
- [ ] 存在，且 `Content-Type: text/plain`
- [ ] 含「一句话站点摘要」（AI 引用时直接抄这个定位）
- [ ] 列出核心页 URL（**含每个语言版本**，不用 AI 解析 hreflang）
- [ ] 列出长尾落地页（use-cases / solutions / compare 等）
- [ ] 有 `Product Facts`（产品事实：价格、积分、退款、支持等，让 AI 不乱编）
- [ ] 有 `Audience`（目标人群）
- [ ] 有 `Contact`（联系方式）

**常见问题**
- llms.txt 只有几个首页链接，没有产品事实 → AI 引用时只能靠猜，容易出错。
- 没列语言版本 → 多语言站点 AI 只抓到一种语言。

**修复建议**
- 参考 SkuMotion 的 llms.txt 结构：`站点名 → 摘要 → Official URLs → Core Use Cases → Product Facts → Audience → Contact`。
- 「Product Facts」尤其关键：把价格、积分有效期、退款规则、企业版能力写成陈述句，AI 引用时就不会瞎编。

#### B-2. llms-full.txt（全文聚合）

**怎么查**
```bash
curl -s https://你的站点/llms-full.txt | head -c 500
```

**通过标准**
- [ ] 存在，聚合所有 `content/**/*.md` 原文
- [ ] 内容完整、纯文本

**常见问题**
- 缺失 → 中小站点 AI 抓全文要翻多页，抓取预算高、被引用概率低。

**修复建议**
- 若内容量不大（< 几百篇），直接生成 llms-full.txt 一次给全；内容量大则保留 llms.txt 目录 + 每篇 Markdown 端点。

#### B-3. Markdown 原文端点

**怎么查**
```bash
curl -sI https://你的站点/markdown/某篇文章slug   # 看是否返回 text/markdown
```

**通过标准**
- [ ] 每篇内容有 `text/markdown` 原文端点（去掉 HTML/CSS/JS 噪声）
- [ ] 返回类型是 `text/markdown`，不是回退到 HTML 壳

**常见问题**
- 缺失 → AI 只能解析带噪声的 HTML，引用精度下降。

**修复建议**
- 加 `/markdown/:slug` 路由，直接把 Markdown 原文按 `text/markdown` 输出。

#### B-4. RSS

**怎么查**
```bash
curl -sI https://你的站点/blog/rss.xml   # 或 /feed.xml、/rss.xml、/feed
```

**通过标准**
- [ ] 存在 RSS/Atom，含 `pubDate / updated`
- [ ] 更新条目带完整正文或摘要

**常见问题**
- 404 → 更新无法分发给聚合器和 AI，新内容发现慢。

**修复建议**
- 加 `/blog/rss.xml`，用 `pubDate` 保证时间戳，输出最近 N 篇。

#### B-5. 商业事实端点 pricing.txt（可选，但很加分）

**怎么查**
```bash
curl -s https://你的站点/pricing.txt
```

**通过标准**
- [ ] 存在，含价格模型、官方定价页链接、商业事实（积分/退款/企业版/联系方式）

**修复建议**
- AI 常被问「XX 多少钱」，单独维护 pricing.txt 能避免价格引用错误。SkuMotion 已做，值得抄。

#### B-6. Content-Signal（版权信号，Cloudflare 新规范）

**怎么查**
```bash
curl -s https://你的站点/robots.txt | grep -i content-signal
```

**通过标准**
- [ ] robots.txt 声明 `Content-Signal: search=yes,ai-input=yes,ai-train=no,use=reference`

**含义**
- `search=yes` 允许建搜索索引；`ai-input=yes` 允许输入 AI 模型；`ai-train=no` 禁止训练；`use=reference` 引用方式为参考引用。

**修复建议**
- 想「被 AI 引用但不想被白嫖训练」，就加这一行。这是目前最干净的版权声明方式。

---

### C. 结构化数据（JSON-LD）

**怎么查**
```bash
curl -s https://你的站点/ | grep -o 'application/ld+json' | wc -l   # 看数量
curl -s https://你的站点/ | grep -A2 'application/ld+json'          # 看内容
```

**通过标准（核心实体）**
- [ ] `Organization`：name、url、logo、**sameAs（社交档案，不能为空）**
- [ ] `WebSite`：`potentialAction`（SearchAction 站内搜索）
- [ ] `SoftwareApplication` 或 `Product`：**offers（价格）+ featureList + audience**（工具类站点必配）
- [ ] `FAQPage`：与页面**可见**的 Q&A 同源（不能只给 JSON-LD 不给正文）
- [ ] `Article`/`BlogPosting`：`datePublished / dateModified / wordCount / keywords / articleSection / author / publisher`
- [ ] `BreadcrumbList`：面包屑（可选但推荐）

**常见问题**
- `sameAs: []` 空数组 → 实体对账弱，AI 无法跨平台确认品牌。
- 只有 Organization，没有 WebSite/SoftwareApplication → 工具类站点浪费了「应用/产品」富结果。
- FAQPage 与页面内容不一致 → 被判结构化数据欺诈。

**修复建议**
- 给 `sameAs` 填上 GitHub/Twitter/X/YouTube/Discord 等真实主页。
- 工具类站点务必加 `SoftwareApplication + Offer`（价格、币种、availability）。
- FAQ 用同一份数据渲染正文和 JSON-LD，保证同源。

---

### D. 内容可引用性（Citability）

**怎么查**（人工 + 抓正文）
```bash
curl -s https://你的站点/ | grep -o '<h[12][^>]*>'   # 看标题结构
```

**通过标准**
- [ ] 标题有 ID / 锚点，可深链到具体段落（`#小节`）
- [ ] 段首直接给结论 / TL;DR，答案前置
- [ ] H2 用问句（「怎么做」「多少钱」「适合谁」）
- [ ] 每段自包含，不依赖上下文也能读懂
- [ ] 关键信息用表格/列表对比，事实密度高
- [ ] front-matter 带 `summary / updated / keywords / author`

**常见问题**
- 标题无锚点 → AI 只能引用整页，无法精确到段落，被引概率下降。
- 开头长篇铺垫、结论埋在后面 → 答案引擎抽取不到「像答案的句子」。
- 段落依赖上下文 → 单独摘出来看不懂，AI 放弃引用。

**修复建议**
- Markdown 渲染加 `rehypeSlug + rehypeAutolinkHeadings`（MkFast 模板 `src/lib/markdown.ts` 已这么做）。
- 每个 H2 写成一个可独立回答的问答块：问句标题 + 第一句给答案 + 表格支撑。
- 把「产品事实」写成陈述短句，AI 直接抄。

---

### E. 新鲜度与可信度

**怎么查**
```bash
curl -s https://你的站点/ | grep -i 'dateModified\|updated'
curl -s https://你的站点/sitemap.xml | grep -o '<lastmod>[^<]*' | head
```

**通过标准**
- [ ] 内容有 `updated / dateModified` 时间戳
- [ ] sitemap 每条有 `lastmod`
- [ ] 文章有 `author`（Person 或 Organization）
- [ ] 有 `wordCount`

**常见问题**
- 无更新时间 → AI 认为内容陈旧，跳过。
- 无作者/机构 → 可信度低，被引用意愿低。

**修复建议**
- 文章 front-matter 加 `updated`，JSON-LD 输出 `dateModified`，sitemap 输出 `lastmod`，三者同源。
- 定期刷新（哪怕只改小处）并更新 `updated` 字段。

---

### F. 度量与归因

**怎么查**（看你的分析工具）
```bash
# 在分析后台查引荐来源
```

**通过标准**
- [ ] 能区分 `chatgpt.com / perplexity.ai / gemini.google.com / copilot.microsoft.com / 豆包` 等 AI 引荐来源
- [ ] 记录「AI 引用带来了多少访问/转化」

**常见问题**
- 不统计 AI 引荐 → GEO 投入无法衡量，等于玄学。

**修复建议**
- 在分析里给 AI 引荐域名打标，单独看趋势；再决定要不要继续投 GEO。

---

## 三、评分表（快速打分）

每项 0/1/2 分（0=缺失，1=有但不完整，2=达标）：

| 检查项 | 权重 | 得分 |
|---|---|---|
| A 抓取与门禁 | 高 | /2 |
| B-1 llms.txt | 高 | /2 |
| B-2 llms-full.txt | 中 | /2 |
| B-3 Markdown 端点 | 中 | /2 |
| B-4 RSS | 中 | /2 |
| B-5 pricing.txt | 低 | /2 |
| C JSON-LD | 高 | /2 |
| D 内容可引用性 | 高 | /2 |
| E 新鲜度可信度 | 中 | /2 |
| F 度量归因 | 中 | /2 |

> 总分 ≥ 16/20：GEO 成熟；12–15：及格但有明显缺口；< 12：基本没做 GEO。

---

## 四、检查报告模板（每次体检输出用）

```markdown
# [站点名] GEO 体检报告

日期：
网址：
结论：GEO 成熟度（X/20）

## 达标项
- ✅ ...

## 缺失/不达标项（按优先级）
1. 🔴 高优先：...
   - 现状：...
   - 建议：...
2. 🟡 中优先：...
3. 🟢 低优先：...

## 具体动作清单
- [ ] 动作 1（对应文件/端点）
- [ ] 动作 2
```

---

## 五、实测案例：SkuMotion（skumotion.com）

> 用途：拿一个真实站点对照上面的清单，看「看起来很强」的站实际差在哪。

### 达标项 ✅

- **robots.txt**：`Content-Signal: search=yes,ai-input=yes,ai-train=no,use=reference` + 100+ 个 user-agent 分段（发现/引用类 Allow，训练类 Disallow，私有路径 Disallow）
- **llms.txt**：教科书级（摘要 + Official URLs + Core Use Cases 长尾 + Product Facts + Audience + Contact）
- **pricing.txt**：独立商业事实端点，含定价页、积分规则、退款、企业版、联系方式
- **JSON-LD**：`Organization + WebSite(SearchAction) + SoftwareApplication(Offer $9.9 + featureList + audience) + FAQPage`
- **SEO 基础**：canonical + hreflang（zh-CN/en-US/x-default）、完整 OG/Twitter（含 og:image 1200×630）、三搜索引擎验证、sitemap 带 hreflang

### 缺口 🔴🟡

1. 🔴 **RSS 缺失**（`/blog/rss.xml` 实测 404）→ 更新无法分发，AI/聚合器发现新内容慢。
   - 建议：加 RSS，带 `pubDate/updated`。

2. 🔴 **Markdown 原文端点缺失**（`/markdown/...` 实测回退到 SPA 壳，非 text/markdown）→ AI 只能啃带噪声的 HTML。
   - 建议：加 `/markdown/:slug`，输出 `text/markdown`。

3. 🟡 **llms-full.txt 缺失**（连接被中止，大概率无）→ 抓全文成本高。
   - 建议：内容量不大就生成 llms-full.txt 聚合全文。

4. 🟡 **`Organization.sameAs: []` 为空** → 实体对账弱，AI 无法跨平台确认品牌。
   - 建议：填 GitHub/Twitter/YouTube 等真实主页。

5. 🟡 **未见 BreadcrumbList** → 面包屑结构化数据缺失（低优先）。
   - 建议：文章/产品页加 BreadcrumbList。

### 一句话结论

SkuMotion 的「门禁 + 入口 + 结构化数据」做得很强，但**「分发(RSS) + 原文(Markdown) + 实体(sameAs)」这三角是短板**——正好是 AI 引用时最影响「被引用精度和频率」的几项。补齐这三项，GEO 从「能抓到」升级到「被高频、精准引用」。
