# Tool 11: AEO Readiness Score (Composite)

## What This Replicates
- Nothing (novel RankPrompt IP)
- Conceptually similar to Lighthouse's composite scoring but for AI/AEO

## What It Does
This is the master AEO tool. It runs ALL other tools and produces a single, comprehensive "AEO Readiness Score" for a page or site. Think of it as "Lighthouse for AI engines." It aggregates results from schema validation, robots.txt analysis, AI bot access, llms.txt, content citability, meta tags, and headers into one unified score with a clear action plan.

This should be the flagship tool in RankPrompt's toolkit, the one that gets shared and talked about.

---

## How It Works

```
User enters URL
      |
      v
+-----+------+------+------+------+------+------+
| Schema | Robots | AI Bot | llms | Meta | Head | Content |
| Valid. | Analyz | Access | .txt | Tags | ers  | Citab.  |
+-----+------+------+------+------+------+------+
      |      |      |      |      |      |      |
      v      v      v      v      v      v      v
+--------------------------------------------------+
|         AEO Readiness Score Engine                |
|  Weights, normalizes, and aggregates all results  |
+--------------------------------------------------+
      |
      v
+--------------------------------------------------+
| Score: 67/100  Grade: C+                          |
| Category scores + prioritized action plan         |
+--------------------------------------------------+
```

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-aeo-readiness
```

### Request
```json
{
  "url": "https://example.com/blog/best-crm-software-2024",
  "options": {
    "checkHomepage": true,
    "targetQuery": "what is the best CRM software",
    "competitor": "https://competitor.com/blog/best-crm"
  }
}
```

### Response
```json
{
  "score": 67,
  "grade": "C+",
  "url": "https://example.com/blog/best-crm-software-2024",
  "scanDuration": 8.2,
  "categories": {
    "aiAccess": {
      "score": 85,
      "weight": 25,
      "weightedScore": 21.25,
      "label": "AI Crawler Access",
      "description": "Can AI engines crawl and access your content?",
      "status": "good",
      "highlights": [
        "GPTBot: allowed",
        "ClaudeBot: allowed",
        "PerplexityBot: blocked (robots.txt)"
      ],
      "topIssue": "PerplexityBot is blocked, removing access for 100M+ Perplexity users"
    },
    "structuredData": {
      "score": 45,
      "weight": 20,
      "weightedScore": 9.0,
      "label": "Structured Data & Schema",
      "description": "Do you have schema markup that AI engines can extract?",
      "status": "needs_work",
      "highlights": [
        "Article schema: present",
        "FAQPage schema: missing",
        "BreadcrumbList: present",
        "Speakable: missing"
      ],
      "topIssue": "No FAQ schema despite having FAQ-style content. This is a major missed opportunity."
    },
    "contentCitability": {
      "score": 58,
      "weight": 25,
      "weightedScore": 14.5,
      "label": "Content Citability",
      "description": "How likely are AI engines to cite your content?",
      "status": "needs_work",
      "highlights": [
        "Definition blocks: 1/3 found",
        "Direct answer: missing",
        "Statistics: 4 found (2 unsourced)",
        "Expert quotes: 0 found"
      ],
      "topIssue": "No direct answer to the target query in the first 200 words"
    },
    "technicalFoundation": {
      "score": 90,
      "weight": 15,
      "weightedScore": 13.5,
      "label": "Technical Foundation",
      "description": "Meta tags, headers, canonicals, and page health",
      "status": "excellent",
      "highlights": [
        "Meta description: present (good length)",
        "Canonical: self-referencing (correct)",
        "HTTPS: yes",
        "Mobile-friendly: yes"
      ],
      "topIssue": null
    },
    "aiDiscoverability": {
      "score": 55,
      "weight": 15,
      "weightedScore": 8.25,
      "label": "AI Discoverability",
      "description": "Can AI engines find and understand your site structure?",
      "status": "needs_work",
      "highlights": [
        "llms.txt: not found",
        "Sitemap: present, valid",
        "robots.txt: has issues",
        "Internal linking: moderate"
      ],
      "topIssue": "No llms.txt file to guide AI engines"
    }
  },
  "compositeCalculation": {
    "aiAccess": "85 x 0.25 = 21.25",
    "structuredData": "45 x 0.20 = 9.00",
    "contentCitability": "58 x 0.25 = 14.50",
    "technicalFoundation": "90 x 0.15 = 13.50",
    "aiDiscoverability": "55 x 0.15 = 8.25",
    "total": "66.50 -> 67"
  },
  "actionPlan": [
    {
      "rank": 1,
      "action": "Add a direct answer to your target query",
      "category": "contentCitability",
      "impact": "high",
      "effort": "low",
      "estimatedScoreGain": 8,
      "howTo": "In the first paragraph, state: 'The best CRM software in 2024 is [X] for most businesses.' AI engines cite pages that answer queries immediately."
    },
    {
      "rank": 2,
      "action": "Add FAQPage schema markup",
      "category": "structuredData",
      "impact": "high",
      "effort": "medium",
      "estimatedScoreGain": 7,
      "howTo": "Your page has 6 question-formatted headers. Wrap them in FAQPage schema (JSON-LD). AI engines love FAQ schema because it provides pre-formatted Q&A pairs."
    },
    {
      "rank": 3,
      "action": "Unblock PerplexityBot",
      "category": "aiAccess",
      "impact": "high",
      "effort": "low",
      "estimatedScoreGain": 5,
      "howTo": "Remove the PerplexityBot Disallow rule from robots.txt. Perplexity has 100M+ monthly users who could see your content."
    },
    {
      "rank": 4,
      "action": "Create llms.txt",
      "category": "aiDiscoverability",
      "impact": "medium",
      "effort": "medium",
      "estimatedScoreGain": 5,
      "howTo": "Use RankPrompt's llms.txt Generator to create this file. Place it at your site root."
    },
    {
      "rank": 5,
      "action": "Add expert quotes and source your statistics",
      "category": "contentCitability",
      "impact": "medium",
      "effort": "medium",
      "estimatedScoreGain": 4,
      "howTo": "Add 2-3 named expert quotes. Add source attribution ('according to Gartner 2023') to your unsourced statistics."
    }
  ],
  "potentialScore": 96,
  "competitorComparison": {
    "you": 67,
    "competitor": 78,
    "competitorUrl": "https://competitor.com/blog/best-crm",
    "competitorStrengths": [
      "Has FAQ schema (+10)",
      "Has direct answer in first paragraph (+8)",
      "Has llms.txt (+5)"
    ],
    "competitorWeaknesses": [
      "Blocks GPTBot (-10)",
      "No expert quotes (-5)"
    ]
  }
}
```

---

## Category Weights

```
AI Crawler Access:      25%  -- Can AI engines reach your content at all?
Content Citability:     25%  -- Will AI engines want to cite your content?
Structured Data:        20%  -- Can AI engines extract structured information?
Technical Foundation:   15%  -- Are the basics correct (meta, canonical, HTTPS)?
AI Discoverability:     15%  -- Can AI engines find and map your content?
```

These weights reflect AEO priorities: access and citability matter most, followed by structured data that AI can parse.

---

## Grade Scale

```
A+ : 95-100  "AI-first content. Maximum AI visibility."
A  : 90-94   "Excellent. AI engines love your content."
A- : 85-89   "Great. Minor optimizations available."
B+ : 80-84   "Good. A few gaps to close."
B  : 70-79   "Solid foundation with room to grow."
C+ : 60-69   "Decent but missing key AEO signals."
C  : 50-59   "Below average. Several important gaps."
D  : 40-49   "Needs significant work."
F  : 0-39    "Major issues. AI engines can't effectively use your content."
```

---

## Frontend: AEO Dashboard

### Hero Section
```
+----------------------------------------------------+
|     AEO READINESS SCORE                             |
|                                                     |
|        +--------+                                   |
|        |        |                                   |
|        |   67   |  Grade: C+                       |
|        |        |  "Decent but missing key AEO      |
|        +--------+   signals"                        |
|                                                     |
|  Potential: 96  (+29 with recommended changes)      |
+----------------------------------------------------+
```

### Category Bars
```
AI Crawler Access     [===============-----]  85%  Good
Content Citability    [====================]  58%  Needs Work
Structured Data       [==========-----------]  45%  Needs Work
Technical Foundation  [===================-]  90%  Excellent
AI Discoverability    [===========----------]  55%  Needs Work
```

### Action Plan (Ranked)
Cards sorted by impact/effort ratio, each showing:
- Estimated score gain
- Difficulty badge (Easy / Medium / Hard)
- Category it affects
- Step-by-step instructions
- "Fix This" button (links to relevant standalone tool)

### Competitor Comparison (Premium)
Side-by-side radar chart showing you vs competitor across all 5 categories.

---

## Integration

- **Auto-run on audit**: When a user runs a Technical Audit, offer to also run AEO Readiness
- **Historical tracking**: Store scores over time, show trend line
- **Scheduled reports**: Run weekly/monthly, alert on score changes > 10 points
- **Agent Mode**: "What's my AEO readiness for [page]?" triggers this tool
- **Export**: PDF report with all categories, action plan, and competitor comparison

---

## Scoring Precision

Each sub-tool's score feeds in directly:
```typescript
function calculateAEOReadiness(results: {
  aiAccess: number;        // From Tool 08
  structuredData: number;  // From Tool 01
  citability: number;      // From Tool 10
  technical: number;       // Average of Tools 04, 05, 07
  discoverability: number; // Average of Tools 02, 03, 09
}): number {
  return Math.round(
    results.aiAccess * 0.25 +
    results.structuredData * 0.20 +
    results.citability * 0.25 +
    results.technical * 0.15 +
    results.discoverability * 0.15
  );
}
```

---

## Implementation Notes

- This tool orchestrates all other tools. It should call them in parallel where possible.
- Use the shared HTML Fetcher to avoid fetching the same page multiple times.
- The first run is expensive (7+ sub-tools). Cache aggressively (15 min minimum).
- Competitor comparison doubles the cost (runs everything on both URLs). Make it premium.
- Start with a simpler v1 that just checks the 5 categories with lighter-weight checks, then progressively wire in the full sub-tools.
