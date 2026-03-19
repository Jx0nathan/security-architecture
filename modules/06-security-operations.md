# Module 06: Security Operations
**MAS TRM §12 Monitoring | §14 Incident Management**
Status: In Progress

---

## 1. SIEM Architecture (Elastic Security)

**Log Sources (must all feed into SIEM):**
```
Network Layer:
  → Cloudflare WAF logs (all requests + decisions)
  → AWS WAF logs
  → VPC Flow Logs
  → ALB access logs

Application Layer:
  → API Gateway access logs (all requests with user ID, IP, endpoint, status)
  → Application error logs (structured JSON, no PII in logs)
  → K8s audit logs
  → Istio access logs (inter-service calls)

Data Layer:
  → RDS audit logs (all queries on sensitive tables)
  → Redis access logs
  → S3 access logs (especially KYC bucket)

Identity Layer:
  → SSO login events (success + failure)
  → AWS CloudTrail (all API calls)
  → Vault audit logs (all secret access)
  → SealSuite endpoint events

Security Tools:
  → SealSuite EDR alerts
  → AWS Security Hub findings
  → CSPM alerts

Log retention:
  → Hot (searchable): 90 days in Elasticsearch
  → Cold (S3 archive): 3 years total
  → PII in logs: must be masked before ingestion
```

---

## 2. Critical Alert Rules

**P1 — Immediate Response Required (15 min SLA)**

```
Rule: API_BULK_SCRAPING
Trigger: Single IP > 1000 requests to same endpoint in 10 minutes
Action:  Auto-block at Cloudflare + PagerDuty alert

Rule: INTERNAL_API_PUBLIC_ACCESS
Trigger: Any HTTP 200 response from /internal/ endpoint
         where source IP is not in private subnet CIDR
Action:  IMMEDIATE - auto-block + PagerDuty + call CISO

Rule: MASS_DATA_EXPORT
Trigger: Single user/session retrieves > 500 unique user records in 1 hour
Action:  Auto-suspend session + PagerDuty alert

Rule: PRIVILEGED_AFTER_HOURS
Trigger: Admin/root login between 23:00-07:00 SGT from unusual IP
Action:  Require MFA re-auth + alert Security team

Rule: NEW_ADMIN_CREATED
Trigger: New IAM admin role assigned or new admin account created
Action:  Alert CISO + require approval within 1 hour
```

**P2 — Investigate Within 2 Hours**

```
Rule: INJECTION_PROBE
Trigger: WAF blocks > 20 injection attempts from same IP in 1 hour
Action:  Extend block to 24h + alert SOC analyst

Rule: REPEATED_AUTH_FAILURE
Trigger: > 10 failed logins for same account in 5 minutes
Action:  Temporarily lock account + alert

Rule: LARGE_FILE_DOWNLOAD
Trigger: User downloads > 100MB from internal systems in 1 session
Action:  Alert SOC for review

Rule: CERTIFICATE_EXPIRY
Trigger: Any TLS certificate expiring within 14 days
Action:  Alert DevOps + Security
```

---

## 3. SOC Structure & Escalation

```
L1 — Alert Triage (24×7)
  → Review incoming SIEM alerts
  → Classify: Real incident / False positive / Needs investigation
  → Escalate to L2 within 15 min for P1 alerts
  → Response SLA: 15 min for P1, 2 hr for P2, next business day for P3

L2 — Incident Investigation
  → Deep-dive investigation of confirmed incidents
  → Coordinate with system owners for containment
  → Evidence preservation (logs, memory dumps)
  → Escalate to L3 if scope is expanding

L3 — Incident Commander (CISO / Senior Security Lead)
  → Major incident management
  → MAS notification decision
  → External party engagement (Blackpanda, MAS)
  → Executive communication
```

---

## 4. Incident Response SOP (MAS TRM §14)

```
T+0:00   Detection
         → SIEM alert fires / user reports / external notification
         → L1 analyst receives alert

T+0:15   Initial Triage
         → L1 assesses severity
         → If P1: escalate to L2 + notify L3 (CISO)
         → Start incident ticket (mandatory - evidence chain)

T+1:00   ⚠️ MAS NOTIFICATION DEADLINE (for major incidents)
         → L3 decides: is this reportable to MAS?
         Major incident criteria:
           • Core system unavailable > 30 min
           • Any customer data exposure (any volume)
           • Unauthorized system access
           • Fraud affecting customer funds
         → If yes: CISO calls MAS (+65 6225 5577 or designated contact)
         → Document: who called, what time, what was said

T+4:00   Containment
         → Isolate affected systems
         → Preserve evidence (snapshot, logs)
         → Implement temporary fixes

T+24:00  ⚠️ MAS WRITTEN REPORT DEADLINE
         → Submit initial written report to MAS
         → Include: timeline, impact scope, containment actions taken

T+ongoing Daily MAS updates until incident closed

T+30d    Post-Incident Report to MAS
(after   → Full RCA
closure) → Permanent remediation plan
         → Lessons learned
         → Control improvements implemented
```

---

## 5. EDR / SealSuite Configuration

**Monitoring scope:**
→ All corporate devices (laptop, workstation)
→ Server endpoints (where agent can be installed)

**Alerts to configure in SealSuite:**
```
→ USB device connected: ALERT + require approval for data copy
→ Screen capture on sensitive applications (trading, card admin): BLOCK
→ Upload to personal cloud storage (Google Drive, Dropbox): BLOCK
→ Large file transfer (>50MB) via email: CHALLENGE
→ Application installed outside approved list: ALERT
→ Device policy compliance failed: ISOLATE from network
```

---

## 6. CSPM — AWS Security Hub

**Required checks (enable all):**
```
→ AWS Foundational Security Best Practices
→ CIS AWS Foundations Benchmark
→ PCI DSS v3.2.1

Critical findings to fix within 24 hours:
  → S3 bucket public access
  → Security Group: 0.0.0.0/0 on port 22 or 3306 or 5432
  → RDS snapshot public
  → IAM root account used recently
  → MFA not enabled on root account
  → CloudTrail disabled in any region
```

---

## 7. VAPT Schedule (MAS TRM §8)

```
Annual External VAPT:
  → Scope: All production-facing APIs, web applications, infrastructure
  → Provider: Halborn or Blackpanda (minimum scope: 10 person-days)
  → Timing: Q4 each year + after major changes
  → Report delivered within 2 weeks of testing completion
  → Critical findings: remediated within 14 days
  → High findings: remediated within 30 days

Quarterly Internal Scans:
  → Automated vulnerability scanning (AWS Inspector + Nessus)
  → Scope: All EC2 instances, RDS, container images
  → Results reviewed by Security team within 1 week

Smart Contract Audit:
  → Annual audit of stablecoin settlement contracts
  → Provider: OpenZeppelin or Halborn
  → Before any major contract upgrade
```
