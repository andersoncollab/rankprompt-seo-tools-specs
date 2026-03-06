# RankPrompt Technical SEO & AEO Tools Suite

Comprehensive specifications for 18 technical SEO tools built into [RankPrompt](https://app.rankprompt.com). Every tool includes traditional SEO checks **plus** AEO (Answer Engine Optimization) analysis, making this the only toolkit that audits for both Google and AI engines (ChatGPT, Perplexity, Claude, Gemini) simultaneously.

---

## Tool Index

### Tier 1: Core Technical SEO

| # | Tool | What It Does | Replaces |
|---|------|-------------|----------|
| [01](specs/technical-seo-tools/01-schema-validator.md) | **Schema Markup Validator** | Extracts and validates JSON-LD, Microdata, and RDFa. Checks Google Rich Result eligibility + AI extractability. | validator.schema.org, Rich Results Test |
| [02](specs/technical-seo-tools/02-robots-txt-analyzer.md) | **Robots.txt Analyzer** | Parses robots.txt, validates syntax, tests URL blocking, and maps all AI crawler access rules. | Google robots.txt Tester |
| [03](specs/technical-seo-tools/03-sitemap-validator.md) | **Sitemap Validator** | Validates XML sitemaps and sitemap indexes. Checks URL limits, status codes, and staleness. | XML-Sitemaps.com |
| [04](specs/technical-seo-tools/04-meta-tag-analyzer.md) | **Meta Tag Analyzer** | Audits title, description, OG, Twitter Cards, canonical, viewport, and robots meta. Includes SERP preview. | Metatags.io, SEOptimer |
| [05](specs/technical-seo-tools/05-http-header-inspector.md) | **HTTP Header Inspector** | Analyzes response headers for SEO, security, and performance. Checks X-Robots-Tag, HSTS, CSP, compression. | SecurityHeaders.com |
| [06](specs/technical-seo-tools/06-redirect-chain-checker.md) | **Redirect Chain Checker** | Follows redirect chains, reports every hop with status codes and timing. Detects loops and long chains. | httpstatus.io |
| [07](specs/technical-seo-tools/07-canonical-checker.md) | **Canonical URL Checker** | Cross-validates HTML canonical, HTTP header, and sitemap URLs. Detects conflicts and chains. | ContentKing |

### Tier 2: AEO-Specific (RankPrompt Differentiators)

| # | Tool | What It Does | Competitive Advantage |
|---|------|-------------|----------------------|
| [08](specs/technical-seo-tools/08-ai-bot-access-checker.md) | **AI Bot Access Checker** | Unified AI crawler policy analysis across robots.txt, headers, and meta tags. Database of 18 AI bots with impact scoring. | Expands existing AI Visibility tab into a full standalone tool |
| [09](specs/technical-seo-tools/09-llms-txt-validator.md) | **llms.txt Validator & Generator** | Validates existing llms.txt files or generates one by crawling the site. | **First-mover. No competitor tool exists.** |
| [10](specs/technical-seo-tools/10-content-citability-scorer.md) | **Content Citability Scorer** | Scores how likely AI engines are to cite your content. Analyzes definition blocks, direct answers, statistics, expert quotes, and paragraph structure. | **Novel IP. Nobody else does this.** |
| [11](specs/technical-seo-tools/11-aeo-readiness-score.md) | **AEO Readiness Score** | Composite score aggregating all tools into a single "Lighthouse for AI engines" metric with a prioritized action plan. | **The flagship tool. Single score across 5 categories.** |

### Tier 3: Advanced SEO

| # | Tool | What It Does |
|---|------|-------------|
| [12](specs/technical-seo-tools/12-heading-structure-analyzer.md) | **Heading Structure Analyzer** | Validates H1-H6 hierarchy, keyword usage, and query-matching headers. |
| [13](specs/technical-seo-tools/13-image-seo-audit.md) | **Image SEO Audit** | Checks alt text, file sizes, modern formats, lazy loading, and descriptive filenames. |
| [14](specs/technical-seo-tools/14-internal-link-analyzer.md) | **Internal Link Analyzer** | Audits internal link structure, anchor text distribution, and broken links. |
| [15](specs/technical-seo-tools/15-open-graph-social-preview.md) | **Open Graph & Social Preview** | Visual previews for Facebook, Twitter/X, LinkedIn, Discord, and Slack. |
| [16](specs/technical-seo-tools/16-ssl-security-check.md) | **SSL/TLS Security Check** | Certificate validation, HTTPS redirect, HSTS, and mixed content detection. |
| [17](specs/technical-seo-tools/17-hreflang-validator.md) | **Hreflang Validator** | Validates hreflang implementation with return tag compliance and language code checking. |
| [18](specs/technical-seo-tools/18-duplicate-content-detector.md) | **Duplicate Content Detector** | Near-duplicate detection using MinHash/Jaccard similarity across internal pages. |

---

## Architecture

Start with the [Master Plan](specs/technical-seo-tools/00-master-plan.md) for:

- **Shared infrastructure** (HTML fetcher, database schema, frontend components)
- **Implementation phases** (4 phases over 8 weeks)
- **Integration points** with existing RankPrompt features (Technical Audit, AI Visibility, Agent Mode, Scheduled Reports)
- **Credit pricing** per tool tier
- **Tech stack notes** (Supabase Edge Functions, Cheerio, Cloud Run for heavy tools)

```
Frontend (Bolt.new)  -->  Supabase Edge Functions  -->  Supabase DB
                              |                         (results + history)
                         Shared HTML Fetcher
                              |
                     External APIs (DNS, SSL, PageSpeed)
                     n8n Workflows (scheduled scans)
```

## What Each Spec Contains

Every tool spec includes:

- **Purpose and what it replaces** (existing tools in the market)
- **API endpoint** with full request/response JSON schemas
- **Implementation code** (TypeScript) for core logic
- **Validation rules** with severity codes (critical / warning / info / passed)
- **AEO-specific checks** unique to RankPrompt
- **Scoring algorithm** (0-100 with weighted deductions)
- **Frontend UI mockups** (ASCII wireframes)
- **Edge cases** and error handling

## Build Order

| Phase | Weeks | Tools | Focus |
|-------|-------|-------|-------|
| 1 | 1-2 | Shared infra + Meta Tags + Robots.txt + Headers | Foundation + quick wins |
| 2 | 3-4 | Schema + Sitemap + Redirects + Canonical + Headings | Core SEO |
| 3 | 5-6 | AI Bot Checker + llms.txt + Citability + AEO Score | AEO differentiators |
| 4 | 7-8 | Social Preview + Image + Links + SSL + Hreflang + Duplicates | Advanced + polish |

---

Built by [Anderson Collaborative](https://andersoncollaborative.com) for [RankPrompt](https://rankprompt.com)
