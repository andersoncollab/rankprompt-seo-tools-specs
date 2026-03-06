# Tool 06: Redirect Chain Checker

## What This Replicates
- Redirect Checker (httpstatus.io)
- Screaming Frog redirect analysis
- Varvy redirect checker

## What It Does
Follows a URL through its full redirect chain, reporting every hop with status codes, headers, and timing. Identifies issues like redirect loops, chains longer than 3 hops, mixed HTTP/HTTPS, and SEO-damaging redirect patterns.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-redirect-checker
```

### Request
```json
{
  "urls": [
    "http://example.com",
    "https://example.com/old-page",
    "https://example.com/products/widget?ref=homepage"
  ],
  "maxRedirects": 10,
  "userAgent": "Googlebot"
}
```

### Response
```json
{
  "score": 65,
  "results": [
    {
      "inputUrl": "http://example.com",
      "chain": [
        {
          "url": "http://example.com",
          "statusCode": 301,
          "statusText": "Moved Permanently",
          "redirectTo": "https://example.com/",
          "headers": { "location": "https://example.com/" },
          "timing": 45
        },
        {
          "url": "https://example.com/",
          "statusCode": 301,
          "statusText": "Moved Permanently",
          "redirectTo": "https://www.example.com/",
          "headers": { "location": "https://www.example.com/" },
          "timing": 38
        },
        {
          "url": "https://www.example.com/",
          "statusCode": 200,
          "statusText": "OK",
          "redirectTo": null,
          "headers": { ... },
          "timing": 120
        }
      ],
      "hops": 2,
      "totalTime": 203,
      "finalUrl": "https://www.example.com/",
      "finalStatus": 200,
      "issues": [
        {
          "code": "REDIRECT_CHAIN_TOO_LONG",
          "severity": "warning",
          "title": "Redirect chain has 2 hops",
          "message": "http://example.com redirects to https://example.com/ which redirects to https://www.example.com/. Consider consolidating to a single redirect."
        }
      ]
    }
  ],
  "summary": {
    "totalUrls": 3,
    "urlsWith200": 1,
    "urlsWithRedirects": 2,
    "urlsWith404": 0,
    "averageChainLength": 1.3,
    "longestChain": 2
  }
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `REDIRECT_CHAIN_TOO_LONG` | warning | More than 1 redirect hop (ideal is direct) |
| `REDIRECT_CHAIN_CRITICAL` | critical | More than 3 hops (Google may stop following) |
| `REDIRECT_LOOP` | critical | URL redirects back to itself or a previous URL in chain |
| `REDIRECT_302_SHOULD_BE_301` | warning | Using 302 (temporary) for what appears permanent |
| `REDIRECT_307_FOUND` | info | 307 Temporary Redirect found |
| `REDIRECT_META_REFRESH` | warning | Redirect via meta refresh tag (not recommended) |
| `REDIRECT_JS_REDIRECT` | warning | Redirect via JavaScript (invisible to crawlers) |
| `REDIRECT_MIXED_PROTOCOL` | warning | Chain mixes HTTP and HTTPS hops |
| `REDIRECT_CROSS_DOMAIN` | info | Redirect goes to a different domain |
| `REDIRECT_STRIPS_TRAILING_SLASH` | info | Redirect adds/removes trailing slash |
| `REDIRECT_DROPS_PARAMS` | warning | Query parameters lost during redirect |
| `NO_REDIRECT` | passed | URL resolves directly with 200 |
| `FINAL_404` | critical | Redirect chain ends in 404 |
| `FINAL_5XX` | critical | Redirect chain ends in server error |

### AEO-Specific
| Code | Severity | Check |
|------|----------|-------|
| `AEO_LONG_CHAIN_SLOW_CRAWL` | warning | Long redirect chains waste AI crawler's time budget |
| `AEO_REDIRECT_TO_BLOCKED` | critical | Chain ends at a URL blocked by robots.txt for AI bots |

---

## Implementation

```typescript
async function followRedirects(
  url: string,
  maxRedirects: number = 10,
  userAgent: string = 'RankPromptBot/1.0'
): Promise<RedirectChain> {
  const chain: RedirectHop[] = [];
  let currentUrl = url;
  const visited = new Set<string>();

  for (let i = 0; i <= maxRedirects; i++) {
    if (visited.has(currentUrl)) {
      chain.push({
        url: currentUrl,
        statusCode: 0,
        statusText: 'REDIRECT_LOOP',
        redirectTo: null,
        headers: {},
        timing: 0
      });
      break;
    }
    visited.add(currentUrl);

    const start = Date.now();
    const response = await fetch(currentUrl, {
      method: 'HEAD',
      redirect: 'manual',  // Don't auto-follow
      headers: { 'User-Agent': userAgent }
    });
    const timing = Date.now() - start;

    const headers: Record<string, string> = {};
    response.headers.forEach((value, key) => { headers[key] = value; });

    const location = response.headers.get('location');
    const redirectTo = location ? new URL(location, currentUrl).href : null;

    chain.push({
      url: currentUrl,
      statusCode: response.status,
      statusText: response.statusText,
      redirectTo,
      headers,
      timing
    });

    if (response.status >= 300 && response.status < 400 && redirectTo) {
      currentUrl = redirectTo;
    } else {
      break; // Final destination
    }
  }

  return {
    inputUrl: url,
    chain,
    hops: chain.length - 1,
    totalTime: chain.reduce((sum, hop) => sum + hop.timing, 0),
    finalUrl: chain[chain.length - 1].url,
    finalStatus: chain[chain.length - 1].statusCode
  };
}
```

---

## Frontend: Visual Chain Display

```
http://example.com
  |
  | 301 Moved Permanently (45ms)
  v
https://example.com/
  |
  | 301 Moved Permanently (38ms)
  v
https://www.example.com/
  |
  | 200 OK (120ms)
  v
  [FINAL DESTINATION]

Total: 2 hops, 203ms
[!] Chain too long - consolidate to single redirect
```

Use colored arrows: green for final 200, yellow for redirects, red for errors.

---

## Batch Mode

Allow users to paste up to 50 URLs (one per line) for bulk redirect checking. Results displayed as a table:

| Input URL | Hops | Final URL | Final Status | Issues |
|-----------|------|-----------|--------------|--------|
| http://example.com | 2 | https://www.example.com/ | 200 | Chain too long |
| /old-page | 1 | /new-page | 200 | OK |
| /deleted | 0 | /deleted | 404 | CRITICAL |

---

## Scoring

```
Per-URL scoring, then average:
- Direct 200 (no redirects): 100
- Single 301: 90
- 2 hops: 70
- 3+ hops: 50
- Redirect loop: 0
- Final 404: 20
- Final 5xx: 10
- 302 instead of 301: -10
- Mixed protocol: -5
```
