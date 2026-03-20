# DCS Card Centre — Security Architecture

**Compliant with MAS TRM 2021 · PDPA · PCI DSS v4.0**

> ⚠️ Confidential — Internal use only. Do not distribute externally.

---

## Architecture Diagrams

### Overall Security Architecture

![Overall Architecture](architecture/overall-architecture.png)

### Security Tooling — Traffic Flow & Layer Controls

![Security Tools](architecture/security-tools.png)

**Traffic flow (top → bottom):**

```
🌐 Internet  (Browser · Mobile App · Partner API · Attacker)
        │  HTTPS / TLS 1.3
        ▼
🛡 Cloudflare Edge
   Cloudflare WAF (Managed Ruleset)  ·  DDoS Shield Advanced
   DNS / DNSSEC  ·  TLS 1.3 Termination  ·  Bot Management
        │  Origin Pull · TLS
        ▼
🔥 AWS Perimeter
   ALB  ·  AWS WAF (OWASP CRS)  ·  AWS Shield Advanced
   ACM Certificate  ·  Rate Limiting  ·  Geo Blocking
        │  HTTP/2 · VPC Internal
        ▼
🚦 Kubernetes Ingress  (Traefik)
   Namespace Routing  ·  Istio mTLS  ·  JWT Validation
        │  Namespace Route · mTLS
        ▼
⚙️ Application Services  (K8s + Istio Sidecar)
   Istio mTLS  ·  OAuth2 + JWT  ·  SonarCloud SAST
   Snyk SCA  ·  42Crunch API Audit  ·  OWASP ZAP DAST
        │  DB / Cache · TLS
        ▼
🗄 Data Layer
   RDS PostgreSQL (AES-256-GCM)  ·  Redis (TLS)  ·  S3 (SSE-KMS)
   AWS KMS  ·  HCP Vault  ·  AWS Macie  ·  SealSuite DLP
```

All layers emit logs → **Elastic Cloud SIEM** → PagerDuty / Tines SOAR → SOC

---

## Repository Structure

```
architecture/
  overall-architecture.png          ← Full security architecture diagram
  security-tools.png                ← Security tooling by layer (traffic flow)

security-implementation-guide/
  01-边界安全.md                     ← Perimeter Security       (MAS TRM §9)
  02-内网安全.md                     ← Network Security          (MAS TRM §9)
  03-数据安全.md                     ← Data Security             (MAS TRM §13 · PDPA · PCI DSS)
  04-身份与访问管理.md                ← Identity & Access Mgmt   (MAS TRM §10)
  05-安全运营.md                     ← Security Operations       (MAS TRM §12 · §14)

mas-trm/
  MAS TRM Guidelines.pdf            ← MAS TRM 2021 official source document
```

---

## Security Tooling — SaaS & Managed Only

All tools selected are SaaS or fully managed — no self-hosted infrastructure required.

**Perimeter & Edge**
- Cloudflare WAF — Managed Ruleset (SaaS)
- Cloudflare DDoS Shield Advanced — L3/L4/L7
- AWS WAF — OWASP Core Rule Set (managed)
- AWS Shield Advanced — DDoS managed service
- AWS ACM — TLS certificate auto-renew (managed)

**Network & Ingress**
- Traefik — Namespace-level routing
- Istio Service Mesh — mTLS all pod-to-pod traffic (managed)
- JWT Validation — enforced per ingress endpoint

**Application Security**
- SonarCloud — SAST static analysis (SaaS)
- Snyk SCA — dependency + image vulnerability scan (SaaS)
- OWASP ZAP — DAST pre-release scan
- 42Crunch — OpenAPI spec security audit (SaaS)

**Data Security**
- AWS KMS — master key management (managed)
- HCP Vault — dynamic secrets + PAM (SaaS)
- HCP Vault Transit — card number tokenization (SaaS)
- AWS Macie — PII auto-classification (managed)
- SealSuite DLP — endpoint data control (SaaS)

**Identity & Access Management**
- Okta SSO + MFA — Identity Provider, TOTP/Push (SaaS)
- HCP Vault — Just-in-Time access, secret rotation (SaaS)
- AWS IAM — role-based access, no hardcoded credentials
- AWS CloudTrail — full API audit log (managed)

**Security Operations**
- Elastic Cloud SIEM — log aggregation, correlation, Kibana (managed)
- SealSuite EDR — endpoint detection + response (SaaS)
- AWS Security Hub — CSPM, GuardDuty, Inspector (managed)
- Recorded Future — threat intelligence feed (SaaS)
- PagerDuty — P1/P2 on-call alert + phone escalation (SaaS)
- Tines SOAR — automated response playbooks (SaaS)
- Tenable.io — VAPT scanning, annual + post-change (SaaS)

**CI/CD Security Pipeline**
- SonarCloud → Snyk → OWASP ZAP → 42Crunch → Release Gate

**Compliance & GRC**
- Vanta — ISO 27001 / SOC 2 compliance automation (SaaS)
- Cobalt.io — PTaaS penetration testing (SaaS)
- Trustwave — PCI DSS QSA (managed service)
- Jira Cloud — unified issue and risk tracking (SaaS)

---

## MAS TRM 2021 — Key Compliance Requirements

**§6 Governance & Oversight**
- CISO reports independently to CEO / Board
- IT risk management framework formally established, annual review

**§8 System Security**
- CIS Benchmark hardening applied to all systems
- VAPT annually + after every major change
- SDL (Secure Development Lifecycle) embedded in engineering

**§9 Network Security**
- WAF with OWASP CRS at perimeter
- Network segmentation: Public / Private / Data / Management subnets
- Internal API namespaces must not be exposed to public internet
- All network config changes require CAB approval

**§10 Identity & Access Management**
- MFA mandatory for all production system access
- Principle of least privilege enforced
- PAM via HCP Vault (Just-in-Time access)
- Quarterly access review for all privileged accounts

**§11 Business Continuity**
- Critical system availability: ≥ 99.95% (≤ 4.4 hrs downtime/year)
- RTO ≤ 4 hours · RPO ≤ 4 hours
- Full DR drill annually

**§12 Security Monitoring**
- SIEM 24×7 alert coverage
- Real-time anomaly detection on all API traffic
- P1 alert response: ≤ 15 minutes

**§14 Incident Management**
- Initial notification to MAS: within 1 hour (phone)
- Written report to MAS: within 24 hours
- Full RCA submission: within 30 days post-closure

---

## Implementation Status

| Module | Status | Target |
|--------|--------|--------|
| WAF (Cloudflare + AWS OWASP CRS) | 🔄 In Progress | Q1 2026 |
| Elastic Cloud SIEM | 🔄 In Progress | Q1 2026 |
| HCP Vault (secrets management) | 📋 Planned | Q2 2026 |
| Okta SSO + MFA | 🔄 In Progress | Q1 2026 |
| SonarCloud + Snyk (CI/CD) | 📋 Planned | Q2 2026 |
| AWS Macie (PII scanning) | 📋 Planned | Q2 2026 |
| Vanta (ISO 27001 automation) | 📋 Planned | Q3 2026 |
| Tines SOAR | 📋 Planned | Q3 2026 |
| ISO 27001 Certification | 📋 Planned | Q4 2026 |

---

## Incident Reference

**Feb 2026 — API Exposure Incident (Resolved)**

Root cause: WAF IP allowlist removed Dec 3, 2025 → `/internal/` API namespace exposed to public internet.

- First unauthorized access: Feb 19, 2026 10:07 SGT
- Last unauthorized access: Feb 20, 2026 05:52 SGT
- Detection gap: 12+ hours (no real-time API anomaly alerting)
- Scale: 69,945 requests · 23,079 PII records + 12,000+ financial records exposed
- Assessment: opportunistic, not targeted

Remediations driven by this incident:
- WAF OWASP CRS re-enabled and hardened
- Elastic SIEM deployment fast-tracked
- `/internal/` namespace routing blocked at Traefik ingress
- MAS TRM §9 and §12 gap remediation in progress

---

*Last updated: March 2026 | Owner: Head of Technology*
