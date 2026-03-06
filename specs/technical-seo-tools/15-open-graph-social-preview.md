# Tool 15: Open Graph & Social Share Preview

## What This Replicates
- Metatags.io social previews
- Facebook Sharing Debugger
- Twitter Card Validator
- LinkedIn Post Inspector

## What It Does
Extracts all social sharing metadata (Open Graph, Twitter Card) and generates visual previews showing exactly how the page will look when shared on Facebook, Twitter/X, LinkedIn, Discord, Slack, and iMessage. Also validates image dimensions and aspect ratios.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-social-preview
```

### Request
```json
{
  "url": "https://example.com/blog/crm-guide"
}
```

### Response
```json
{
  "score": 80,
  "metadata": {
    "og": {
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM software.",
      "image": "https://example.com/images/crm-og.jpg",
      "imageWidth": 1200,
      "imageHeight": 630,
      "url": "https://example.com/blog/crm-guide",
      "type": "article",
      "siteName": "Example Blog",
      "locale": "en_US"
    },
    "twitter": {
      "card": "summary_large_image",
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM.",
      "image": "https://example.com/images/crm-tw.jpg",
      "site": "@exampleblog",
      "creator": "@authorhandle"
    }
  },
  "previews": {
    "facebook": {
      "imageUrl": "https://example.com/images/crm-og.jpg",
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM software.",
      "domain": "example.com",
      "imageAspectRatio": "1.91:1",
      "imageValid": true
    },
    "twitter": {
      "cardType": "summary_large_image",
      "imageUrl": "https://example.com/images/crm-tw.jpg",
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM.",
      "domain": "example.com",
      "imageValid": true
    },
    "linkedin": {
      "imageUrl": "https://example.com/images/crm-og.jpg",
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM software.",
      "domain": "example.com",
      "imageRecommendation": "1200x627 recommended for LinkedIn"
    },
    "discord": {
      "embedColor": null,
      "siteName": "Example Blog",
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM software.",
      "imageUrl": "https://example.com/images/crm-og.jpg",
      "themeColor": null
    },
    "slack": {
      "title": "The Ultimate CRM Guide 2024",
      "description": "Everything you need to know about CRM software.",
      "imageUrl": "https://example.com/images/crm-og.jpg",
      "favicon": "https://example.com/favicon.ico"
    }
  },
  "issues": [
    {
      "code": "OG_IMAGE_ASPECT_RATIO",
      "severity": "info",
      "title": "og:image is 1200x630 (1.90:1). Ideal is 1.91:1 (1200x628)",
      "message": "Minor difference, will display fine on all platforms."
    },
    {
      "code": "TW_DESC_TRUNCATED",
      "severity": "info",
      "title": "Twitter description may be truncated on mobile",
      "message": "Twitter shows ~125 characters on mobile. Your description is 47 characters, so it fits."
    }
  ]
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `OG_MISSING_REQUIRED` | critical | Missing og:title, og:type, og:image, or og:url |
| `OG_IMAGE_NOT_FOUND` | critical | og:image URL returns 404 |
| `OG_IMAGE_TOO_SMALL` | warning | Image smaller than 1200x630 |
| `OG_IMAGE_WRONG_RATIO` | info | Image not 1.91:1 aspect ratio |
| `OG_IMAGE_TOO_LARGE` | warning | Image file larger than 8MB (Facebook limit) |
| `TW_CARD_MISSING` | warning | No twitter:card meta tag |
| `TW_IMAGE_NOT_FOUND` | warning | twitter:image URL returns 404 |
| `SOCIAL_NO_METADATA` | critical | No Open Graph or Twitter Card tags at all |
| `SOCIAL_TITLE_TOO_LONG` | warning | Title exceeds 60 characters (may truncate) |
| `SOCIAL_DESC_TOO_LONG` | warning | Description exceeds 155 characters |
| `SOCIAL_MISSING_SITE_NAME` | info | No og:site_name |

---

## Frontend: Visual Previews

Show pixel-accurate mockups of how the URL will appear when shared. Use CSS to replicate each platform's card design:

```
+------------------------------------------------------------+
| Facebook Preview                                            |
| +-------------------------------------------------------+  |
| | [========== IMAGE 1200x630 ===========]               |  |
| | EXAMPLE.COM                                           |  |
| | The Ultimate CRM Guide 2024                          |  |
| | Everything you need to know about CRM software.       |  |
| +-------------------------------------------------------+  |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Twitter/X Preview (Large Image Card)                        |
| +-------------------------------------------------------+  |
| | [========== IMAGE ===========]                        |  |
| | The Ultimate CRM Guide 2024                          |  |
| | Everything you need to know about CRM.                |  |
| | example.com                                           |  |
| +-------------------------------------------------------+  |
+------------------------------------------------------------+

+------------------------------------------------------------+
| LinkedIn Preview                                            |
| +-------------------------------------------------------+  |
| | [========== IMAGE ===========]                        |  |
| | The Ultimate CRM Guide 2024                          |  |
| | example.com                                           |  |
| +-------------------------------------------------------+  |
+------------------------------------------------------------+
```

Include a "Copy Share URL" button and "Test on Platform" links (to Facebook Debugger, Twitter Validator, LinkedIn Inspector).

---

## Scoring

```
Score starts at 100:
- No OG tags at all: -40
- Missing og:image: -20
- Missing og:title: -15
- Missing og:description: -10
- Image returns 404: -20
- Image too small: -10
- No Twitter Card tags: -10
- Twitter image returns 404: -10
- Title too long: -3
- Description too long: -3
```
