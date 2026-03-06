# Tool 04: Meta Tag Analyzer

## What This Replicates
- Metatags.io
- SEOptimer meta tag checker
- Moz On-Page Grader (meta portion)

## What It Does
Extracts and validates all meta tags from a page's `<head>`. Checks title, description, Open Graph, Twitter Cards, viewport, canonical, robots meta, and dozens of other tags. Provides character count analysis, SERP preview, and social share preview. AEO angle: AI engines rely heavily on clean meta descriptions and structured page signals.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-meta-analyzer
```

### Request
```json
{
  "url": "https://example.com/products/widget"
}
```

### Response
```json
{
  "score": 72,
  "url": "https://example.com/products/widget",
  "title": {
    "value": "Super Widget - Best Widgets for 2024 | Example",
    "length": 49,
    "pixelWidth": 445,
    "issues": [
      { "code": "TITLE_GOOD_LENGTH", "severity": "passed", "title": "Title length is within recommended range (50-60 chars)" }
    ]
  },
  "description": {
    "value": "Shop the Super Widget, rated #1 by Consumer Reports. Free shipping on orders over $50.",
    "length": 86,
    "issues": [
      { "code": "DESC_TOO_SHORT", "severity": "warning", "title": "Meta description is shorter than recommended (120-160 chars)", "message": "Your description is 86 characters. Aim for 120-160 for maximum SERP and AI snippet coverage." }
    ]
  },
  "canonical": {
    "value": "https://example.com/products/widget",
    "selfReferencing": true,
    "issues": []
  },
  "robots": {
    "value": "index, follow",
    "index": true,
    "follow": true,
    "nosnippet": false,
    "maxSnippet": null,
    "issues": []
  },
  "viewport": {
    "value": "width=device-width, initial-scale=1",
    "issues": []
  },
  "openGraph": {
    "og:title": "Super Widget - Best Widgets for 2024",
    "og:description": "Shop the Super Widget, rated #1 by Consumer Reports.",
    "og:image": "https://example.com/images/widget.jpg",
    "og:url": "https://example.com/products/widget",
    "og:type": "product",
    "og:site_name": "Example Store",
    "issues": [
      { "code": "OG_IMAGE_GOOD", "severity": "passed", "title": "og:image is present" },
      { "code": "OG_MISSING_LOCALE", "severity": "info", "title": "Missing og:locale tag" }
    ]
  },
  "twitterCard": {
    "twitter:card": "summary_large_image",
    "twitter:title": "Super Widget",
    "twitter:description": "Shop the Super Widget.",
    "twitter:image": "https://example.com/images/widget-tw.jpg",
    "twitter:site": "@examplestore",
    "issues": []
  },
  "otherMeta": {
    "charset": "utf-8",
    "language": "en",
    "author": "Example Inc.",
    "generator": null,
    "theme-color": "#ffffff",
    "format-detection": "telephone=no"
  },
  "linkTags": {
    "canonical": "https://example.com/products/widget",
    "alternate": [],
    "prev": null,
    "next": null,
    "hreflang": []
  },
  "serpPreview": {
    "title": "Super Widget - Best Widgets for 2024 | Example",
    "url": "example.com > products > widget",
    "description": "Shop the Super Widget, rated #1 by Consumer Reports. Free shipping on orders over $50.",
    "truncated": false
  },
  "aeoAnalysis": {
    "descriptionQuality": "partial",
    "entityClarity": "The meta description mentions the product name and a third-party endorsement, which AI engines can extract.",
    "suggestions": [
      "Extend meta description to 120-160 characters for maximum AI snippet coverage",
      "Include key product attributes (price, rating) in the description for richer AI answers",
      "Consider adding max-snippet meta directive to control how much AI engines can excerpt"
    ]
  },
  "issues": [ ... ]
}
```

---

## Extraction Logic

```typescript
function extractMetaTags($: cheerio.CheerioAPI): MetaTagResult {
  // Title
  const title = $('title').first().text().trim();

  // Meta description
  const description = $('meta[name="description"]').attr('content')?.trim() || null;

  // Canonical
  const canonical = $('link[rel="canonical"]').attr('href')?.trim() || null;

  // Robots meta
  const robotsMeta = $('meta[name="robots"]').attr('content')?.trim() || null;
  const googlebotMeta = $('meta[name="googlebot"]').attr('content')?.trim() || null;

  // Viewport
  const viewport = $('meta[name="viewport"]').attr('content')?.trim() || null;

  // Charset
  const charset = $('meta[charset]').attr('charset') ||
    $('meta[http-equiv="Content-Type"]').attr('content')?.match(/charset=([^\s;]+)/)?.[1] || null;

  // Open Graph
  const og: Record<string, string> = {};
  $('meta[property^="og:"]').each((_, el) => {
    const property = $(el).attr('property') || '';
    const content = $(el).attr('content') || '';
    og[property] = content;
  });

  // Twitter Card
  const twitter: Record<string, string> = {};
  $('meta[name^="twitter:"]').each((_, el) => {
    const name = $(el).attr('name') || '';
    const content = $(el).attr('content') || '';
    twitter[name] = content;
  });

  // All other meta tags
  const other: Record<string, string> = {};
  $('meta[name]').each((_, el) => {
    const name = $(el).attr('name') || '';
    if (!name.startsWith('twitter:') && name !== 'description' &&
        name !== 'robots' && name !== 'viewport' && name !== 'googlebot') {
      other[name] = $(el).attr('content') || '';
    }
  });

  // Link tags
  const hreflang: Array<{ lang: string; href: string }> = [];
  $('link[rel="alternate"][hreflang]').each((_, el) => {
    hreflang.push({
      lang: $(el).attr('hreflang') || '',
      href: $(el).attr('href') || ''
    });
  });

  return { title, description, canonical, robotsMeta, googlebotMeta, viewport, charset, og, twitter, other, hreflang };
}
```

---

## Validation Rules

### Title Tag
| Code | Severity | Check |
|------|----------|-------|
| `TITLE_MISSING` | critical | No `<title>` tag found |
| `TITLE_EMPTY` | critical | Title tag is empty |
| `TITLE_TOO_SHORT` | warning | Fewer than 30 characters |
| `TITLE_TOO_LONG` | warning | More than 60 characters (truncated in SERP) |
| `TITLE_GOOD_LENGTH` | passed | Between 30-60 characters |
| `TITLE_DUPLICATE_SITE_NAME` | info | Title contains the site name twice |
| `TITLE_ALL_CAPS` | info | Title is all uppercase |
| `TITLE_PIXEL_OVERFLOW` | warning | Exceeds ~580px (Google's display limit) |

### Meta Description
| Code | Severity | Check |
|------|----------|-------|
| `DESC_MISSING` | warning | No meta description |
| `DESC_EMPTY` | warning | Meta description is empty |
| `DESC_TOO_SHORT` | warning | Fewer than 70 characters |
| `DESC_TOO_LONG` | warning | More than 160 characters (truncated) |
| `DESC_GOOD_LENGTH` | passed | Between 120-160 characters |
| `DESC_DUPLICATE_TITLE` | info | Description is identical to title |

### Open Graph
| Code | Severity | Check |
|------|----------|-------|
| `OG_MISSING` | warning | No Open Graph tags found |
| `OG_MISSING_TITLE` | warning | Missing og:title |
| `OG_MISSING_DESCRIPTION` | warning | Missing og:description |
| `OG_MISSING_IMAGE` | warning | Missing og:image |
| `OG_MISSING_URL` | info | Missing og:url |
| `OG_MISSING_TYPE` | info | Missing og:type |
| `OG_IMAGE_TOO_SMALL` | warning | og:image smaller than 1200x630 |
| `OG_COMPLETE` | passed | All required OG tags present |

### Twitter Card
| Code | Severity | Check |
|------|----------|-------|
| `TW_MISSING` | info | No Twitter Card tags |
| `TW_MISSING_CARD` | warning | Missing twitter:card type |
| `TW_MISSING_IMAGE` | info | Missing twitter:image |
| `TW_COMPLETE` | passed | All Twitter Card tags present |

### Technical Meta
| Code | Severity | Check |
|------|----------|-------|
| `VIEWPORT_MISSING` | critical | No viewport meta (not mobile-friendly) |
| `CHARSET_MISSING` | warning | No charset declaration |
| `CHARSET_NOT_UTF8` | warning | Charset is not UTF-8 |
| `CANONICAL_MISSING` | warning | No canonical tag |
| `CANONICAL_MISMATCH` | warning | Canonical URL doesn't match current URL |
| `ROBOTS_NOINDEX` | info | Page is set to noindex |
| `ROBOTS_NOSNIPPET` | warning | nosnippet prevents SERP previews AND AI excerpts |
| `MULTIPLE_TITLES` | warning | More than one title tag |
| `MULTIPLE_DESCRIPTIONS` | warning | More than one meta description |
| `MULTIPLE_CANONICALS` | critical | More than one canonical tag |

### AEO-Specific Meta Checks
| Code | Severity | Check |
|------|----------|-------|
| `AEO_DESC_EXTRACTABLE` | passed | Meta description is a clean, factual statement AI engines can cite |
| `AEO_DESC_TOO_VAGUE` | warning | Description uses vague marketing language instead of factual claims |
| `AEO_NOSNIPPET_BLOCKS_AI` | critical | nosnippet or max-snippet:0 prevents AI from excerpting content |
| `AEO_NO_LANGUAGE` | info | No html lang attribute or language meta (AI engines use this for relevance) |
| `AEO_DESC_MISSING_ENTITY` | warning | Description doesn't mention the primary entity/brand name |

---

## SERP Preview

Generate a Google SERP preview:
```typescript
function generateSerpPreview(url: string, title: string, description: string): SerpPreview {
  // Google truncates at approximately:
  // Title: 580px (roughly 55-60 chars)
  // Description: 920px (roughly 155-160 chars)
  // URL: displayed as breadcrumb path

  const parsedUrl = new URL(url);
  const breadcrumb = [
    parsedUrl.hostname,
    ...parsedUrl.pathname.split('/').filter(Boolean).map(s => s.replace(/-/g, ' '))
  ].join(' > ');

  return {
    title: title.length > 60 ? title.substring(0, 57) + '...' : title,
    url: breadcrumb,
    description: description && description.length > 160
      ? description.substring(0, 157) + '...'
      : description || 'No meta description provided. Google will auto-generate one from page content.',
    truncated: (title?.length || 0) > 60 || (description?.length || 0) > 160
  };
}
```

---

## Scoring

```
Score starts at 100:
- Missing title: -25
- Title too short/long: -5
- Missing description: -15
- Description too short/long: -5
- Missing canonical: -10
- Missing viewport: -15
- Missing charset: -5
- Missing og:title, og:description, og:image: -5 each
- nosnippet blocking AI: -15
- Multiple titles/canonicals: -10
- No language declaration: -3
```
