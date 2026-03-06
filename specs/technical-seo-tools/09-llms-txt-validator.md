# Tool 09: llms.txt Validator & Generator

## What This Replicates
- No direct competitor tool exists yet (first-mover opportunity)
- Loosely inspired by robots.txt testers but for the llms.txt standard
- Reference spec: https://llmstxt.org/

## What It Does
Validates an existing llms.txt file against the emerging specification, or generates one from scratch by crawling the site. llms.txt is a markdown file at the root of a website that provides context to AI language models about the site's content, structure, and preferred usage. Think of it as "robots.txt for AI context, not access."

This is a major differentiator for RankPrompt. Very few sites have llms.txt yet, and no tool exists to validate or generate them.

---

## Background: What is llms.txt?

The llms.txt specification (llmstxt.org) defines a markdown file placed at `/llms.txt` that helps LLMs understand:
- What the site/company does
- Key pages and their purposes
- How the content should be interpreted
- What information is most important

### llms.txt Format
```markdown
# Site Name

> Brief description of the site/company

## Key Information
- Important fact 1
- Important fact 2

## Sections

### Products
- [Product A](/products/a): Description of product A
- [Product B](/products/b): Description of product B

### Documentation
- [Getting Started](/docs/start): How to get started
- [API Reference](/docs/api): Full API documentation

## Optional

- [Blog](/blog): Latest news and articles
- [About](/about): Company background
```

There's also `llms-full.txt` which can contain the full text content for AI training.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-llms-txt
```

### Request (Validate Mode)
```json
{
  "mode": "validate",
  "url": "https://example.com"
}
```

### Request (Generate Mode)
```json
{
  "mode": "generate",
  "url": "https://example.com",
  "options": {
    "crawlDepth": 2,
    "maxPages": 50,
    "includeDescriptions": true,
    "tone": "professional"
  }
}
```

### Response (Validate)
```json
{
  "score": 65,
  "found": true,
  "url": "https://example.com/llms.txt",
  "fullTxtFound": false,
  "fullTxtUrl": "https://example.com/llms-full.txt",
  "raw": "# Example Corp\n\n> We sell widgets...\n\n...",
  "parsed": {
    "title": "Example Corp",
    "description": "We sell widgets and gadgets for enterprise customers.",
    "sections": [
      {
        "name": "Products",
        "links": [
          { "text": "Super Widget", "url": "/products/widget", "description": "Our flagship product" },
          { "text": "Mega Gadget", "url": "/products/gadget", "description": null }
        ]
      }
    ],
    "optionalLinks": [
      { "text": "Blog", "url": "/blog", "description": "Latest news" }
    ]
  },
  "validation": {
    "structure": [
      { "code": "LLMS_HAS_TITLE", "severity": "passed", "title": "Has H1 title" },
      { "code": "LLMS_HAS_DESCRIPTION", "severity": "passed", "title": "Has blockquote description" },
      { "code": "LLMS_HAS_SECTIONS", "severity": "passed", "title": "Has organized sections" },
      { "code": "LLMS_MISSING_OPTIONAL", "severity": "info", "title": "No 'Optional' section" }
    ],
    "links": [
      { "code": "LLMS_LINK_200", "severity": "passed", "url": "/products/widget", "status": 200 },
      { "code": "LLMS_LINK_404", "severity": "warning", "url": "/products/gadget", "status": 404, "title": "Broken link in llms.txt" }
    ],
    "content": [
      { "code": "LLMS_DESCRIPTION_GOOD", "severity": "passed", "title": "Description is concise and informative" },
      { "code": "LLMS_MISSING_LINK_DESCRIPTIONS", "severity": "warning", "title": "2 links missing descriptions", "message": "Adding descriptions to links helps AI understand what each page offers." }
    ],
    "completeness": [
      { "code": "LLMS_MISSING_KEY_PAGES", "severity": "warning", "title": "Key pages not included", "message": "Your homepage, about page, and pricing page were detected on your site but not listed in llms.txt." },
      { "code": "LLMS_NO_FULL_TXT", "severity": "info", "title": "No llms-full.txt found", "message": "Consider creating llms-full.txt with expanded content for deeper AI understanding." }
    ]
  },
  "issues": [ ... ]
}
```

### Response (Generate)
```json
{
  "generated": "# Example Corp\n\n> Example Corp is an enterprise widget and gadget manufacturer based in Dallas, TX. We serve 500+ B2B customers across manufacturing, logistics, and healthcare.\n\n## Products\n\n- [Super Widget](/products/widget): Our flagship IoT-enabled widget for real-time monitoring. Starting at $299/unit.\n- [Mega Gadget](/products/gadget): Enterprise-grade gadget with API integration. Custom pricing.\n- [Widget Pro](/products/widget-pro): Advanced widget with AI analytics. $599/unit.\n\n## Resources\n\n- [Documentation](/docs): Technical documentation and API reference\n- [Case Studies](/case-studies): Customer success stories and ROI data\n- [Blog](/blog): Industry insights, product updates, and best practices\n\n## Company\n\n- [About Us](/about): Company history, mission, and leadership team\n- [Contact](/contact): Sales inquiries, support, and partnership opportunities\n- [Careers](/careers): Open positions and company culture\n\n## Optional\n\n- [Pricing](/pricing): Product pricing and plan comparison\n- [FAQ](/faq): Frequently asked questions about products and services\n- [Terms of Service](/terms): Legal terms and privacy policy\n",
  "crawledPages": 47,
  "sections": ["Products", "Resources", "Company", "Optional"],
  "keyPagesIncluded": 12,
  "instructions": "Save this as /llms.txt at your site root. Review and customize the descriptions to accurately reflect your content. Consider also creating /llms-full.txt with expanded content."
}
```

---

## Validation Rules

### Structure
| Code | Severity | Check |
|------|----------|-------|
| `LLMS_NOT_FOUND` | critical | No llms.txt at domain root |
| `LLMS_EMPTY` | critical | File exists but is empty |
| `LLMS_NOT_MARKDOWN` | warning | File doesn't appear to be valid markdown |
| `LLMS_NO_TITLE` | critical | Missing H1 title |
| `LLMS_NO_DESCRIPTION` | warning | Missing blockquote description |
| `LLMS_NO_SECTIONS` | warning | No organized sections (H2 headings) |
| `LLMS_TOO_LONG` | info | File exceeds 10KB (AI models may truncate) |
| `LLMS_TOO_SHORT` | warning | File is less than 200 characters (too sparse) |

### Links
| Code | Severity | Check |
|------|----------|-------|
| `LLMS_LINK_200` | passed | Link returns 200 |
| `LLMS_LINK_404` | warning | Broken link |
| `LLMS_LINK_REDIRECT` | info | Link redirects |
| `LLMS_LINK_EXTERNAL` | info | Link points outside the domain |
| `LLMS_MISSING_LINK_DESCRIPTIONS` | warning | Links without descriptions |
| `LLMS_RELATIVE_LINKS` | info | Using relative links (absolute are clearer for AI) |

### Completeness
| Code | Severity | Check |
|------|----------|-------|
| `LLMS_MISSING_KEY_PAGES` | warning | Important pages (homepage, about, products, contact) not listed |
| `LLMS_MISSING_SITEMAP_PAGES` | info | Pages in sitemap not represented in llms.txt |
| `LLMS_NO_FULL_TXT` | info | No llms-full.txt companion file |
| `LLMS_STALE` | warning | Links point to moved/changed content |

### Content Quality
| Code | Severity | Check |
|------|----------|-------|
| `LLMS_DESCRIPTION_GOOD` | passed | Description is informative and specific |
| `LLMS_DESCRIPTION_GENERIC` | warning | Description is too generic ("We are a company...") |
| `LLMS_MARKETING_HEAVY` | info | Heavy marketing language (AI prefers factual) |
| `LLMS_DUPLICATE_LINKS` | warning | Same URL listed multiple times |

---

## Generation Logic

When generating llms.txt from scratch:

1. **Crawl the site** (up to 50 pages, configurable)
   - Start from homepage
   - Follow internal links (breadth-first)
   - Extract: title, meta description, H1, page type (product, blog, about, etc.)

2. **Classify pages** into sections
   - Products/Services
   - Documentation/Resources
   - Blog/Content
   - Company/About
   - Support/Contact
   - Legal/Policies
   - Use URL patterns + content signals for classification

3. **Generate descriptions** for each page
   - Use meta description if available and good quality
   - Fallback to first paragraph or H1 + context
   - Keep descriptions to 1-2 sentences

4. **Assemble the markdown** following the llms.txt spec format

5. **Optional: Generate llms-full.txt** with expanded content for each page

```typescript
async function generateLlmsTxt(domain: string, options: GenerateOptions): Promise<string> {
  // 1. Crawl
  const pages = await crawlSite(domain, options.crawlDepth, options.maxPages);

  // 2. Classify
  const sections = classifyPages(pages);

  // 3. Build markdown
  let md = '';

  // Title (from homepage or brand name)
  const homepage = pages.find(p => p.url === '/');
  const siteName = homepage?.ogSiteName || homepage?.title?.split(' | ')[0] || domain;
  md += `# ${siteName}\n\n`;

  // Description
  const desc = homepage?.metaDescription || `${siteName} website`;
  md += `> ${desc}\n\n`;

  // Sections
  for (const [sectionName, sectionPages] of Object.entries(sections)) {
    if (sectionPages.length === 0) continue;
    md += `## ${sectionName}\n\n`;
    for (const page of sectionPages) {
      const description = page.metaDescription
        ? `: ${page.metaDescription.substring(0, 120)}`
        : '';
      md += `- [${page.title || page.url}](${page.url})${description}\n`;
    }
    md += '\n';
  }

  return md;
}
```

---

## Frontend

### Validate Mode
- Left panel: Raw llms.txt content with syntax highlighting
- Right panel: Validation results grouped by category
- Link status indicators (green/red dots next to each link)
- "Completeness meter" showing what % of key pages are covered

### Generate Mode
- Step 1: Enter domain, configure options
- Step 2: Watch crawl progress (live updates via SSE)
- Step 3: Review generated llms.txt in editable textarea
- Step 4: Download or copy to clipboard
- "Deploy" button: gives instructions for where to place the file

---

## Scoring

```
Validate mode:
- Not found: score = 0
- Found, starts at 100:
  - Missing title: -20
  - Missing description: -10
  - No sections: -15
  - Broken links: -5 each (max -20)
  - Missing key pages: -3 each (max -15)
  - Too short/sparse: -10
  - Generic descriptions: -5
  - Missing link descriptions: -5

Generate mode: no score (just generates the file)
```
