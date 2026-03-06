# Tool 14: Internal Link Analyzer

## What This Replicates
- Screaming Frog internal link report
- Ahrefs internal link analysis
- Sitebulb link equity flow

## What It Does
Crawls internal links on a page (or small set of pages) and analyzes the linking structure: anchor text distribution, link equity flow, orphan page detection, and broken internal links. AEO angle: AI engines follow internal links to understand content relationships and topic authority.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-link-analyzer
```

### Request
```json
{
  "url": "https://example.com/blog/crm-guide",
  "options": {
    "depth": 1,
    "maxPages": 20,
    "includeExternal": false
  }
}
```

### Response
```json
{
  "score": 70,
  "page": {
    "url": "https://example.com/blog/crm-guide",
    "internalLinks": 18,
    "externalLinks": 7,
    "brokenLinks": 1,
    "nofollowLinks": 2
  },
  "internalLinks": [
    {
      "href": "/products/salesforce-integration",
      "anchorText": "Salesforce integration",
      "context": "...our powerful Salesforce integration that syncs...",
      "position": "body",
      "nofollow": false,
      "status": 200,
      "issues": []
    },
    {
      "href": "/blog/old-crm-post",
      "anchorText": "click here",
      "context": "For more details, click here...",
      "position": "body",
      "nofollow": false,
      "status": 404,
      "issues": [
        { "code": "LINK_BROKEN", "severity": "critical", "title": "Internal link returns 404" },
        { "code": "LINK_GENERIC_ANCHOR", "severity": "warning", "title": "Anchor text 'click here' is not descriptive" }
      ]
    }
  ],
  "anchorTextDistribution": {
    "descriptive": 14,
    "generic": 3,
    "naked_url": 1,
    "mostUsedAnchors": [
      { "text": "CRM software", "count": 3 },
      { "text": "learn more", "count": 2 }
    ]
  },
  "linkPositionDistribution": {
    "navigation": 5,
    "body": 10,
    "sidebar": 2,
    "footer": 1
  },
  "issues": [ ... ]
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `LINK_BROKEN` | critical | Internal link returns 404 |
| `LINK_REDIRECT` | warning | Internal link redirects |
| `LINK_GENERIC_ANCHOR` | warning | Anchor text is "click here", "learn more", "read more" |
| `LINK_NAKED_URL` | info | Anchor text is the raw URL |
| `LINK_TOO_MANY` | info | Page has 100+ internal links |
| `LINK_TOO_FEW` | warning | Page has fewer than 3 internal links |
| `LINK_ALL_NOFOLLOW` | warning | All internal links are nofollow |
| `LINK_DUPLICATE_ANCHOR` | info | Same anchor text used for different URLs |
| `AEO_LINK_GOOD_CONTEXT` | passed | Links with descriptive anchors help AI understand topic relationships |
| `AEO_LINK_POOR_CONTEXT` | warning | Generic anchors don't help AI engines follow topic threads |

---

## Scoring

```
Score starts at 100:
- Each broken link: -10 (max -30)
- Each redirecting link: -3 (max -15)
- Each generic anchor: -3 (max -15)
- Too few internal links: -15
- Too many internal links: -5
- All nofollow: -10
```
