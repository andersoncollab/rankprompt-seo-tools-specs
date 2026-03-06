# Tool 10: Content Citability Scorer

## What This Replicates
- Nothing directly (this is novel IP for RankPrompt)
- Loosely inspired by Clearscope content scoring, MarketMuse content analysis
- Based on research into how AI engines select passages for citation

## What It Does
Analyzes a page's content structure and determines how "citable" it is by AI engines. AI assistants (ChatGPT, Perplexity, Claude) tend to cite content that follows specific patterns: clear definitions, structured data, authoritative claims, factual lists, and well-formatted Q&A. This tool scores content against those patterns and provides specific rewrite suggestions.

This is RankPrompt's most differentiated tool. Nobody else does this.

---

## How AI Engines Select Content to Cite

Based on analysis of AI citation patterns, AI engines preferentially cite content that has:

1. **Definition Blocks**: Clear "X is Y" statements at the beginning of sections
2. **Structured Lists**: Numbered steps, bulleted lists with consistent formatting
3. **Statistical Claims**: Specific numbers, percentages, dates
4. **Expert Attribution**: Quotes or claims attributed to named experts
5. **Direct Answers**: Content that directly answers a question in 1-3 sentences
6. **Comparison Tables**: Structured comparisons (vs, pros/cons)
7. **Recency Signals**: Recent dates, "updated" timestamps, current year references
8. **Unique Data**: Original research, proprietary statistics, case study results
9. **Concise Paragraphs**: Short (2-4 sentence) paragraphs rather than walls of text
10. **Semantic Headers**: H2/H3 headers that match common search queries

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-citability-scorer
```

### Request
```json
{
  "url": "https://example.com/blog/best-crm-software-2024",
  "options": {
    "renderJS": false,
    "includeRewriteSuggestions": true,
    "targetQuery": "what is the best CRM software"
  }
}
```

### Response
```json
{
  "score": 58,
  "grade": "C+",
  "url": "https://example.com/blog/best-crm-software-2024",
  "contentStats": {
    "wordCount": 2847,
    "paragraphCount": 34,
    "avgParagraphLength": 83,
    "headerCount": { "h1": 1, "h2": 8, "h3": 12, "h4": 2 },
    "listCount": 5,
    "tableCount": 1,
    "imageCount": 6,
    "linkCount": { "internal": 12, "external": 8 }
  },
  "citabilityFactors": {
    "definitionBlocks": {
      "score": 40,
      "maxScore": 100,
      "found": 1,
      "ideal": 3,
      "details": "Only 1 clear definition block found ('CRM software is a tool that...'). AI engines look for concise definitions in the first paragraph of each major section.",
      "examples": [
        {
          "location": "Paragraph 3",
          "text": "CRM software is a tool that helps businesses manage customer relationships, track interactions, and automate sales processes.",
          "quality": "good"
        }
      ]
    },
    "directAnswers": {
      "score": 30,
      "maxScore": 100,
      "found": 0,
      "ideal": 2,
      "details": "No direct answer to the target query found in the first 200 words. AI engines strongly prefer pages that answer the query immediately, then elaborate.",
      "suggestion": "Add a direct answer in the intro: 'The best CRM software in 2024 is [X] for most businesses, followed by [Y] for enterprises and [Z] for small teams.'"
    },
    "structuredLists": {
      "score": 75,
      "maxScore": 100,
      "found": 5,
      "ideal": 4,
      "details": "Good use of structured lists. 3 are numbered (step-by-step), 2 are bulleted (features).",
      "quality": "3/5 lists have clear, concise items. 2 lists have items over 50 words (too verbose for AI extraction)."
    },
    "statisticalClaims": {
      "score": 60,
      "maxScore": 100,
      "found": 4,
      "ideal": 6,
      "details": "4 specific statistics found. More data points increase citation likelihood.",
      "examples": [
        { "text": "91% of companies with 10+ employees use a CRM", "hasSource": true, "source": "Gartner 2023" },
        { "text": "CRM market expected to reach $88.2B by 2025", "hasSource": true, "source": "Grand View Research" },
        { "text": "average ROI of $8.71 per dollar spent", "hasSource": false },
        { "text": "saves 5+ hours per week per sales rep", "hasSource": false }
      ]
    },
    "expertAttribution": {
      "score": 20,
      "maxScore": 100,
      "found": 0,
      "ideal": 2,
      "details": "No expert quotes or named attribution found. AI engines give more weight to attributed claims.",
      "suggestion": "Add quotes from industry experts, customer testimonials with names, or cite specific analysts."
    },
    "comparisonStructure": {
      "score": 65,
      "maxScore": 100,
      "found": 1,
      "ideal": 2,
      "details": "1 comparison table found. Consider adding a quick comparison summary at the top of the article.",
      "tableQuality": "Table has 5 products, 4 comparison columns. Good structure but missing pricing column."
    },
    "recencySignals": {
      "score": 80,
      "maxScore": 100,
      "found": 3,
      "details": "Title includes '2024', article date is recent, and content references current pricing.",
      "lastModified": "2024-02-15",
      "freshness": "Recent content. Good for AI citation."
    },
    "semanticHeaders": {
      "score": 70,
      "maxScore": 100,
      "matchesQueries": 6,
      "totalHeaders": 20,
      "details": "6 of 20 headers match common search queries. This is good but could be improved.",
      "goodHeaders": [
        "What is CRM Software?",
        "How Much Does CRM Software Cost?",
        "Best CRM Software for Small Business"
      ],
      "improvableHeaders": [
        { "current": "Our Top Picks", "suggested": "Best CRM Software of 2024 (Top 10)" },
        { "current": "Features to Look For", "suggested": "What Features Should a CRM Have?" }
      ]
    },
    "paragraphQuality": {
      "score": 55,
      "maxScore": 100,
      "details": "Average paragraph length is 83 words. AI engines prefer 40-60 word paragraphs for easier extraction.",
      "tooLong": 12,
      "optimal": 18,
      "tooShort": 4
    },
    "uniqueData": {
      "score": 30,
      "maxScore": 100,
      "found": 0,
      "details": "No original research, proprietary data, or unique case study results detected. Pages with original data are cited 3x more often by AI engines.",
      "suggestion": "Add original data: survey results, internal benchmarks, customer case study metrics, or proprietary analysis."
    }
  },
  "overallBreakdown": {
    "excellent": ["structuredLists", "recencySignals"],
    "good": ["semanticHeaders", "comparisonStructure", "statisticalClaims"],
    "needsWork": ["paragraphQuality", "definitionBlocks", "directAnswers"],
    "poor": ["uniqueData", "expertAttribution"]
  },
  "topRecommendations": [
    {
      "priority": 1,
      "title": "Add a direct answer in the introduction",
      "impact": "high",
      "effort": "low",
      "instruction": "In the first paragraph, directly state your top pick: 'The best CRM software in 2024 is Salesforce for enterprises, HubSpot for growing teams, and Zoho for budget-conscious businesses.' AI engines prioritize pages that answer the query immediately."
    },
    {
      "priority": 2,
      "title": "Add expert quotes or named attribution",
      "impact": "high",
      "effort": "medium",
      "instruction": "Include 2-3 quotes from industry analysts, CRM consultants, or real customer testimonials with full names. 'According to John Smith, VP of Sales at Acme Corp, switching to HubSpot CRM increased our close rate by 23%.'"
    },
    {
      "priority": 3,
      "title": "Break up long paragraphs",
      "impact": "medium",
      "effort": "low",
      "instruction": "12 paragraphs exceed 80 words. Break these into 2-3 sentence blocks. AI engines extract individual passages, so shorter paragraphs are more citable."
    },
    {
      "priority": 4,
      "title": "Add original data or research",
      "impact": "very high",
      "effort": "high",
      "instruction": "Survey your customers or analyze your own data. 'We surveyed 500 CRM users and found that...' Pages with original research are cited 3x more often by AI assistants."
    },
    {
      "priority": 5,
      "title": "Source your statistics",
      "impact": "medium",
      "effort": "low",
      "instruction": "2 of your 4 statistics lack source attribution. Add 'according to [Source]' to give AI engines confidence in citing these claims."
    }
  ],
  "targetQueryAnalysis": {
    "query": "what is the best CRM software",
    "intentMatch": "informational_comparison",
    "directAnswerPresent": false,
    "topPassageForQuery": {
      "text": "When it comes to choosing the best CRM software, several factors matter: ease of use, pricing, integrations, and scalability.",
      "relevanceScore": 0.6,
      "citabilityScore": 0.3,
      "issue": "This passage answers indirectly. AI engines want a specific recommendation, not criteria."
    },
    "idealPassage": "The best CRM software in 2024 is Salesforce for large enterprises ($25-300/user/mo), HubSpot for growing teams (free-$120/user/mo), and Zoho CRM for budget-conscious businesses ($14-52/user/mo). Each excels in different areas: Salesforce offers the deepest customization, HubSpot the best free tier, and Zoho the best value."
  }
}
```

---

## Scoring Algorithm

```typescript
interface CitabilityWeights {
  definitionBlocks: 10;
  directAnswers: 20;       // Highest weight -- most important for AI citation
  structuredLists: 10;
  statisticalClaims: 10;
  expertAttribution: 10;
  comparisonStructure: 8;
  recencySignals: 8;
  semanticHeaders: 8;
  paragraphQuality: 8;
  uniqueData: 8;
}

// Total weights = 100
// Each factor scored 0-100, then weighted

function calculateCitabilityScore(factors: CitabilityFactors): number {
  const weights: Record<string, number> = {
    definitionBlocks: 10,
    directAnswers: 20,
    structuredLists: 10,
    statisticalClaims: 10,
    expertAttribution: 10,
    comparisonStructure: 8,
    recencySignals: 8,
    semanticHeaders: 8,
    paragraphQuality: 8,
    uniqueData: 8
  };

  let totalScore = 0;
  for (const [factor, weight] of Object.entries(weights)) {
    totalScore += (factors[factor].score / 100) * weight;
  }

  return Math.round(totalScore);
}

// Grade thresholds
// A: 85-100, B: 70-84, C: 55-69, D: 40-54, F: 0-39
```

---

## Content Analysis Implementation

```typescript
// Definition block detection
function findDefinitionBlocks(paragraphs: string[]): DefinitionBlock[] {
  const patterns = [
    /^(.+?)\s+(?:is|are|refers to|means|defines?)\s+(.+)/i,
    /^(?:A|An|The)\s+(.+?)\s+(?:is|are)\s+(.+)/i,
    /^(.+?):\s+(.{20,})/,  // "Term: definition" pattern
  ];

  return paragraphs.map((p, i) => {
    for (const pattern of patterns) {
      const match = p.match(pattern);
      if (match) {
        return { index: i, text: p, term: match[1], definition: match[2], quality: rateDefinition(p) };
      }
    }
    return null;
  }).filter(Boolean);
}

// Direct answer detection
function findDirectAnswers(text: string, targetQuery: string): DirectAnswer[] {
  // Look for passages that directly answer the query in 1-3 sentences
  const sentences = text.split(/[.!?]+/).filter(s => s.trim().length > 20);
  const queryKeywords = extractKeywords(targetQuery);

  return sentences.map((sentence, i) => {
    const relevance = calculateRelevance(sentence, queryKeywords);
    const isDirectAnswer = relevance > 0.6 && sentence.length < 300;
    return isDirectAnswer ? { text: sentence.trim(), index: i, relevance } : null;
  }).filter(Boolean);
}

// Statistical claim detection
function findStatisticalClaims(text: string): StatClaim[] {
  const patterns = [
    /(\d+(?:\.\d+)?%)\s+(?:of|increase|decrease|growth|decline)/gi,
    /\$(\d+(?:,\d{3})*(?:\.\d{2})?)\s*(?:billion|million|per|\/)/gi,
    /(\d+(?:,\d{3})*)\+?\s+(?:users|customers|companies|businesses|employees)/gi,
    /(\d+(?:\.\d+)?x)\s+(?:more|faster|better|increase)/gi,
    /(?:save[sd]?|reduc(?:e[sd]?|tion))\s+(?:up to\s+)?(\d+(?:\.\d+)?%?)/gi,
  ];

  const claims: StatClaim[] = [];
  for (const pattern of patterns) {
    let match;
    while ((match = pattern.exec(text)) !== null) {
      const surroundingText = text.substring(
        Math.max(0, match.index - 50),
        Math.min(text.length, match.index + match[0].length + 100)
      );
      const hasSource = /(?:according to|source:|per |via |from |,\s*\d{4})/i.test(surroundingText);
      claims.push({
        text: match[0],
        context: surroundingText.trim(),
        hasSource,
        sourceText: hasSource ? extractSourceAttribution(surroundingText) : null
      });
    }
  }
  return claims;
}
```

---

## Frontend

### Score Dashboard
Large citability score gauge (0-100) with grade letter. Breakdown donut chart showing factor contributions.

### Factor Cards
Each citability factor gets an expandable card:
```
+----------------------------------------------+
| Direct Answers                   30/100  [!] |
| AI engines want you to answer the query      |
| immediately. You currently bury the answer.  |
| > Found passages:                            |
|   [none that directly answer the query]      |
| > Suggestion:                                |
|   "Add to intro: The best CRM software..."   |
| [Apply Suggestion] [Dismiss]                 |
+----------------------------------------------+
```

### Passage Highlighter
Show the page content with highlighted passages:
- Green: High-citability passages
- Yellow: Moderate citability
- Red: Low citability (walls of text, vague claims)
- Blue: Statistical claims (with/without sources marked)

---

## Notes for Developer

- This tool benefits greatly from headless Chrome rendering for JS-rendered pages
- The "target query" analysis is the premium feature. It requires NLP to match query intent to page content
- Start with a simpler version: just the structural analysis (definition blocks, lists, paragraph length, stats). Add the query-specific analysis later
- Consider using RankPrompt's existing Perplexity API integration to test: "If I ask Perplexity [target query], which passage would it most likely cite?"
- This tool should cost 5 credits (premium) since it involves deep content analysis
