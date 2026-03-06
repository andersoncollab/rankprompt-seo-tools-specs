# Tool 12: Heading Structure Analyzer

## What This Replicates
- SEO Review Tools heading checker
- Screaming Frog H1/H2 analysis
- Moz heading hierarchy checker

## What It Does
Extracts all heading tags (H1-H6) from a page and validates the hierarchy. Checks for missing H1, multiple H1s, skipped levels, keyword usage in headings, and whether headings match common search queries. AEO angle: AI engines use heading structure to understand page organization and identify key topics.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-heading-analyzer
```

### Request
```json
{
  "url": "https://example.com/blog/crm-guide",
  "targetKeywords": ["CRM software", "best CRM", "CRM comparison"]
}
```

### Response
```json
{
  "score": 75,
  "headings": [
    { "level": 1, "text": "The Ultimate Guide to CRM Software in 2024", "wordCount": 8, "hasKeyword": true },
    { "level": 2, "text": "What is CRM Software?", "wordCount": 4, "hasKeyword": true, "matchesQuery": true },
    { "level": 3, "text": "Key Features", "wordCount": 2, "hasKeyword": false },
    { "level": 3, "text": "Benefits for Small Business", "wordCount": 4, "hasKeyword": false },
    { "level": 2, "text": "Top 10 Best CRM Platforms", "wordCount": 5, "hasKeyword": true, "matchesQuery": true },
    { "level": 4, "text": "Pricing Details", "wordCount": 2, "hasKeyword": false, "skippedLevel": true }
  ],
  "structure": {
    "totalHeadings": 22,
    "h1Count": 1,
    "h2Count": 8,
    "h3Count": 10,
    "h4Count": 3,
    "h5Count": 0,
    "h6Count": 0,
    "maxDepth": 4,
    "skippedLevels": 1,
    "keywordInH1": true,
    "keywordInH2": 3,
    "queryMatchingHeaders": 6
  },
  "hierarchyTree": "H1: The Ultimate Guide to CRM Software in 2024\n  H2: What is CRM Software?\n    H3: Key Features\n    H3: Benefits for Small Business\n  H2: Top 10 Best CRM Platforms\n      H4: Pricing Details  [!SKIPPED H3]\n  ...",
  "issues": [
    {
      "code": "HEADING_LEVEL_SKIPPED",
      "severity": "warning",
      "title": "Heading level skipped: H2 -> H4",
      "message": "\"Pricing Details\" jumps from H2 to H4, skipping H3. This confuses both screen readers and AI parsing."
    },
    {
      "code": "AEO_HEADING_NOT_QUESTION",
      "severity": "info",
      "title": "6 of 22 headings match search queries",
      "message": "Consider rephrasing more headings as questions AI users ask (e.g., 'Key Features' -> 'What Features Should a CRM Have?')"
    }
  ]
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `H1_MISSING` | critical | No H1 tag found |
| `H1_MULTIPLE` | warning | More than one H1 tag |
| `H1_EMPTY` | critical | H1 tag exists but is empty |
| `H1_TOO_LONG` | warning | H1 exceeds 70 characters |
| `HEADING_LEVEL_SKIPPED` | warning | Heading levels jump (e.g., H2 to H4) |
| `HEADING_EMPTY` | warning | Empty heading tag |
| `HEADING_TOO_LONG` | info | Heading exceeds 70 characters |
| `HEADING_DUPLICATE` | info | Same heading text appears multiple times |
| `NO_H2_HEADINGS` | warning | Page has H1 but no H2 headings |
| `TOO_MANY_H1_KEYWORDS` | info | H1 appears stuffed with keywords |
| `AEO_HEADING_NOT_QUESTION` | info | Heading could be rephrased as a question for AI queries |
| `AEO_NO_KEYWORD_IN_H1` | warning | Target keyword not in H1 |
| `AEO_FEW_QUERY_MATCHES` | warning | Fewer than 20% of headings match common search queries |

---

## Frontend: Tree Visualization

Display headings as an indented tree with color-coded indicators:
```
v H1: The Ultimate Guide to CRM Software in 2024    [keyword] [OK]
  v H2: What is CRM Software?                       [keyword] [query]
    - H3: Key Features
    - H3: Benefits for Small Business
  v H2: Top 10 Best CRM Platforms                   [keyword] [query]
      ! H4: Pricing Details                          [SKIPPED LEVEL]
  v H2: How to Choose the Right CRM
    - H3: For Small Business
    - H3: For Enterprise
```

---

## Scoring

```
Score starts at 100:
- Missing H1: -25
- Multiple H1s: -10
- Each skipped level: -5 (max -20)
- Empty headings: -5 each
- No H2s: -15
- No keyword in H1: -10
- Fewer than 20% query-matching headings: -10
```
