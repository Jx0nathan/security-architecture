# Module 01: Perimeter Security
**MAS TRM §9 — Network Boundary Controls**
Status: Partially Implemented (post-incident hardening in progress)

---

## 1. Cloudflare WAF Configuration

**Required settings:**
```
Security Level: Medium (not Low)
Bot Fight Mode: ON
Browser Integrity Check: ON
Hotlink Protection: ON

WAF Managed Rules:
  → Cloudflare OWASP Core Ruleset: ENABLED (Sensitivity: Medium)
  → Cloudflare Managed Ruleset: ENABLED
  → Cloudflare Exposed Credentials Check: ENABLED

Rate Limiting Rules:
  → Same IP > 100 req/min to /public/: CHALLENGE
  → Same IP > 50 req/min to /auth/: BLOCK for 1 hour
  → Same IP > 10 req/min to /user/public/v1/pre-login: CHALLENGE
  → Same IP > 200 req/min any path: BLOCK for 24 hours

Custom Rules (must have):
  → BLOCK: URI contains sleep( OR ${  OR #{  OR .to_i  (injection probes)
  → BLOCK: URI contains ..;/ OR //internal (path traversal)
  → BLOCK: User-Agent matches python-urllib OR python-requests (automation)
  → CHALLENGE: Requests without standard browser headers to sensitive endpoints
```

**IP Whitelist Policy:**
```
CRITICAL: Any change to IP whitelist rules requires:
  → Change Advisory Board (CAB) approval ticket
  → Two-person authorization
  → Written justification
  → Auto-expiry after 90 days (must renew)
  → Change logged in audit system

[This is the root cause of Feb 2026 incident]
```

---

## 2. AWS WAF (OWASP CRS)

**Configuration:**
```
Web ACL Rules (in order):
  1. AWS-AWSManagedRulesCommonRuleSet        → Block SQLi, XSS, RFI
  2. AWS-AWSManagedRulesKnownBadInputsRuleSet → Block log4j, SSRF
  3. AWS-AWSManagedRulesAdminProtectionRuleSet → Protect /admin/ paths
  4. AWS-AWSManagedRulesSQLiRuleSet           → Additional SQL injection
  5. Custom: BlockInternalNamespace           → BLOCK PathPrefix /internal/
  6. Custom: RateLimit-100rpm                 → Rate limit per IP

Logging:
  → All WAF decisions logged to S3 + CloudWatch
  → Retention: 1 year
  → Alert on: >10 BLOCK decisions from same IP in 5 minutes
```

**Post-incident mandatory rule:**
```json
{
  "Name": "BlockInternalAPIFromPublic",
  "Priority": 1,
  "Statement": {
    "ByteMatchStatement": {
      "SearchString": "/internal/",
      "FieldToMatch": {"UriPath": {}},
      "TextTransformations": [{"Priority": 0, "Type": "LOWERCASE"}],
      "PositionalConstraint": "CONTAINS"
    }
  },
  "Action": {"Block": {}},
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockInternalAPI"
  }
}
```

---

## 3. TLS Configuration

**Required TLS policy (AWS ALB):**
```
Security Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
Minimum TLS version: 1.2
TLS 1.3: Enabled
Disabled: SSLv3, TLS 1.0, TLS 1.1

Cipher suites (allowed):
  TLS_AES_128_GCM_SHA256 (TLS 1.3)
  TLS_AES_256_GCM_SHA384 (TLS 1.3)
  TLS_CHACHA20_POLY1305_SHA256 (TLS 1.3)
  ECDHE-RSA-AES128-GCM-SHA256 (TLS 1.2)
  ECDHE-RSA-AES256-GCM-SHA384 (TLS 1.2)
```

**Security Headers (must be present on all responses):**
```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'; script-src 'self'
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## 4. DDoS Protection

→ Cloudflare: Layer 3/4 DDoS protection (always on, free tier)
→ AWS Shield Standard: automatically applied to ALB/CloudFront
→ AWS Shield Advanced: [EVALUATE - recommended for financial institution]
→ Rate limiting rules provide application-layer (Layer 7) protection

---

## 5. Certificate Management

→ External certificates: AWS ACM (auto-renewal)
→ Internal certificates: Issued by Istio CA (auto-rotation every 90 days)
→ Certificate inventory: maintained in AWS Certificate Manager
→ Expiry monitoring: CloudWatch alarm 30 days before expiry
→ No self-signed certificates on production endpoints

---

## 6. MAS TRM §9 Compliance Status

→ WAF deployed at perimeter: [IMPLEMENTED]
→ OWASP CRS enabled: [IN PROGRESS - deploying]
→ DDoS protection: [IMPLEMENTED - basic]
→ TLS 1.2+ enforced: [IMPLEMENTED]
→ Network change management process: [IN PROGRESS - CAB being established]
→ Internal namespace segregation: [IMPLEMENTED - post-incident]
→ Regular perimeter security review: [PLANNED - quarterly]
