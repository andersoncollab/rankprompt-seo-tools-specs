# Tool 03: Sitemap Validator

## What This Replicates
- XML-Sitemaps.com validator
- Screaming Frog sitemap analysis
- Google Search Console sitemap report

## What It Does
Fetches and validates XML sitemaps (and sitemap indexes). Checks XML syntax, URL limits, status codes of listed URLs (sample), and reports on sitemap health. AEO angle: sitemaps help AI crawlers discover content efficiently.

---

## User Flow

1. User enters a domain or direct sitemap URL
2. System checks `/sitemap.xml`, `/sitemap_index.xml`, and robots.txt Sitemap directives
3. Fetches and parses sitemaps (follows sitemap index if found)
4. Validates XML structure, URLs, dates, and limits
5. Optionally spot-checks a sample of URLs for 200 status

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-sitemap-validator
```

### Request Body
```json
{
  "url": "https://example.com",
  "sitemapUrl": null,
  "options": {
    "spotCheckUrls": true,
    "spotCheckLimit": 20,
    "followSitemapIndex": true
  }
}
```

### Response
```json
{
  "score": 82,
  "discovery": {
    "method": "robots.txt",
    "sitemapUrls": [
      "https://example.com/sitemap.xml",
      "https://example.com/sitemap-posts.xml",
      "https://example.com/sitemap-pages.xml"
    ]
  },
  "sitemaps": [
    {
      "url": "https://example.com/sitemap.xml",
      "type": "sitemapindex",
      "status": 200,
      "urlCount": 0,
      "childSitemaps": 3,
      "issues": [],
      "lastmod": null,
      "gzipped": false
    },
    {
      "url": "https://example.com/sitemap-posts.xml",
      "type": "urlset",
      "status": 200,
      "urlCount": 847,
      "childSitemaps": 0,
      "issues": [],
      "lastmod": "2024-03-01",
      "gzipped": false,
      "urlSample": [
        { "loc": "https://example.com/blog/post-1", "lastmod": "2024-02-28", "changefreq": "monthly", "priority": "0.8", "httpStatus": 200 },
        { "loc": "https://example.com/blog/old-post", "lastmod": "2022-01-15", "changefreq": null, "priority": null, "httpStatus": 301 }
      ]
    }
  ],
  "summary": {
    "totalSitemaps": 3,
    "totalUrls": 1247,
    "urlsWith200": 18,
    "urlsWithRedirects": 2,
    "urlsWith404": 0,
    "urlsWithLastmod": 1100,
    "urlsWithChangefreq": 200,
    "urlsWithPriority": 200,
    "oldestLastmod": "2019-06-15",
    "newestLastmod": "2024-03-01"
  },
  "issues": [
    {
      "code": "SITEMAP_URL_REDIRECTS",
      "severity": "warning",
      "title": "2 URLs in sitemap return redirects",
      "message": "Sitemaps should contain only canonical, non-redirecting URLs. Update these entries to their final destinations.",
      "urls": ["https://example.com/blog/old-post"]
    },
    {
      "code": "SITEMAP_STALE_LASTMOD",
      "severity": "info",
      "title": "147 URLs have lastmod dates older than 1 year",
      "message": "Consider reviewing and refreshing stale content, or removing outdated pages from the sitemap."
    }
  ]
}
```

---

## Validation Rules

### XML/Structure Checks
| Code | Severity | Check |
|------|----------|-------|
| `SITEMAP_NOT_FOUND` | critical | No sitemap found at standard locations |
| `SITEMAP_XML_INVALID` | critical | XML parsing error |
| `SITEMAP_WRONG_NAMESPACE` | warning | Missing or incorrect XML namespace |
| `SITEMAP_EXCEEDS_50K_URLS` | critical | More than 50,000 URLs (Google's limit per sitemap) |
| `SITEMAP_EXCEEDS_50MB` | critical | Uncompressed file exceeds 50MB |
| `SITEMAP_NOT_UTF8` | warning | Encoding is not UTF-8 |
| `SITEMAP_HTTP_ERROR` | critical | Sitemap returned non-200 status |

### URL Checks
| Code | Severity | Check |
|------|----------|-------|
| `SITEMAP_URL_REDIRECTS` | warning | URL returns 301/302 |
| `SITEMAP_URL_404` | critical | URL returns 404 |
| `SITEMAP_URL_5XX` | critical | URL returns server error |
| `SITEMAP_URL_NOT_ABSOLUTE` | warning | URL is not absolute (relative path) |
| `SITEMAP_URL_DIFFERENT_DOMAIN` | warning | URL points to a different domain |
| `SITEMAP_URL_BLOCKED_ROBOTS` | warning | URL is disallowed in robots.txt |
| `SITEMAP_DUPLICATE_URL` | warning | Same URL appears multiple times |

### Metadata Checks
| Code | Severity | Check |
|------|----------|-------|
| `SITEMAP_NO_LASTMOD` | info | URLs missing lastmod dates |
| `SITEMAP_STALE_LASTMOD` | info | lastmod dates older than 1 year |
| `SITEMAP_FUTURE_LASTMOD` | warning | lastmod date is in the future |
| `SITEMAP_INVALID_LASTMOD` | warning | lastmod is not valid W3C datetime |
| `SITEMAP_INVALID_PRIORITY` | warning | Priority value not between 0.0 and 1.0 |
| `SITEMAP_INVALID_CHANGEFREQ` | warning | Changefreq not a recognized value |

### Discovery Checks
| Code | Severity | Check |
|------|----------|-------|
| `SITEMAP_NOT_IN_ROBOTS` | warning | Sitemap exists but not referenced in robots.txt |
| `SITEMAP_ROBOTS_MISMATCH` | warning | Robots.txt references a sitemap URL that doesn't exist |
| `SITEMAP_NO_INDEX` | info | No sitemap index found (not required but recommended for large sites) |

### AEO Checks
| Code | Severity | Check |
|------|----------|-------|
| `AEO_SITEMAP_HELPS_DISCOVERY` | passed | Sitemap is accessible and helps AI crawlers find content |
| `AEO_NO_SITEMAP_AI_IMPACT` | warning | Without a sitemap, AI crawlers may miss important pages |
| `AEO_BLOG_NOT_IN_SITEMAP` | warning | Blog/article pages not found in sitemap (these are most cited by AI) |

---

## Implementation Notes

### Sitemap Discovery Order
1. Check if user provided a direct sitemap URL
2. Try `{domain}/sitemap.xml`
3. Try `{domain}/sitemap_index.xml`
4. Parse robots.txt for Sitemap directives
5. Try common CMS patterns: `/wp-sitemap.xml`, `/sitemap.xml.gz`, `/sitemap/sitemap-index.xml`

### XML Parsing
Use a streaming XML parser for large sitemaps. In Deno/Edge Functions:
```typescript
// For Edge Functions, use a lightweight XML parser
// Option 1: fast-xml-parser (works in Deno)
import { XMLParser } from 'fast-xml-parser';

const parser = new XMLParser({
  ignoreAttributes: false,
  attributeNamePrefix: '@_',
});

const parsed = parser.parse(xmlContent);
```

### Spot-Checking URLs
Don't check all URLs (could be 50K+). Take a random sample:
```typescript
function sampleUrls(urls: string[], limit: number): string[] {
  if (urls.length <= limit) return urls;
  const shuffled = [...urls].sort(() => Math.random() - 0.5);
  return shuffled.slice(0, limit);
}
```

Make HEAD requests (not GET) to check status codes. Use `Promise.allSettled` with concurrency limit (5 parallel requests).

---

## Scoring

```
Score starts at 100:
- No sitemap found: -40
- XML parse error: -30
- Exceeds URL/size limit: -20
- Each URL returning 404 (in sample): -3
- Each URL redirecting (in sample): -2
- Missing from robots.txt: -5
- No lastmod on any URL: -5
- Duplicate URLs found: -5
- URLs blocked by robots.txt: -5
```
