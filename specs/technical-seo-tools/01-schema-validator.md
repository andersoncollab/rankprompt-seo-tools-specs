# Tool 01: Schema Markup Validator

## What This Replicates
- Google's Rich Results Test (search.google.com/test/rich-results)
- Schema.org Validator (validator.schema.org)
- Merkle Schema Markup Validator

## What It Does
Extracts, parses, and validates all structured data (JSON-LD, Microdata, RDFa) from a page. Reports errors, warnings, and optimization suggestions. Goes beyond schema.org validation by checking **Google Rich Result eligibility** and **AI engine extractability**.

---

## User Flow

1. User enters a URL OR pastes raw HTML/JSON-LD
2. System fetches the page (or uses pasted content)
3. Extracts all structured data in all three formats
4. Validates against schema.org vocabulary
5. Checks Google Rich Result requirements
6. Checks AEO optimization signals
7. Displays results in a tree view with expandable details

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-schema-validator
```

### Request Body
```json
{
  "url": "https://example.com/product/widget",
  "rawInput": null,
  "inputType": "url",
  "options": {
    "checkRichResults": true,
    "checkAEO": true,
    "renderJS": false
  }
}
```

`inputType` can be `"url"`, `"html"`, or `"jsonld"`. If `"html"` or `"jsonld"`, the content goes in `rawInput`.

### Response
```json
{
  "score": 85,
  "summary": {
    "totalItems": 4,
    "formats": { "jsonLd": 3, "microdata": 1, "rdfa": 0 },
    "errors": 1,
    "warnings": 2,
    "passed": 12
  },
  "items": [
    {
      "format": "json-ld",
      "type": "Product",
      "raw": "{ ... original JSON-LD ... }",
      "parsed": { ... normalized object ... },
      "location": {
        "source": "script[type='application/ld+json']",
        "index": 0,
        "line": 45
      },
      "checks": [
        {
          "code": "SCHEMA_TYPE_VALID",
          "severity": "passed",
          "title": "Valid schema.org type",
          "message": "Product is a recognized schema.org type"
        },
        {
          "code": "RICH_RESULT_MISSING_FIELD",
          "severity": "warning",
          "title": "Missing 'aggregateRating' for Product rich result",
          "message": "Google recommends aggregateRating for Product rich results. Adding this field increases your chance of earning a rich snippet.",
          "field": "aggregateRating",
          "richResultType": "Product",
          "aeoImpact": "AI engines use rating data when recommending products. Without this, your product may be described without ratings context."
        }
      ],
      "richResultEligibility": {
        "eligible": true,
        "type": "Product",
        "missingRequired": [],
        "missingRecommended": ["aggregateRating", "review"],
        "status": "partial"
      }
    }
  ],
  "aeoAnalysis": {
    "entityCoverage": "good",
    "faqPresent": false,
    "howToPresent": false,
    "breadcrumbPresent": true,
    "speakablePresent": false,
    "suggestions": [
      "Add FAQPage schema to increase chances of AI engines citing your FAQ answers",
      "Add Speakable schema to help voice assistants identify key content",
      "Consider adding HowTo schema if your page contains instructional content"
    ]
  }
}
```

---

## Extraction Logic

### 1. JSON-LD Extraction
```typescript
import * as cheerio from 'cheerio';

interface ExtractedItem {
  format: 'json-ld' | 'microdata' | 'rdfa';
  type: string | string[];
  data: Record<string, any>;
  raw: string;
  location: { source: string; index: number; line?: number };
}

function extractJsonLd($: cheerio.CheerioAPI): ExtractedItem[] {
  const items: ExtractedItem[] = [];

  $('script[type="application/ld+json"]').each((index, el) => {
    const rawContent = $(el).html() || '';
    try {
      const parsed = JSON.parse(rawContent);

      // Handle @graph arrays
      const entities = parsed['@graph'] || [parsed];

      for (const entity of entities) {
        if (!entity['@type']) continue;
        items.push({
          format: 'json-ld',
          type: entity['@type'],
          data: entity,
          raw: JSON.stringify(entity, null, 2),
          location: { source: 'script[type="application/ld+json"]', index }
        });
      }
    } catch (e) {
      // Still report the broken JSON-LD as an error item
      items.push({
        format: 'json-ld',
        type: 'PARSE_ERROR',
        data: { error: (e as Error).message, raw: rawContent.substring(0, 500) },
        raw: rawContent,
        location: { source: 'script[type="application/ld+json"]', index }
      });
    }
  });

  return items;
}
```

### 2. Microdata Extraction
```typescript
function extractMicrodata($: cheerio.CheerioAPI): ExtractedItem[] {
  const items: ExtractedItem[] = [];

  // Find top-level itemscope elements (not nested inside another itemscope)
  $('[itemscope]').not('[itemscope] [itemscope]').each((index, el) => {
    const item = parseMicrodataItem($, $(el));
    items.push({
      format: 'microdata',
      type: item.type,
      data: item.properties,
      raw: $.html(el) || '',
      location: { source: `[itemscope]:eq(${index})`, index }
    });
  });

  return items;
}

function parseMicrodataItem($: cheerio.CheerioAPI, el: cheerio.Cheerio<any>): {
  type: string;
  properties: Record<string, any>;
} {
  const itemType = el.attr('itemtype') || 'Unknown';
  // Extract schema.org type from full URL
  const type = itemType.replace('https://schema.org/', '').replace('http://schema.org/', '');
  const properties: Record<string, any> = {};

  el.find('[itemprop]').each((_, propEl) => {
    const $prop = $(propEl);
    const propName = $prop.attr('itemprop') || '';

    // If this prop is inside a nested itemscope that isn't our element, skip
    const parentScope = $prop.closest('[itemscope]');
    if (parentScope[0] !== el[0] && !el.is(parentScope)) return;

    let value: any;
    if ($prop.is('[itemscope]')) {
      // Nested entity
      value = parseMicrodataItem($, $prop);
    } else if ($prop.is('meta')) {
      value = $prop.attr('content') || '';
    } else if ($prop.is('a, link')) {
      value = $prop.attr('href') || '';
    } else if ($prop.is('img')) {
      value = $prop.attr('src') || '';
    } else if ($prop.is('time')) {
      value = $prop.attr('datetime') || $prop.text();
    } else {
      value = $prop.text().trim();
    }

    // Handle multiple values for same property
    if (properties[propName]) {
      if (!Array.isArray(properties[propName])) {
        properties[propName] = [properties[propName]];
      }
      properties[propName].push(value);
    } else {
      properties[propName] = value;
    }
  });

  return { type, properties };
}
```

### 3. RDFa Extraction
```typescript
function extractRdfa($: cheerio.CheerioAPI): ExtractedItem[] {
  const items: ExtractedItem[] = [];

  $('[typeof]').each((index, el) => {
    const $el = $(el);
    const type = $el.attr('typeof') || 'Unknown';
    const vocab = $el.attr('vocab') || findParentVocab($, $el);
    const properties: Record<string, any> = {};

    $el.find('[property]').each((_, propEl) => {
      const $prop = $(propEl);
      const propName = $prop.attr('property') || '';
      const content = $prop.attr('content') || $prop.text().trim();
      properties[propName] = content;
    });

    if (vocab?.includes('schema.org')) {
      items.push({
        format: 'rdfa',
        type,
        data: properties,
        raw: $.html(el) || '',
        location: { source: `[typeof]:eq(${index})`, index }
      });
    }
  });

  return items;
}

function findParentVocab($: cheerio.CheerioAPI, el: cheerio.Cheerio<any>): string | undefined {
  let current = el.parent();
  while (current.length) {
    const vocab = current.attr('vocab');
    if (vocab) return vocab;
    current = current.parent();
  }
  return undefined;
}
```

---

## Validation Rules

### Level 1: Syntax Validation
| Code | Severity | Check |
|------|----------|-------|
| `JSONLD_PARSE_ERROR` | critical | JSON-LD block contains invalid JSON |
| `JSONLD_MISSING_CONTEXT` | warning | Missing `@context` declaration |
| `JSONLD_INVALID_CONTEXT` | warning | `@context` is not schema.org |
| `MICRODATA_MISSING_ITEMTYPE` | warning | `itemscope` without `itemtype` attribute |
| `MICRODATA_INVALID_URL` | warning | `itemtype` is not a valid schema.org URL |

### Level 2: Schema.org Vocabulary Validation
| Code | Severity | Check |
|------|----------|-------|
| `TYPE_NOT_FOUND` | critical | `@type` is not a recognized schema.org type |
| `PROPERTY_NOT_FOUND` | warning | Property does not exist on the declared type |
| `PROPERTY_WRONG_TYPE` | warning | Property value type doesn't match expected range |
| `DEPRECATED_TYPE` | info | Using a deprecated schema.org type |
| `DEPRECATED_PROPERTY` | info | Using a deprecated property |

### Level 3: Google Rich Results Validation
| Code | Severity | Check |
|------|----------|-------|
| `RICH_RESULT_MISSING_REQUIRED` | critical | Missing a required field for rich result eligibility |
| `RICH_RESULT_MISSING_RECOMMENDED` | warning | Missing a recommended field |
| `RICH_RESULT_ELIGIBLE` | passed | All required fields present for rich result |
| `RICH_RESULT_IMAGE_TOO_SMALL` | warning | Image doesn't meet minimum size requirements |

**Google Rich Result Types to Validate:**
- Product (name, image, offers with price+priceCurrency+availability)
- Article (headline, image, datePublished, author)
- LocalBusiness (name, address, telephone)
- FAQPage (mainEntity with Question+acceptedAnswer)
- HowTo (name, step with text)
- Recipe (name, image, author, datePublished, description, prepTime, cookTime)
- Event (name, startDate, location, eventAttendanceMode)
- BreadcrumbList (itemListElement with position+name+item)
- VideoObject (name, description, thumbnailUrl, uploadDate)
- Review (itemReviewed, reviewRating, author)
- Organization (name, url, logo)
- WebSite (with SearchAction for sitelinks search box)
- SoftwareApplication (name, offers, applicationCategory)
- Course (name, description, provider)
- JobPosting (title, description, datePosted, hiringOrganization, jobLocation)

### Level 4: AEO-Specific Checks
| Code | Severity | Check |
|------|----------|-------|
| `AEO_NO_FAQ_SCHEMA` | warning | Page has FAQ-like content but no FAQPage schema |
| `AEO_NO_HOWTO_SCHEMA` | info | Page has instructional content but no HowTo schema |
| `AEO_NO_SPEAKABLE` | info | No Speakable schema (helps voice/AI assistants) |
| `AEO_MISSING_ABOUT` | warning | Organization/WebPage missing `about` property (helps AI understand topic) |
| `AEO_MISSING_DESCRIPTION` | warning | Entity missing `description` (AI engines rely heavily on this) |
| `AEO_NO_BREADCRUMB` | warning | No BreadcrumbList (helps AI understand site hierarchy) |
| `AEO_THIN_ENTITY` | warning | Entity has fewer than 3 properties (too sparse for AI extraction) |

---

## Schema.org Vocabulary Cache

Download and cache the full schema.org vocabulary for validation:

```typescript
// Download once, cache in Supabase Storage or as a KV entry
const SCHEMA_ORG_URL = 'https://schema.org/version/latest/schemaorg-current-https.jsonld';

interface SchemaVocab {
  types: Map<string, SchemaType>;
  properties: Map<string, SchemaProperty>;
  lastUpdated: string;
}

interface SchemaType {
  id: string;
  label: string;
  comment: string;
  subClassOf: string[];
  properties: string[];  // Property IDs that belong to this type
  supersededBy?: string;
}

interface SchemaProperty {
  id: string;
  label: string;
  comment: string;
  domainIncludes: string[];  // Types this property belongs to
  rangeIncludes: string[];   // Expected value types
  supersededBy?: string;
}

async function loadSchemaVocab(): Promise<SchemaVocab> {
  // Check cache first (Supabase storage or KV)
  const cached = await getCachedVocab();
  if (cached && isLessThanOneWeekOld(cached.lastUpdated)) {
    return cached;
  }

  // Fetch fresh
  const response = await fetch(SCHEMA_ORG_URL);
  const data = await response.json();
  const vocab = parseSchemaOrgVocab(data);

  // Cache it
  await setCachedVocab(vocab);
  return vocab;
}

function parseSchemaOrgVocab(raw: any): SchemaVocab {
  const types = new Map<string, SchemaType>();
  const properties = new Map<string, SchemaProperty>();

  for (const item of raw['@graph']) {
    const id = item['@id']?.replace('schema:', '');
    if (!id) continue;

    if (item['@type'] === 'rdfs:Class') {
      const subClassOf = Array.isArray(item['rdfs:subClassOf'])
        ? item['rdfs:subClassOf'].map((s: any) => s['@id']?.replace('schema:', ''))
        : item['rdfs:subClassOf']?.['@id']
          ? [item['rdfs:subClassOf']['@id'].replace('schema:', '')]
          : [];

      types.set(id, {
        id,
        label: item['rdfs:label']?.['@value'] || item['rdfs:label'] || id,
        comment: item['rdfs:comment']?.['@value'] || item['rdfs:comment'] || '',
        subClassOf,
        properties: [],
        supersededBy: item['schema:supersededBy']?.['@id']?.replace('schema:', '')
      });
    }

    if (item['@type'] === 'rdf:Property') {
      const domainIncludes = normalizeToArray(item['schema:domainIncludes'])
        .map((d: any) => d['@id']?.replace('schema:', '')).filter(Boolean);
      const rangeIncludes = normalizeToArray(item['schema:rangeIncludes'])
        .map((r: any) => r['@id']?.replace('schema:', '')).filter(Boolean);

      properties.set(id, {
        id,
        label: item['rdfs:label']?.['@value'] || item['rdfs:label'] || id,
        comment: item['rdfs:comment']?.['@value'] || item['rdfs:comment'] || '',
        domainIncludes,
        rangeIncludes,
        supersededBy: item['schema:supersededBy']?.['@id']?.replace('schema:', '')
      });

      // Add property to its domain types
      for (const domain of domainIncludes) {
        const type = types.get(domain);
        if (type) type.properties.push(id);
      }
    }
  }

  return { types, properties, lastUpdated: new Date().toISOString() };
}

function normalizeToArray(val: any): any[] {
  if (!val) return [];
  return Array.isArray(val) ? val : [val];
}
```

---

## Scoring Algorithm

```
Total Score = weighted average of:
  - Syntax (30%): Are all structured data blocks valid?
  - Completeness (30%): Are rich-result-required fields present?
  - Best Practices (20%): Recommended fields, proper nesting, no duplicates
  - AEO Readiness (20%): FAQ schema, speakable, entity descriptions

Deductions:
  - Each critical error: -15 points (floor at 0)
  - Each warning: -5 points
  - Each info/suggestion: -1 point
  - Bonus: +5 for each rich-result-eligible entity (cap at +15)
```

---

## Frontend Display

### Tree View Component
The main output should be a collapsible tree showing the schema hierarchy:

```
v Organization (JSON-LD)                          VALID
  name: "Acme Corp"
  url: "https://acme.com"
  logo: "https://acme.com/logo.png"
  v address (PostalAddress)
    streetAddress: "123 Main St"
    addressLocality: "Dallas"
    addressRegion: "TX"
  v contactPoint (ContactPoint)
    telephone: "+1-555-0123"
    contactType: "customer service"
  [!] Missing: description                        WARNING
  [!] Missing: sameAs (social profiles)           INFO

v Product (JSON-LD)                               2 ISSUES
  name: "Super Widget"
  v offers (Offer)                                WARNING
    price: "29.99"
    [!!] Missing: priceCurrency                   CRITICAL
    [!!] Missing: availability                    CRITICAL
  [!] Missing: aggregateRating                    WARNING
  [!] Missing: review                             INFO
```

### Tabs
1. **All Items** -- Tree view of all extracted entities
2. **Rich Results** -- Which rich results are eligible/not, with Google's requirements
3. **AEO Analysis** -- AI-specific suggestions (FAQ schema, entity coverage, etc.)
4. **Raw Data** -- Syntax-highlighted JSON/HTML of extracted data
5. **Compare** -- Side-by-side with competitor (premium feature)

---

## Edge Cases to Handle

1. **Multiple JSON-LD blocks** -- Common. Validate each independently.
2. **@graph arrays** -- Unwrap and validate each entity in the graph.
3. **Mixed formats** -- Same page with JSON-LD AND Microdata. Show both, note potential conflicts.
4. **Broken JSON** -- Report the parse error, show the raw content, suggest fix.
5. **Relative URLs** -- Resolve against the page's base URL before validation.
6. **Very large pages** -- Cap HTML at 5MB. Truncate and warn.
7. **JavaScript-rendered schema** -- Offer a "Render JS" toggle (uses headless Chrome, costs more credits).
8. **Nested itemscope** -- Properly handle parent-child microdata relationships.
9. **Schema.org aliases** -- Some types have aliases (e.g., `Article` subclasses). Resolve via vocabulary.
10. **Non-schema.org structured data** -- Detect Open Graph, Twitter Cards, Dublin Core. Report them separately.
