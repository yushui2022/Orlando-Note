# GEO 网站特点（生成式引擎优化）

> GEO = Generative Engine Optimization，让内容能被 ChatGPT / Perplexity / Gemini / Google AI Overviews / Claude 等**答案引擎抓取、引用、推荐**。
> 传统 SEO 争「排名」，GEO 争「被 AI 引用为答案来源」。
> 参考实现：MkFast（TanStarter）模板 `G:\Projects\seo+geo模板项目\mkfast-template`。

---

## 一、可被 AI 爬虫抓取（门禁先打开）

- `robots.txt` 里**显式允许 AI 爬虫**，而不是默认只写 `User-agent: *`。
- 模板做法（`src/routes/robots[.]txt.ts:12-25`）：为 12 个 AI 爬虫各写一个分组，显式 `Allow: /`：

```
GPTBot / OAI-SearchBot / ChatGPT-User / ClaudeBot / Claude-User /
PerplexityBot / Google-Extended / Applebot-Extended /
Bytespider / CCBot / DuckAssistBot / cohere-ai
```

- 知识点：**AI 爬虫和普通搜索引擎爬虫是两套 user-agent**。想进 ChatGPT / Perplexity / Gemini 的语料，就得单独给它们开门；同时私有路径（/auth /admin 等）对它们同样 Disallow。

---

## 二、给机器「干净的入口」（llms.txt / Markdown / RSS）

AI 抓取器预算有限，要给它**结构化、去噪、省 token** 的入口。

### 1. llms.txt（站点目录）
- 模板实现（`src/routes/llms[.]txt.ts`）：输出 `站点名 + 摘要 + 分节目录`，分四节：
  - Machine-readable endpoints（sitemap / llms-full / RSS / Markdown 原文 / robots）
  - Core pages（核心页 + 每个语言版本的 URL）
  - Blog（文章标题 + 链接 + 日期 + 描述）
  - Legal（条款/隐私/协议）
- 知识点：`llms.txt` 相当于「给 AI 的 README」，一个文件讲清「这是什么站、有哪些页、去哪抓全文」。**语言版本直接展开成 URL 列出来**，AI 不用解析 hreflang。

### 2. llms-full.txt（全文聚合）
- 把所有 `content/**/*.md` 原文聚合到一个文件。
- 知识点：一次抓全，省抓取预算，适合中小站点让 AI 直接读完。

### 3. Markdown 原文端点
- 每篇文章提供一个 `text/markdown` 原始端点（如 `/markdown/blog/getting-started`）。
- 知识点：**给 LLM 最干净的输入**——去掉 HTML/CSS/JS 噪声，只留正文，最容易被准确引用。

### 4. RSS
- `/blog/rss.xml` 带 `pubDate / updated`。
- 知识点：RSS 是「更新分发」通道，聚合器和 AI 都吃，能更快发现新内容。

---

## 三、结构化数据（JSON-LD）

JSON-LD 让 AI 无需「猜」就能理解实体和关系。

### 1. 实体对账（Entity Reconciliation）
- `Organization` 全站只发一次，其它实体用 `@id` 引用（`src/lib/jsonld.ts:25-31`）。
- 知识点：`@id` 把零散的结构化数据**对账成同一个品牌**，AI 能确认「这些内容来自同一个组织」。

### 2. 文章信号（`jsonld.ts:170-201`）
- `BlogPosting` 带：`datePublished / dateModified / wordCount / keywords / articleSection / author / publisher`。
- 知识点：答案引擎在意「**新鲜度、字数、分类、作者、发布者**」——这些字段决定内容是否值得被引用。

### 3. FAQPage 同源
- `FAQPage` 的 JSON-LD 与页面**可见**的 Q&A 用同一份数据生成（`jsonld.ts:126-145`）。
- 知识点：结构化数据不能和页面内容「漂移」，否则被判欺诈。

### 4. sameAs 社交档案
- `Organization` 带 `sameAs`（GitHub/Twitter/Discord 等）。
- 知识点：sameAs 帮 AI **跨平台确认品牌身份**，增强实体可信度。

---

## 四、内容可引用性（Citability）

GEO 的核心是让内容「**好被引用**」。

### 1. 标题可深链（锚点）
- `src/lib/markdown.ts` 用 `rehypeSlug` + `rehypeAutolinkHeadings`：每个标题自动加 ID 和锚点。
- 知识点：AI 引用时能**指到具体段落**（带 `#锚点` 的链接），比整页引用更精准、更可信。

### 2. 答案前置
- 段首直接给结论 / TL;DR，不要铺垫。
- 知识点：答案引擎会抽取「最像答案」的句子，结论放开头被抽取概率最高。

### 3. 结构化写作
- H2 用**问句**、每段**自包含**、用**表格对比**、列表化。
- 知识点：问答结构（问句标题 + 直接答案）最容易被 AI 当作 FAQ / 答案片断引用。

### 4. front-matter 元数据
- `content-collections.ts` 的 schema 支持 `summary / faq / updated / keywords / author`。
- 知识点：把「摘要、FAQ、更新时间、关键词」结构化，页面渲染和 JSON-LD **同一来源**，既服务人又服务机器。

---

## 五、新鲜度与可信度信号

- `updated` / `dateModified` / sitemap `lastmod`：告诉 AI「内容还在维护」。
- `wordCount`：字数信号。
- `author`（Person 或 Organization）：作者实体。
- 知识点：AI 倾向引用「**持续更新、有明确作者/机构、信息完整**」的内容；陈年无更新的内容容易被跳过。

---

## 六、度量与归因（否则全是玄学）

- 归因 **AI 引荐来源**：`chatgpt.com / perplexity.ai / gemini.google.com / copilot / 豆包` 等。
- 知识点：不统计「流量里有多少来自 AI 引用」，就不知道 GEO 投入有没有回报；这是 GEO 的「数据闭环」。

---

## 七、GEO vs SEO 速览

| 维度 | SEO（传统搜索） | GEO（答案引擎） |
|---|---|---|
| 目标 | 排名靠前、拿点击 | 被 AI 引用为答案来源 |
| 用户代理 | Googlebot / Bingbot | GPTBot / ClaudeBot / PerplexityBot… |
| 入口 | sitemap.xml / robots.txt | llms.txt / Markdown / RSS |
| 内容 | 关键词覆盖 | 答案前置、问句结构、可引用段落 |
| 结构化 | 富摘要（FAQ/Review） | JSON-LD 实体对账 + 文章信号 |
| 度量 | 排名 / 点击 / 展现 | AI 引荐来源归因 |

---

## 八、GEO 网站验收清单

- [ ] robots.txt 显式允许 AI 爬虫（GPTBot / ClaudeBot / PerplexityBot…）
- [ ] 提供 llms.txt（站点定位 + 核心页 + 文章清单）
- [ ] 提供 llms-full.txt（全文聚合）或 Markdown 原文端点
- [ ] 提供 RSS（带 pubDate/updated）
- [ ] JSON-LD：Organization（@id + sameAs）、文章（日期/字数/作者）、FAQPage 同源
- [ ] 标题可深链（heading ID + 锚点）
- [ ] 内容答案前置、H2 问句、段落自包含、表格对比
- [ ] front-matter 带 summary / updated / keywords / author
- [ ] 有 AI 引荐来源归因
