# Tool 08: AI Bot Access Checker

## What This Replicates
- Dark Visitors (darkvisitors.com) -- AI crawler database
- This is partially built in RankPrompt's Technical Audit (see screenshot AI Visibility tab)
- No major competitor has a comprehensive standalone tool for this yet

## What It Does
This is RankPrompt's signature AEO tool. It analyzes a website's entire AI crawler policy: robots.txt rules for all known AI bots, HTTP headers that affect AI access (nosnippet, max-snippet), meta robots directives, and the emerging llms.txt standard. Provides a unified "AI Accessibility Score" with actionable recommendations.

This expands the existing AI Visibility tab in Technical Audit into a full standalone tool.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-ai-bot-checker
```

### Request
```json
{
  "url": "https://example.com",
  "checkPages": ["/", "/blog", "/products", "/about"],
  "options": {
    "checkRobotsTxt": true,
    "checkHeaders": true,
    "checkMetaRobots": true,
    "checkLlmsTxt": true,
    "checkAiTxt": true
  }
}
```

### Response
```json
{
  "score": 72,
  "domain": "example.com",
  "summary": {
    "totalBots": 18,
    "allowed": 14,
    "blocked": 3,
    "partiallyBlocked": 1,
    "noExplicitRules": 12,
    "aiAccessGrade": "B"
  },
  "bots": [
    {
      "name": "GPTBot",
      "organization": "OpenAI",
      "purpose": "Powers ChatGPT responses via web browsing and training",
      "importance": "critical",
      "status": "blocked",
      "sources": {
        "robotsTxt": {
          "status": "blocked",
          "rule": "Disallow: /",
          "line": 8,
          "userAgent": "GPTBot"
        },
        "httpHeader": {
          "status": "no_restriction",
          "xRobotsTag": null
        },
        "metaRobots": {
          "status": "no_restriction",
          "tag": null
        }
      },
      "impact": "HIGH: Blocking GPTBot prevents your content from appearing in ChatGPT answers. ChatGPT has 200M+ weekly users.",
      "recommendation": "Remove the GPTBot Disallow rule unless you have specific IP/training concerns. You can allow browsing while blocking training with: Allow: / and adding to ai.txt or machine-readable preference."
    },
    {
      "name": "ClaudeBot",
      "organization": "Anthropic",
      "purpose": "Powers Claude AI assistant responses",
      "importance": "high",
      "status": "allowed",
      "sources": {
        "robotsTxt": { "status": "allowed", "rule": null, "inheritsFrom": "*" },
        "httpHeader": { "status": "no_restriction" },
        "metaRobots": { "status": "no_restriction" }
      },
      "impact": "Allowed. Claude can access and cite your content.",
      "recommendation": null
    }
  ],
  "llmsTxt": {
    "found": false,
    "url": "https://example.com/llms.txt",
    "status": 404,
    "recommendation": "Create an llms.txt file to provide AI-optimized context about your site. See spec: llmstxt.org"
  },
  "aiTxt": {
    "found": false,
    "url": "https://example.com/ai.txt",
    "status": 404,
    "recommendation": "ai.txt is an emerging standard for declaring AI usage preferences. Consider creating one."
  },
  "pageChecks": [
    {
      "url": "/",
      "metaRobots": "index, follow",
      "xRobotsTag": null,
      "maxSnippet": null,
      "nosnippet": false,
      "noAiIssues": true
    },
    {
      "url": "/blog",
      "metaRobots": "index, follow, max-snippet:-1",
      "xRobotsTag": null,
      "maxSnippet": -1,
      "nosnippet": false,
      "noAiIssues": true
    }
  ],
  "issues": [ ... ],
  "actionPlan": [
    {
      "priority": 1,
      "action": "Unblock GPTBot in robots.txt",
      "impact": "Opens your site to ChatGPT's 200M+ weekly users",
      "difficulty": "easy",
      "howTo": "Remove these lines from robots.txt:\nUser-agent: GPTBot\nDisallow: /"
    },
    {
      "priority": 2,
      "action": "Create llms.txt file",
      "impact": "Helps AI engines understand your site structure and key content",
      "difficulty": "medium",
      "howTo": "See Tool 09 (llms.txt Validator) for generation instructions"
    },
    {
      "priority": 3,
      "action": "Remove max-snippet:0 from /pricing page",
      "impact": "Allows AI engines to excerpt your pricing information in answers",
      "difficulty": "easy",
      "howTo": "Change <meta name='robots' content='max-snippet:0'> to max-snippet:-1 or remove it"
    }
  ]
}
```

---

## Comprehensive AI Bot Database

This is the core data asset. Keep this updated as new AI crawlers emerge.

```typescript
interface AIBot {
  userAgent: string;
  organization: string;
  purpose: string;
  importance: 'critical' | 'high' | 'medium' | 'low';
  infoUrl: string;
  category: 'search' | 'assistant' | 'training' | 'data';
  monthlyUsers?: string;  // Approximate
}

const AI_BOTS: AIBot[] = [
  // CRITICAL -- These directly power consumer-facing AI products
  {
    userAgent: 'GPTBot',
    organization: 'OpenAI',
    purpose: 'Web crawling for ChatGPT training and retrieval',
    importance: 'critical',
    infoUrl: 'https://platform.openai.com/docs/bots',
    category: 'assistant',
    monthlyUsers: '200M+'
  },
  {
    userAgent: 'ChatGPT-User',
    organization: 'OpenAI',
    purpose: 'Real-time web browsing when ChatGPT users click "Browse"',
    importance: 'critical',
    infoUrl: 'https://platform.openai.com/docs/bots',
    category: 'search',
    monthlyUsers: '200M+'
  },
  {
    userAgent: 'ClaudeBot',
    organization: 'Anthropic',
    purpose: 'Web crawling for Claude training and tool use',
    importance: 'high',
    infoUrl: 'https://docs.anthropic.com/en/docs/web-crawler',
    category: 'assistant',
    monthlyUsers: '100M+'
  },
  {
    userAgent: 'PerplexityBot',
    organization: 'Perplexity AI',
    purpose: 'Real-time web search and citation for Perplexity answers',
    importance: 'critical',
    infoUrl: 'https://docs.perplexity.ai/docs/perplexitybot',
    category: 'search',
    monthlyUsers: '100M+'
  },
  {
    userAgent: 'Google-Extended',
    organization: 'Google',
    purpose: 'Gemini/AI Overviews training (separate from Googlebot indexing)',
    importance: 'critical',
    infoUrl: 'https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers',
    category: 'training',
    monthlyUsers: 'N/A (training)'
  },

  // HIGH -- Significant AI platforms
  {
    userAgent: 'Anthropic-AI',
    organization: 'Anthropic',
    purpose: 'General Anthropic web crawling',
    importance: 'high',
    infoUrl: 'https://docs.anthropic.com/en/docs/web-crawler',
    category: 'training'
  },
  {
    userAgent: 'Applebot-Extended',
    organization: 'Apple',
    purpose: 'Apple Intelligence and Siri AI training',
    importance: 'high',
    infoUrl: 'https://support.apple.com/en-us/119829',
    category: 'training',
    monthlyUsers: '1B+ Apple devices'
  },
  {
    userAgent: 'Meta-ExternalAgent',
    organization: 'Meta',
    purpose: 'Meta AI assistant training and browsing',
    importance: 'high',
    infoUrl: 'https://about.meta.com/ai-agent',
    category: 'assistant'
  },
  {
    userAgent: 'FacebookBot',
    organization: 'Meta',
    purpose: 'Meta AI training data collection',
    importance: 'high',
    infoUrl: 'https://developers.facebook.com/docs/sharing/bot',
    category: 'training'
  },

  // MEDIUM -- Notable AI platforms and data providers
  {
    userAgent: 'CCBot',
    organization: 'Common Crawl',
    purpose: 'Open web dataset used by many AI models for training',
    importance: 'medium',
    infoUrl: 'https://commoncrawl.org/ccbot',
    category: 'data'
  },
  {
    userAgent: 'Cohere-AI',
    organization: 'Cohere',
    purpose: 'Enterprise AI model training',
    importance: 'medium',
    infoUrl: 'https://cohere.com',
    category: 'training'
  },
  {
    userAgent: 'Amazonbot',
    organization: 'Amazon',
    purpose: 'Alexa AI and Amazon search',
    importance: 'medium',
    infoUrl: 'https://developer.amazon.com/amazonbot',
    category: 'assistant'
  },
  {
    userAgent: 'YouBot',
    organization: 'You.com',
    purpose: 'You.com AI search engine',
    importance: 'medium',
    infoUrl: 'https://about.you.com/youbot',
    category: 'search'
  },
  {
    userAgent: 'Bytespider',
    organization: 'ByteDance',
    purpose: 'TikTok/ByteDance AI and search',
    importance: 'medium',
    infoUrl: 'https://www.bytedance.com',
    category: 'training'
  },

  // LOW -- Niche or secondary
  {
    userAgent: 'OmgiliBot',
    organization: 'Webz.io',
    purpose: 'Web data collection and discussion mining',
    importance: 'low',
    infoUrl: 'https://webz.io',
    category: 'data'
  },
  {
    userAgent: 'Diffbot',
    organization: 'Diffbot',
    purpose: 'Structured web data extraction for AI',
    importance: 'low',
    infoUrl: 'https://www.diffbot.com',
    category: 'data'
  },
  {
    userAgent: 'ImagesiftBot',
    organization: 'Hive',
    purpose: 'Image and visual AI training',
    importance: 'low',
    infoUrl: 'https://hive.blog',
    category: 'training'
  },
  {
    userAgent: 'Timpibot',
    organization: 'Timpi',
    purpose: 'Decentralized search and AI',
    importance: 'low',
    infoUrl: 'https://timpi.io',
    category: 'data'
  },
  {
    userAgent: 'Kangaroo Bot',
    organization: 'Kangaroo LLM',
    purpose: 'AI model training data',
    importance: 'low',
    infoUrl: 'https://kangaroo-llm.com',
    category: 'training'
  }
];
```

---

## Grading System

```
AI Access Grade (A-F):

A  (90-100): All critical + high bots allowed, llms.txt present, no nosnippet
A- (85-89):  All critical bots allowed, minor issues
B+ (80-84):  Most bots allowed, 1-2 non-critical blocks
B  (70-79):  Some important bots blocked, but Googlebot + 1 AI bot allowed
C  (50-69):  Multiple important AI bots blocked
D  (30-49):  Most AI bots blocked, or nosnippet on key pages
F  (0-29):   All or nearly all AI bots blocked, plus nosnippet

Point allocation:
- Each critical bot allowed: +15
- Each high bot allowed: +8
- Each medium bot allowed: +4
- Each low bot allowed: +2
- llms.txt present: +10
- ai.txt present: +5
- No nosnippet on any checked page: +10
- max-snippet:-1 (unlimited) on key pages: +5
```

---

## Frontend: Bot Grid Display

```
+------------------------------------------------------+
| AI Bot Access Report           Grade: B  Score: 72   |
+------------------------------------------------------+
|                                                       |
| CRITICAL BOTS                                         |
| +--------+ +--------+ +--------+ +--------+          |
| |  [O]   | |  [O]   | |  [X]   | |  [O]   |          |
| | GPTBot | |ChatGPT | |Perplx  | |Google  |          |
| |ALLOWED | |ALLOWED | |BLOCKED | |ALLOWED |          |
| +--------+ +--------+ +--------+ +--------+          |
|                                                       |
| HIGH IMPORTANCE                                       |
| +--------+ +--------+ +--------+ +--------+          |
| |  [O]   | |  [X]   | |  [O]   | |  [O]   |          |
| |Claude  | |Apple   | |Meta    | |Anthro  |          |
| |ALLOWED | |BLOCKED | |ALLOWED | |ALLOWED |          |
| +--------+ +--------+ +--------+ +--------+          |
|                                                       |
| MEDIUM / LOW (collapsed, expandable)                  |
| 8/10 allowed, 2 blocked                              |
+------------------------------------------------------+
| Action Plan                                           |
| 1. [!] Unblock PerplexityBot     [How to fix ->]    |
| 2. [!] Unblock Applebot-Extended [How to fix ->]    |
| 3. [i] Create llms.txt           [Generate ->]      |
+------------------------------------------------------+
```

Clicking a bot card expands to show:
- Organization and purpose
- robots.txt rule details
- HTTP header directives
- Meta robots status
- Monthly user reach
- Impact explanation
- Fix instructions

---

## Integration with Existing Technical Audit

This tool should power the "AI Visibility" tab in the existing Technical Audit. When a user runs a Technical Audit, it automatically runs this checker and feeds results into the AI Visibility section. The standalone tool provides the deep-dive version.
