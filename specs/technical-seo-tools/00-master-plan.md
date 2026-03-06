# RankPrompt Technical SEO Tools Suite -- Master Plan

## Vision

Build a comprehensive technical SEO toolkit inside RankPrompt (app.rankprompt.com) that goes beyond traditional site auditing by deeply integrating **AEO (Answer Engine Optimization)** signals. Every tool should answer two questions: "Is this technically correct for Google?" AND "Is this optimized for AI engines (ChatGPT, Perplexity, Claude, Gemini)?"

This positions RankPrompt as the only platform where technical SEO auditing and AI visibility tracking live under one roof.

---

## Architecture Overview

```
+----------------------------------------------------------+
|                    Frontend (Bolt.new)                     |
|  /seo-audits/tools/:toolSlug                             |
|  Shared: URLInput, ResultsPanel, ScoreCard, ExportBtn    |
+---------------------------+------------------------------+
                            |
                     REST API calls
                            |
+---------------------------v------------------------------+
|              Supabase Edge Functions                      |
|  /functions/v1/seo-tool-{name}                           |
|  Each tool = 1 Edge Function                             |
|  Shared libs: html-fetcher, schema-vocab, dns-resolver   |
+---------------------------+------------------------------+
                            |
              +--------------+--------------+
              |              |              |
        +-----v----+  +-----v----+  +------v-----+
        | Supabase |  | External |  | n8n        |
        | Tables   |  | APIs     |  | Workflows  |
        | (results |  | (DNS,    |  | (scheduled |
        |  history)|  |  SSL,    |  |  scans,    |
        +----------+  |  PageSpd)|  |  alerts)   |
                      +----------+  +------------+
```

### Shared Infrastructure

**HTML Fetcher Service** -- Used by most tools. A single Edge Function or Cloud Run service that:
1. Accepts a URL
2. Fetches the raw HTML (with proper User-Agent, timeout, redirect following)
3. Optionally renders JS via headless Chrome (for SPAs)
4. Returns: raw HTML, final URL (after redirects), status code, response headers, timing
5. Caches results for 15 minutes (keyed by URL + render mode)

**Why a shared fetcher?** When a user runs multiple tools on the same URL, we fetch once and fan out. Saves time and avoids getting rate-limited.

```typescript
// Shared type used across all tools
interface FetchResult {
  url: string;           // Original URL
  finalUrl: string;      // After redirects
  statusCode: number;
  headers: Record<string, string>;
  html: string;
  timing: {
    dns: number;
    connect: number;
    ttfb: number;
    download: number;
    total: number;
  };
  redirectChain: Array<{ url: string; status: number }>;
  fetchedAt: string;     // ISO timestamp
}
```

---

## Tool Index

### Tier 1 -- Core Technical SEO (Ship First)

| # | Tool | Spec File | Priority | Complexity |
|---|------|-----------|----------|------------|
| 1 | Schema Markup Validator | `01-schema-validator.md` | P0 | High |
| 2 | Robots.txt Analyzer | `02-robots-txt-analyzer.md` | P0 | Medium |
| 3 | Sitemap Validator | `03-sitemap-validator.md` | P0 | Medium |
| 4 | Meta Tag Analyzer | `04-meta-tag-analyzer.md` | P0 | Medium |
| 5 | HTTP Header Inspector | `05-http-header-inspector.md` | P1 | Low |
| 6 | Redirect Chain Checker | `06-redirect-chain-checker.md` | P1 | Medium |
| 7 | Canonical URL Checker | `07-canonical-checker.md` | P1 | Medium |

### Tier 2 -- AEO-Specific (RankPrompt Differentiator)

| # | Tool | Spec File | Priority | Complexity |
|---|------|-----------|----------|------------|
| 8 | AI Bot Access Checker | `08-ai-bot-access-checker.md` | P0 | Medium |
| 9 | llms.txt Validator & Generator | `09-llms-txt-validator.md` | P0 | Medium |
| 10 | Content Citability Scorer | `10-content-citability-scorer.md` | P0 | High |
| 11 | AEO Readiness Score | `11-aeo-readiness-score.md` | P0 | High |

### Tier 3 -- Advanced SEO

| # | Tool | Spec File | Priority | Complexity |
|---|------|-----------|----------|------------|
| 12 | Heading Structure Analyzer | `12-heading-structure-analyzer.md` | P1 | Low |
| 13 | Image SEO Audit | `13-image-seo-audit.md` | P1 | Low |
| 14 | Internal Link Analyzer | `14-internal-link-analyzer.md` | P2 | High |
| 15 | Open Graph / Social Preview | `15-open-graph-social-preview.md` | P1 | Medium |
| 16 | SSL/TLS Security Check | `16-ssl-security-check.md` | P2 | Medium |
| 17 | Hreflang Validator | `17-hreflang-validator.md` | P2 | Medium |
| 18 | Duplicate Content Detector | `18-duplicate-content-detector.md` | P2 | High |

---

## Database Schema (Shared)

All tool results share a common table structure for historical tracking.

```sql
-- Core results table (all tools write here)
CREATE TABLE seo_tool_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  brand_id UUID REFERENCES brands(id) ON DELETE CASCADE,
  tool_slug TEXT NOT NULL,           -- 'schema-validator', 'robots-analyzer', etc.
  input_url TEXT NOT NULL,
  device TEXT DEFAULT 'desktop',     -- 'mobile' | 'desktop'
  score INTEGER,                     -- 0-100 unified score (nullable for tools without scores)
  status TEXT DEFAULT 'completed',   -- 'pending' | 'running' | 'completed' | 'failed'
  results JSONB NOT NULL,            -- Tool-specific results payload
  issues JSONB DEFAULT '[]',         -- Array of { severity, code, message, details }
  metadata JSONB DEFAULT '{}',       -- Timing, fetch info, etc.
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Index for fast lookups
CREATE INDEX idx_seo_results_user_tool ON seo_tool_results(user_id, tool_slug, created_at DESC);
CREATE INDEX idx_seo_results_brand_tool ON seo_tool_results(brand_id, tool_slug, created_at DESC);
CREATE INDEX idx_seo_results_url ON seo_tool_results(input_url, tool_slug, created_at DESC);

-- Unified issue severity enum used across all tools
-- 'critical' = blocks indexing or AI visibility
-- 'warning'  = degrades performance or misses opportunity
-- 'info'     = suggestion or informational
-- 'passed'   = check passed successfully

-- RLS: Users can only see their own results
ALTER TABLE seo_tool_results ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own results" ON seo_tool_results
  FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users insert own results" ON seo_tool_results
  FOR INSERT WITH CHECK (auth.uid() = user_id);
```

---

## Frontend Architecture

### Route Structure
```
/seo-audits/
  overview/              -- Existing: summary dashboard
  lighthouse/            -- Existing: Lighthouse scores
  technical/             -- Existing: technical audit
  competitors/           -- Existing: competitor comparison
  tools/                 -- NEW: tools hub landing page
    schema-validator/
    robots-analyzer/
    sitemap-validator/
    meta-tag-analyzer/
    header-inspector/
    redirect-checker/
    canonical-checker/
    ai-bot-checker/
    llms-txt-validator/
    citability-scorer/
    aeo-readiness/
    heading-analyzer/
    image-seo/
    link-analyzer/
    social-preview/
    ssl-check/
    hreflang-validator/
    duplicate-detector/
```

### Shared Components

```typescript
// URLInput -- Reusable URL input with validation
interface URLInputProps {
  onSubmit: (url: string) => void;
  placeholder?: string;
  allowRawInput?: boolean;  // For schema validator (paste HTML/JSON-LD)
  loading?: boolean;
}

// ScoreCard -- Circular score display (like existing Lighthouse scores)
interface ScoreCardProps {
  score: number;          // 0-100
  label: string;
  color?: 'green' | 'yellow' | 'red';  // Auto from score if not provided
  subtitle?: string;
}

// IssueList -- Expandable list of issues/checks
interface IssueListProps {
  issues: Array<{
    severity: 'critical' | 'warning' | 'info' | 'passed';
    title: string;
    description: string;
    details?: string;       // Expandable
    learnMoreUrl?: string;
    aeoImpact?: string;     // "This affects AI citation likelihood"
  }>;
  groupBy?: 'severity' | 'category';
}

// HistoryPanel -- Show previous scans for this URL
interface HistoryPanelProps {
  toolSlug: string;
  url: string;
  results: SeoToolResult[];
}

// ExportButton -- PDF/CSV export of results
interface ExportButtonProps {
  toolSlug: string;
  resultId: string;
  formats: ('pdf' | 'csv' | 'json')[];
}
```

### UI Pattern (Consistent Across All Tools)

```
+-----------------------------------------------+
| [Tool Name]                    [History] [?]   |
| Brief description of what this tool checks     |
+-----------------------------------------------+
| URL: [________________________] [Analyze]      |
|   OR                                           |
| Paste HTML/code: [textarea]    [Validate]      |
+-----------------------------------------------+
|                                                |
| Score: [87/100]    Checks: 12 passed, 3 issues |
|                                                |
| +-------------------------------------------+ |
| | Tab: All | Critical (1) | Warnings (2) |  | |
| +-------------------------------------------+ |
| | [x] Title tag present            PASSED   | |
| | [!] Missing FAQ schema          WARNING   | |
| | [!!] Robots blocks GPTBot      CRITICAL   | |
| |     > Details expand here...              | |
| |     > AEO Impact: AI engines cannot...    | |
| +-------------------------------------------+ |
|                                                |
| [Export PDF] [Export JSON] [Run Again]          |
+-----------------------------------------------+
| Previous Scans                                  |
| Mar 5: 87/100 | Feb 28: 72/100 | Feb 21: 65   |
+-----------------------------------------------+
```

---

## Integration Points with Existing Features

1. **Technical Audit Tab** -- Tools feed into the existing technical audit score. When a user runs schema-validator, it updates the "Schema" tab in Technical Audit.

2. **AI Visibility Tab** -- AI Bot Checker, llms.txt Validator, and AEO Readiness Score all feed the existing AI Visibility section.

3. **Competitors** -- Allow running any tool against competitor URLs. Show side-by-side comparison.

4. **Scheduled Reports** -- Allow scheduling any tool to run weekly/monthly via n8n workflows. Alert on score drops.

5. **Agent Mode** -- The AI agent should be able to invoke any tool: "Check the schema markup on example.com" triggers the schema validator and returns results conversationally.

6. **Credits System** -- Each tool run costs credits. Suggested pricing:
   - Basic tools (meta tags, headers, robots.txt): 1 credit
   - Medium tools (schema, sitemap, redirects): 2 credits
   - Heavy tools (citability, AEO readiness, full page render): 5 credits

---

## Implementation Order

### Phase 1 (Week 1-2): Foundation + Quick Wins
1. Shared HTML Fetcher service
2. Shared database schema + RLS
3. Shared frontend components (URLInput, ScoreCard, IssueList)
4. Meta Tag Analyzer (simplest, validates the pattern)
5. Robots.txt Analyzer
6. HTTP Header Inspector

### Phase 2 (Week 3-4): Core SEO Tools
7. Schema Markup Validator
8. Sitemap Validator
9. Redirect Chain Checker
10. Canonical URL Checker
11. Heading Structure Analyzer

### Phase 3 (Week 5-6): AEO Differentiators
12. AI Bot Access Checker (expand existing)
13. llms.txt Validator & Generator
14. Content Citability Scorer
15. AEO Readiness Score (composite)

### Phase 4 (Week 7-8): Advanced + Polish
16. Open Graph / Social Preview
17. Image SEO Audit
18. Internal Link Analyzer
19. SSL/TLS Security Check
20. Hreflang Validator
21. Duplicate Content Detector
22. Export + Scheduled Scans + Agent Mode integration

---

## Tech Notes for Developer

- **Edge Functions vs Cloud Run**: Use Supabase Edge Functions for most tools (they're fast, close to the DB, and free on your plan). Use Cloud Run only for tools that need headless Chrome (citability scorer, duplicate content detector).
- **Cheerio vs JSDOM vs linkedom**: Use `cheerio` for HTML parsing. It's the fastest and works in Deno (Edge Functions). Avoid JSDOM -- too heavy.
- **Schema.org vocabulary**: Download and cache `https://schema.org/version/latest/schemaorg-current-https.jsonld`. Store in Supabase Storage or as a static asset. Refresh weekly via cron.
- **DNS lookups**: Use the `dns` module in Node/Deno for SSL checks, redirect resolution, etc.
- **Rate limiting**: Implement per-user rate limiting (10 scans/minute) to prevent abuse. Check against `user_id` in the results table.
- **Error handling**: Every tool must handle: timeout (10s default), DNS failure, SSL error, robots.txt blocking our crawler, empty response, malformed HTML. Return a structured error, never crash.
- **User-Agent**: Use `RankPromptBot/1.0 (+https://rankprompt.com/bot)` for all fetches. Respect robots.txt for the target site (don't crawl if blocked).
