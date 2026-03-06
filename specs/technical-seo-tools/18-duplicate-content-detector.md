# Tool 18: Duplicate Content Detector

## What This Replicates
- Copyscape
- Siteliner (within-site duplicate detection)
- Screaming Frog duplicate content report

## What It Does
Compares a page's content against other pages on the same site (and optionally against external sources) to detect duplicate or near-duplicate content. Uses text similarity algorithms (Jaccard, MinHash, cosine similarity on n-grams) rather than exact matching. AEO angle: AI engines penalize sites with duplicate content and may refuse to cite pages they suspect are duplicated.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-duplicate-detector
```

### Request
```json
{
  "url": "https://example.com/products/widget-pro",
  "options": {
    "compareTo": "internal",
    "samplePages": 20,
    "threshold": 0.6
  }
}
```

`compareTo`: `"internal"` (same domain), `"specific"` (compare against provided URLs), or `"both"`.

### Response
```json
{
  "score": 72,
  "page": {
    "url": "https://example.com/products/widget-pro",
    "wordCount": 1200,
    "uniqueContentRatio": 0.78,
    "boilerplateRatio": 0.22
  },
  "internalMatches": [
    {
      "url": "https://example.com/products/widget-basic",
      "similarity": 0.73,
      "matchType": "near_duplicate",
      "sharedContent": "Product specifications section is 73% identical. Both pages share the same 'Features' and 'How It Works' sections with only the product name changed.",
      "uniqueToThisPage": ["Premium support tier", "Advanced analytics dashboard"],
      "uniqueToOtherPage": ["Basic support", "Standard reporting"]
    },
    {
      "url": "https://example.com/products/widget",
      "similarity": 0.45,
      "matchType": "partial_overlap",
      "sharedContent": "Introduction paragraph and footer CTA are identical."
    }
  ],
  "boilerplateContent": {
    "navigation": true,
    "footer": true,
    "sidebar": true,
    "sharedParagraphs": 3,
    "note": "22% of page content is shared boilerplate (header, footer, sidebar). This is normal and excluded from duplicate scoring."
  },
  "issues": [
    {
      "code": "DUPLICATE_NEAR_MATCH",
      "severity": "warning",
      "title": "73% content overlap with /products/widget-basic",
      "message": "These pages share most of their content. Consider consolidating into one page with a comparison section, or differentiating the content significantly.",
      "aeoImpact": "AI engines may randomly choose which version to cite, or avoid citing either. Consolidate to ensure consistent AI visibility."
    }
  ]
}
```

---

## Implementation: Similarity Algorithm

```typescript
// MinHash-based near-duplicate detection
// Efficient for comparing many pages without O(n^2) full-text comparison

function generateShingles(text: string, shingleSize: number = 3): Set<string> {
  const words = text.toLowerCase()
    .replace(/[^\w\s]/g, '')
    .split(/\s+/)
    .filter(w => w.length > 2);

  const shingles = new Set<string>();
  for (let i = 0; i <= words.length - shingleSize; i++) {
    shingles.add(words.slice(i, i + shingleSize).join(' '));
  }
  return shingles;
}

function jaccardSimilarity(a: Set<string>, b: Set<string>): number {
  const intersection = new Set([...a].filter(x => b.has(x)));
  const union = new Set([...a, ...b]);
  return union.size === 0 ? 0 : intersection.size / union.size;
}

// For the page, strip boilerplate first:
function extractMainContent($: cheerio.CheerioAPI): string {
  // Remove nav, header, footer, sidebar, scripts, styles
  const clone = $.root().clone();
  clone.find('nav, header, footer, aside, script, style, [role="navigation"], [role="banner"], [role="contentinfo"]').remove();

  // Try to find main content area
  const main = clone.find('main, [role="main"], article, .content, #content, .post-content, .entry-content');
  if (main.length) {
    return main.text().trim();
  }

  // Fallback to body
  return clone.find('body').text().trim();
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `DUPLICATE_EXACT_MATCH` | critical | 95%+ identical content on another URL |
| `DUPLICATE_NEAR_MATCH` | warning | 60-94% content overlap |
| `DUPLICATE_PARTIAL_OVERLAP` | info | 30-59% content overlap |
| `DUPLICATE_THIN_CONTENT` | warning | Page has fewer than 300 words of unique content |
| `DUPLICATE_NO_CANONICAL` | critical | Duplicate content found but no canonical to resolve it |
| `DUPLICATE_CANONICAL_WRONG` | critical | Duplicate exists but canonical points to this page (not the original) |
| `AEO_DUPLICATE_CITATION_RISK` | warning | AI engines may avoid citing pages with high internal duplication |
| `AEO_UNIQUE_CONTENT_GOOD` | passed | 80%+ unique content ratio is strong for AI citation |

---

## Scoring

```
Score starts at 100:
- Exact duplicate found: -40
- Near duplicate (>70% similar): -20 each (max -40)
- Partial overlap noted: -5 each (max -15)
- Thin unique content (<300 words): -15
- Duplicate without canonical resolution: -20
- Score cannot go below 0
```

---

## Implementation Notes

- This tool is computationally expensive. Limit to 20 pages per scan.
- Strip boilerplate (nav, footer, sidebar) before comparison to avoid false positives.
- For "internal" mode, use the sitemap to discover comparison pages, or crawl from the target URL.
- Consider running this in Cloud Run rather than Edge Functions due to memory/CPU needs.
- Cache page content fingerprints in the DB for faster repeat comparisons.
- 5-credit tool (premium) due to multi-page crawling.
