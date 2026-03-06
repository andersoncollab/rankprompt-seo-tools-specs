# Tool 16: SSL/TLS Security Check

## What This Replicates
- SSL Labs (ssllabs.com/ssltest)
- SecurityHeaders.com
- WhyNoPadlock.com

## What It Does
Checks SSL/TLS certificate validity, configuration, and common HTTPS issues. Reports certificate expiry, protocol versions, mixed content, and HSTS status. Lightweight version of SSL Labs focused on SEO impact.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-ssl-check
```

### Request
```json
{ "url": "https://example.com" }
```

### Response
```json
{
  "score": 92,
  "certificate": {
    "valid": true,
    "issuer": "Let's Encrypt Authority X3",
    "subject": "example.com",
    "subjectAltNames": ["example.com", "www.example.com"],
    "validFrom": "2024-01-15T00:00:00Z",
    "validTo": "2024-04-15T00:00:00Z",
    "daysUntilExpiry": 41,
    "serialNumber": "03:a1:...",
    "signatureAlgorithm": "SHA256withRSA",
    "keySize": 2048
  },
  "protocol": {
    "version": "TLSv1.3",
    "cipher": "TLS_AES_256_GCM_SHA384"
  },
  "https": {
    "httpToHttpsRedirect": true,
    "hstsEnabled": true,
    "hstsMaxAge": 63072000,
    "hstsIncludeSubDomains": true,
    "hstsPreload": true,
    "mixedContent": false
  },
  "issues": [
    {
      "code": "CERT_EXPIRY_SOON",
      "severity": "warning",
      "title": "Certificate expires in 41 days",
      "message": "Renew your SSL certificate before April 15, 2024. An expired certificate blocks all traffic and destroys search rankings."
    }
  ]
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `CERT_INVALID` | critical | Certificate is not valid |
| `CERT_EXPIRED` | critical | Certificate has expired |
| `CERT_EXPIRY_SOON` | warning | Expires within 30 days |
| `CERT_SELF_SIGNED` | critical | Self-signed certificate |
| `CERT_DOMAIN_MISMATCH` | critical | Certificate doesn't match domain |
| `CERT_WEAK_KEY` | warning | RSA key smaller than 2048 bits |
| `NO_HTTPS` | critical | Site doesn't support HTTPS |
| `NO_HTTP_REDIRECT` | warning | HTTP doesn't redirect to HTTPS |
| `MIXED_CONTENT` | warning | HTTPS page loads HTTP resources |
| `HSTS_MISSING` | warning | No HSTS header |
| `HSTS_SHORT` | info | HSTS max-age less than 1 year |
| `TLS_OLD_VERSION` | warning | Using TLSv1.0 or TLSv1.1 |
| `AEO_HTTPS_REQUIRED` | passed | HTTPS is a baseline signal for AI engines to trust content |
| `AEO_CERT_EXPIRY_RISK` | warning | If cert expires, AI crawlers will fail silently |

---

## Implementation Note

For Edge Functions (Deno), you can get basic TLS info from the `fetch` response. For full certificate details, you may need a Cloud Run service with Node.js `tls` module, or use an external API like `https://api.ssllabs.com/api/v3/analyze`.

Simpler approach: just check if HTTPS works, follow the redirect from HTTP, check HSTS header, and test for mixed content by parsing the HTML for `http://` resource URLs.

---

## Scoring

```
Score starts at 100:
- No HTTPS: -50
- Expired certificate: -40
- Self-signed: -30
- Domain mismatch: -30
- Expiry within 30 days: -10
- No HTTP->HTTPS redirect: -15
- Mixed content: -10
- No HSTS: -5
- Old TLS version: -10
```
