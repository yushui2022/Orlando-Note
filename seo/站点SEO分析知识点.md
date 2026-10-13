# 站点 SEO 分析知识点

> 分析对象：MkFast（TanStarter）模板 — `G:\Projects\seo+geo模板项目\mkfast-template`
> 核心结论：这套模板「天生易被搜索引擎和 AI 答案引擎收录」，因为它把 SEO 做成了**分层、可复用、自动化**的工程体系，而不是每个页面手写 meta。

---

## 一、地基：SSR（服务端渲染）

- TanStack Start + Cloudflare Workers，HTML 在**服务端渲染完成**后才返回。
- 爬虫（尤其不执行 JS 的爬虫）拿到的是一份**完整正文 HTML**，而不是空壳 SPA。
- 知识点：纯客户端渲染（SPA）内容在 JS 执行后才出现，爬虫可能只抓到空 `<div id="root">`；**SSR 让每个 URL 都有一份可直接解析的 HTML**，这是被收录的前提。

---

## 二、Head 元数据：统一入口 + 兜底

代码：`src/lib/seo.ts` + `src/routes/__root.tsx`

### 1. 统一 `seo()` 入口（seo.ts:22-64）
- 每个页面 `head()` 只调 `seo(path, { title, description })`，返回标准 `{ meta, links }`。
- 知识点：把「标题/描述/canonical/OG/Twitter」**收口到一个函数**，改一处全站生效，避免每个页面漏写或不一致。

### 2. canonical（首选网址）
- `seo()` 返回 `{ rel: 'canonical', href: url }`，用 `getCanonicalUrlForLocale()` 拼绝对地址。
- 知识点：canonical 告诉搜索引擎「这个页面的唯一官方 URL」，避免同一内容因多 URL（参数/大小写/多语言）被判定为重复内容。

### 3. 元数据全家桶（seo.ts:66-129 的 `metadata()`）
- 基础：`title`、`description`、`keywords`
- Open Graph：`og:type / og:site_name / og:locale / og:locale:alternate / og:title / og:description / og:url / og:image`
- Twitter Card：`twitter:title / twitter:site / twitter:description / twitter:url / twitter:card=summary_large_image / twitter:image`
- 知识点：**OG 管社交分享预览**（微信/FB/LinkedIn），**Twitter Card 管 X/Twitter 卡片**；用 `summary_large_image` 才会展示大图。

### 4. root 兜底（__root.tsx:37-120）
- root 的 `head()` 给全局默认：`charset=utf-8`、`viewport`、`theme-color`、默认 OG/Twitter、favicon、`manifest`，以及**站点级 Organization JSON-LD**。
- 知识点：页面即使忘记写 `seo()` 也有兜底 meta（404/错误页不裸奔）；子页面 head 会**覆盖** root 默认值。

---

## 三、多语言 SEO：hreflang 体系（重点）

### 1. hreflang alternate + x-default（seo.ts:40-58）
- 本地化页面输出 `rel="alternate" hreflang="en"`、`hreflang="zh-CN"` 和 `hreflang="x-default"`。
- 知识点：hreflang 告诉 Google「这页有其它语言版本」，避免把中文页当英文页排名；`x-default` 是「语言无法匹配时的默认页」。

### 2. 两个易错点（seo.ts:84-87）
- `<html lang>` 和 hreflang 用 **BCP 47 连字符**（en、zh-CN）；
- OG 的 `og:locale` 用 **下划线**（en_US、zh_CN）。
- 知识点：格式写错会丢失多语言信号。

### 3. sitemap 里也带 hreflang（sitemap.ts:46-54）
- 每个 `<url>` 内嵌 `<xhtml:link rel="alternate" hreflang="...">` 和 `x-default`。
- 知识点：**sitemap 里的 hreflang 是最权威的多语言声明**，帮 Google 建立各语言版本的对应关系。

---

## 四、结构化数据：JSON-LD

代码：`src/lib/jsonld.ts`

### 1. 用 `@id` 做「实体对账」（jsonld.ts:25-31, 56, 81）
- `Organization` 全站只发一次（root scripts），其它实体用 `publisher: { "@id": ".../#organization" }` 引用。
- 知识点：JSON-LD 用 `@id` 复用实体，搜索引擎能把零散的结构化数据**对账成同一个品牌/组织**，而不是一堆孤立对象。

### 2. 常用实体（jsonld.ts）
- `Organization`（含 logo + sameAs 社交档案）→ 品牌识别
- `WebSite`（inLanguage、publisher 指向 Organization）
- `WebPage / AboutPage / ContactPage`（isPartOf、publisher）
- `BreadcrumbList` 面包屑
- `FAQPage`（与页面**可见** Q&A 同源，防止结构化数据与内容漂移）
- `BlogPosting`：`datePublished / dateModified / wordCount / keywords / articleSection / author`（jsonld.ts:170-201）
- 知识点：文章类要带「**新鲜度(updated)、字数、分类、作者**」这些答案引擎在意的信号。

---

## 五、机器入口：robots.txt / sitemap.xml / manifest

### 1. 动态 sitemap.xml（sitemap[.]xml.ts）
- 静态页手工列出 + 博客由 `getSortedPosts(locale)` **自动收集**。
- 每个 URL 带 `lastmod / changefreq / priority`；博客自动带 `lastmod`（取文章 date）。
- 知识点：sitemap 是「给搜索引擎的目录」，新内容进 sitemap 能更快被发现；`lastmod` 帮助判断更新频率。

### 2. 动态 robots.txt（robots[.]txt.ts）
- `User-agent: * Allow: /` + 私有路径 Disallow（/auth /admin /settings /dashboard，含多语言变体）。
- **AI 爬虫分段**：GPTBot / OAI-SearchBot / ClaudeBot / PerplexityBot / Google-Extended / Bytespider 等 12 个，每个显式 `Allow: /`。
- 声明 `Sitemap:` 和 `# llms.txt:`。
- 知识点：robots.txt 是爬虫的「门禁」；**显式对 AI 爬虫 Allow**，是让站点进入 ChatGPT/Perplexity/Gemini 等答案引擎的入场券（GEO）。

### 3. manifest.json + favicon
- root head 挂 manifest 和多尺寸 favicon / apple-touch-icon（__root.tsx:92-112）。
- 知识点：manifest 影响 PWA 与移动端体验信号；favicon 影响品牌与搜索结果里的展示。

---

## 六、内容层：Content Collections

代码：`content-collections.ts`

- 3 个集合：blog / pages / changelog，front-matter 用 **Zod schema 校验**（title/description/date/category/image/updated/keywords/author…）。
- 文件命名 `*.en.md` / `*.zh.md` 自动识别语言（`getLocaleSlug`）。
- 知识点：**内容与代码解耦**——丢一个 `.md` 进 `content/blog/` 就自动进列表、详情页、sitemap、JSON-LD，不用改代码。这是「持续产出内容」站点的核心提效点。

---

## 七、GEO（生成式引擎优化）——比 SEO 更进一步

这套模板已为「AI 搜索引用」预留入口。知识点：

1. **llms.txt / llms-full.txt**：给 AI 抓取器的「站点目录 + 全文聚合」，省抓取预算。
2. **RSS**：更新分发给聚合器和 AI。
3. **Markdown 原文端点**（`text/markdown`）：给 LLM 最干净输入，去掉 HTML 噪声。
4. **AI 爬虫分段 Allow**：明确「允许 AI 引用」。
5. **度量层**：AI 引荐来源归因（chatgpt.com / perplexity.ai / gemini…），否则前面全是玄学。

---

## 八、易踩的坑

1. `/pricing` 支付未开启时 307 跳首页 → 同时从 sitemap 消失（sitemap[.]xml.ts:40-42 由 `websiteConfig.payment.enable` 控制）。上线前要么开支付，要么别在 sitemap 留死链。
2. XML 必须转义 `& < > " '`（sitemap.ts:18-28），否则 sitemap 解析失败。
3. FAQPage 的 JSON-LD 必须与页面**可见** Q&A 同源（jsonld.ts:126-145），否则会被判「结构化数据欺诈」。
4. AI demo 页（/ai）**无鉴权无限流**（docs/ai.md:68）——SEO 引流可以，但公开会被人刷付费额度，需加限流/登录墙。

---

## 九、验收清单（一个站点「易被收录」应具备什么）

- [ ] SSR / 静态渲染，爬虫能拿到完整 HTML
- [ ] 每个页面唯一 title / description / canonical
- [ ] OG + Twitter Card 齐全
- [ ] 多语言页有 hreflang + x-default（页面和 sitemap 都带）
- [ ] JSON-LD：Organization + WebSite + 页面类型 + 文章类型（含日期/字数/作者）
- [ ] 动态 sitemap.xml（自动收内容、带 lastmod）
- [ ] robots.txt（Disallow 私有区、显式 Allow AI 爬虫、声明 sitemap）
- [ ] 内容与代码解耦（Markdown 进 sitemap 自动化）
- [ ] 若要 GEO：llms.txt / RSS / Markdown 原文 + AI 引荐度量
