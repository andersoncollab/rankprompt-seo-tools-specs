# Tool 02: Robots.txt Analyzer

## What This Replicates
- Google's robots.txt Tester (in Search Console)
- Merkle Robots.txt Validator
- TechnicalSEO.com robots.txt tools

## What It Does
Fetches, parses, and validates a site's robots.txt file. Checks syntax, identifies blocking issues, and critically for RankPrompt, analyzes **AI crawler access** (GPTBot, ClaudeBot, CCBot, etc.). Also tests whether specific URLs are allowed/blocked for specific user agents.

---

## User Flow

1. User enters a domain (we auto-append `/robots.txt`)
2. System fetches the robots.txt
3. Parses all directives (User-agent, Allow, Disallow, Sitemap, Crawl-delay, etc.)
4. Validates syntax
5. Highlights AI bot rules specifically
6. Allows user to test "Is URL X blocked for bot Y?"

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-robots-analyzer
```

### Request Body
```json
{
  "url": "https://example.com",
  "testUrls": ["/products/widget", "/api/data"],
  "testAgents": ["Googlebot", "GPTBot", "ClaudeBot"]
}
```

### Response
```json
{
  "score": 78,
  "robotsTxtUrl": "https://example.com/robots.txt",
  "status": {
    "httpStatus": 200,
    "size": 1247,
    "lastModified": "2024-11-15T00:00:00Z",
    "contentType": "text/plain"
  },
  "raw": "User-agent: *\nDisallow: /admin\n...",
  "parsed": {
    "groups": [
      {
        "userAgents": ["*"],
        "rules": [
          { "type": "disallow", "path": "/admin", "line": 2 },
          { "type": "disallow", "path": "/api/", "line": 3 },
          { "type": "allow", "path": "/api/public/", "line": 4 }
        ],
        "crawlDelay": null
      },
      {
        "userAgents": ["GPTBot"],
        "rules": [
          { "type": "disallow", "path": "/", "line": 8 }
        ],
        "crawlDelay": null
      }
    ],
    "sitemaps": [
      "https://example.com/sitemap.xml"
    ],
    "host": null
  },
  "issues": [
    {
      "code": "AI_BOT_FULLY_BLOCKED",
      "severity": "critical",
      "title": "GPTBot is fully blocked",
      "message": "Your robots.txt blocks GPTBot (OpenAI's crawler) from accessing your entire site. This prevents your content from appearing in ChatGPT responses.",
      "line": 8,
      "aeoImpact": "Blocking GPTBot means ChatGPT cannot crawl your site for training or retrieval. Your brand will not appear in ChatGPT answers based on direct crawling."
    },
    {
      "code": "NO_SITEMAP_DIRECTIVE",
      "severity": "warning",
      "title": "No sitemap directive found",
      "message": "Consider adding a Sitemap directive to help all crawlers discover your content."
    }
  ],
  "aiCrawlerSummary": {
    "GPTBot": { "status": "blocked", "reason": "Disallow: /", "line": 8 },
    "ChatGPT-User": { "status": "allowed", "reason": "No specific rules (falls through to *)" },
    "ClaudeBot": { "status": "allowed", "reason": "No specific rules" },
    "Anthropic-AI": { "status": "allowed", "reason": "No specific rules" },
    "CCBot": { "status": "allowed", "reason": "No specific rules" },
    "Cohere-AI": { "status": "allowed", "reason": "No specific rules" },
    "PerplexityBot": { "status": "allowed", "reason": "No specific rules" },
    "Google-Extended": { "status": "allowed", "reason": "No specific rules" },
    "Bytespider": { "status": "allowed", "reason": "No specific rules" },
    "OmgiliBot": { "status": "allowed", "reason": "No specific rules" },
    "Applebot-Extended": { "status": "allowed", "reason": "No specific rules" },
    "FacebookBot": { "status": "allowed", "reason": "No specific rules" }
  },
  "urlTests": [
    {
      "url": "/products/widget",
      "results": {
        "Googlebot": { "allowed": true, "matchedRule": null, "line": null },
        "GPTBot": { "allowed": false, "matchedRule": "Disallow: /", "line": 8 },
        "ClaudeBot": { "allowed": true, "matchedRule": null, "line": null }
      }
    }
  ]
}
```

---

## Parsing Logic

```typescript
interface RobotsGroup {
  userAgents: string[];
  rules: RobotsRule[];
  crawlDelay: number | null;
}

interface RobotsRule {
  type: 'allow' | 'disallow';
  path: string;
  line: number;
}

function parseRobotsTxt(content: string): {
  groups: RobotsGroup[];
  sitemaps: string[];
  errors: Array<{ line: number; message: string }>;
} {
  const lines = content.split('\n');
  const groups: RobotsGroup[] = [];
  const sitemaps: string[] = [];
  const errors: Array<{ line: number; message: string }> = [];

  let currentGroup: RobotsGroup | null = null;

  for (let i = 0; i < lines.length; i++) {
    const lineNum = i + 1;
    const raw = lines[i];
    const line = raw.split('#')[0].trim(); // Strip comments

    if (!line) continue;

    const colonIndex = line.indexOf(':');
    if (colonIndex === -1) {
      errors.push({ line: lineNum, message: `Invalid directive (no colon): "${raw.trim()}"` });
      continue;
    }

    const directive = line.substring(0, colonIndex).trim().toLowerCase();
    const value = line.substring(colonIndex + 1).trim();

    switch (directive) {
      case 'user-agent':
        if (!currentGroup || currentGroup.rules.length > 0) {
          // Start a new group
          currentGroup = { userAgents: [value], rules: [], crawlDelay: null };
          groups.push(currentGroup);
        } else {
          // Add to existing group (multiple user-agents before first rule)
          currentGroup.userAgents.push(value);
        }
        break;

      case 'disallow':
        if (!currentGroup) {
          errors.push({ line: lineNum, message: 'Disallow without User-agent' });
          break;
        }
        if (value) { // Empty disallow = allow all
          currentGroup.rules.push({ type: 'disallow', path: value, line: lineNum });
        }
        break;

      case 'allow':
        if (!currentGroup) {
          errors.push({ line: lineNum, message: 'Allow without User-agent' });
          break;
        }
        currentGroup.rules.push({ type: 'allow', path: value, line: lineNum });
        break;

      case 'sitemap':
        sitemaps.push(value);
        break;

      case 'crawl-delay':
        if (currentGroup) {
          const delay = parseFloat(value);
          if (isNaN(delay)) {
            errors.push({ line: lineNum, message: `Invalid crawl-delay value: "${value}"` });
          } else {
            currentGroup.crawlDelay = delay;
          }
        }
        break;

      case 'host':
        // Non-standard but used by Yandex. Just note it.
        break;

      default:
        errors.push({ line: lineNum, message: `Unknown directive: "${directive}"` });
    }
  }

  return { groups, sitemaps, errors };
}
```

### URL Testing Logic
```typescript
// Implements Google's robots.txt matching spec (RFC 9309)
function isUrlAllowed(
  groups: RobotsGroup[],
  url: string,
  userAgent: string
): { allowed: boolean; matchedRule: string | null; line: number | null } {
  // 1. Find the most specific group for this user agent
  const specificGroup = groups.find(g =>
    g.userAgents.some(ua => ua.toLowerCase() === userAgent.toLowerCase())
  );
  const wildcardGroup = groups.find(g =>
    g.userAgents.some(ua => ua === '*')
  );
  const group = specificGroup || wildcardGroup;

  if (!group || group.rules.length === 0) {
    return { allowed: true, matchedRule: null, line: null };
  }

  // 2. Find the most specific matching rule
  // Rules: longer path = more specific. Allow takes precedence over Disallow at same length.
  let bestMatch: RobotsRule | null = null;
  let bestMatchLength = -1;

  for (const rule of group.rules) {
    if (urlMatchesPattern(url, rule.path)) {
      const matchLength = rule.path.length;
      if (matchLength > bestMatchLength ||
          (matchLength === bestMatchLength && rule.type === 'allow')) {
        bestMatch = rule;
        bestMatchLength = matchLength;
      }
    }
  }

  if (!bestMatch) {
    return { allowed: true, matchedRule: null, line: null };
  }

  return {
    allowed: bestMatch.type === 'allow',
    matchedRule: `${bestMatch.type === 'allow' ? 'Allow' : 'Disallow'}: ${bestMatch.path}`,
    line: bestMatch.line
  };
}

function urlMatchesPattern(url: string, pattern: string): boolean {
  // Convert robots.txt pattern to regex
  // * matches any sequence, $ matches end of URL
  let regex = pattern
    .replace(/[.+?^{}()|[\]\\]/g, '\\$&')  // Escape regex special chars (except * and $)
    .replace(/\*/g, '.*');

  if (regex.endsWith('\\$')) {
    regex = regex.slice(0, -2) + '$';
  } else {
    regex = regex; // Implicit prefix match
  }

  return new RegExp('^' + regex).test(url);
}
```

---

## Validation Rules

### Syntax Checks
| Code | Severity | Check |
|------|----------|-------|
| `ROBOTS_NOT_FOUND` | critical | No robots.txt found (404) |
| `ROBOTS_SERVER_ERROR` | critical | robots.txt returned 5xx |
| `ROBOTS_TOO_LARGE` | warning | File exceeds 500KB (Google's limit) |
| `ROBOTS_WRONG_CONTENT_TYPE` | warning | Content-Type is not text/plain |
| `ROBOTS_SYNTAX_ERROR` | warning | Invalid directive syntax |
| `ROBOTS_UNKNOWN_DIRECTIVE` | info | Unrecognized directive (e.g., Host) |
| `ROBOTS_EMPTY` | warning | File exists but is empty |

### SEO Checks
| Code | Severity | Check |
|------|----------|-------|
| `NO_SITEMAP_DIRECTIVE` | warning | No Sitemap line in robots.txt |
| `BLOCKS_IMPORTANT_PATHS` | warning | Blocks /css, /js, /images (prevents rendering) |
| `BLOCKS_ENTIRE_SITE` | critical | Disallow: / for Googlebot or * |
| `CRAWL_DELAY_SET` | info | Crawl-delay may slow indexing (Google ignores it but others don't) |
| `WILDCARD_OVERBLOCKING` | warning | Broad wildcard patterns may accidentally block important content |

### AEO-Specific Checks
| Code | Severity | Check |
|------|----------|-------|
| `AI_BOT_FULLY_BLOCKED` | critical | An AI bot is blocked from entire site |
| `AI_BOT_PARTIALLY_BLOCKED` | warning | AI bot blocked from some sections |
| `AI_BOTS_ALL_ALLOWED` | passed | All known AI bots can access the site |
| `AI_BOT_NO_RULES` | info | No explicit AI bot rules (inherits from *) |
| `MISSING_AI_BOT_RULES` | info | Consider adding explicit rules for AI crawlers |

**Complete AI Bot User-Agent List (keep this updated):**
```typescript
const AI_CRAWLERS = [
  { name: 'GPTBot', org: 'OpenAI', purpose: 'ChatGPT training & retrieval' },
  { name: 'ChatGPT-User', org: 'OpenAI', purpose: 'ChatGPT browsing plugin' },
  { name: 'ClaudeBot', org: 'Anthropic', purpose: 'Claude training data' },
  { name: 'Anthropic-AI', org: 'Anthropic', purpose: 'Anthropic general crawling' },
  { name: 'CCBot', org: 'Common Crawl', purpose: 'Open dataset (used by many AI models)' },
  { name: 'Cohere-AI', org: 'Cohere', purpose: 'AI model training' },
  { name: 'PerplexityBot', org: 'Perplexity', purpose: 'Perplexity AI search' },
  { name: 'Google-Extended', org: 'Google', purpose: 'Gemini/Bard training (not Search indexing)' },
  { name: 'Bytespider', org: 'ByteDance', purpose: 'TikTok/ByteDance AI training' },
  { name: 'OmgiliBot', org: 'Omgili/Webz.io', purpose: 'Web data collection' },
  { name: 'Applebot-Extended', org: 'Apple', purpose: 'Apple Intelligence training' },
  { name: 'FacebookBot', org: 'Meta', purpose: 'Meta AI training' },
  { name: 'Amazonbot', org: 'Amazon', purpose: 'Alexa/Amazon AI' },
  { name: 'YouBot', org: 'You.com', purpose: 'You.com AI search' },
  { name: 'Diffbot', org: 'Diffbot', purpose: 'Web data extraction for AI' },
  { name: 'ImagesiftBot', org: 'Hive', purpose: 'Image AI training' },
  { name: 'Timpibot', org: 'Timpi', purpose: 'Decentralized search AI' },
  { name: 'Meta-ExternalAgent', org: 'Meta', purpose: 'Meta AI agent browsing' },
];
```

---

## Frontend Display

### Three-Panel Layout

**Left Panel: Raw robots.txt** with syntax highlighting
- Line numbers
- Color coding: User-agent (blue), Allow (green), Disallow (red), Sitemap (purple), Comments (gray)
- Clickable lines (selecting a line highlights the corresponding parsed rule)
- Error markers on invalid lines

**Center Panel: Parsed Rules**
- Grouped by User-agent
- Each rule shows: directive, path, and which URLs it affects
- Toggle to filter by "AI Bots Only"

**Right Panel: AI Bot Summary**
- Grid of AI bot icons/logos
- Green check = allowed, Red X = blocked, Yellow = partially blocked
- Click a bot to see its specific rules

### URL Tester Section (Below)
```
+------------------------------------------------------+
| Test a URL: [/products/widget_______] [Test]         |
|                                                       |
| Results:                                              |
| Googlebot      [ALLOWED]  (no matching rule)         |
| GPTBot         [BLOCKED]  Disallow: / (line 8)       |
| ClaudeBot      [ALLOWED]  (no matching rule)         |
| PerplexityBot  [ALLOWED]  (no matching rule)         |
+------------------------------------------------------+
```

---

## Scoring

```
Score starts at 100, deductions:
- robots.txt not found: -50 (set to 50 max)
- Syntax errors: -5 each (max -20)
- Blocks CSS/JS/images: -10
- No sitemap directive: -5
- Each fully blocked AI bot: -10 (max -40)
- Each partially blocked AI bot: -5
- Blocks Googlebot entirely: -30
- File too large: -5
```
