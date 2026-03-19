# MAS TRM 2021 — Control Mapping
**DCS Card Centre Implementation Status**
Last updated: 2026-03-15

Legend: ✅ IMPLEMENTED | 🔄 IN PROGRESS | 📋 PLANNED | ❌ GAP

---

## §6 — Governance and Oversight

**§6.1 Board and Senior Management Responsibility**
→ Board must approve IT risk appetite and strategy
  Status: 📋 PLANNED — IT Risk policy pending board approval
→ Senior management accountable for IT risk management
  Status: 🔄 IN PROGRESS — CISO role being established

**§6.2 IT Risk Management Framework**
→ Formal IT Risk Management Framework documented and reviewed annually
  Status: 📋 PLANNED
→ IT risk included in enterprise risk management
  Status: 📋 PLANNED

**§6.3 Chief Information Security Officer (CISO)**
→ Dedicated CISO with direct access to board
  Status: 🔄 IN PROGRESS — Recruiting
→ CISO reports independently of CTO
  Status: 📋 PLANNED (currently reporting structure TBD)

---

## §8 — IT Security

**§8.1 System Security**
→ Hardening standards for all systems (OS, middleware, database)
  Status: 🔄 IN PROGRESS
→ Patch management: Critical patches within 14 days
  Status: 🔄 IN PROGRESS — Process being formalized
→ Security testing before production deployment
  Status: 🔄 IN PROGRESS — SDL being implemented

**§8.2 Vulnerability Assessment and Penetration Testing**
→ VAPT at least annually, after significant changes
  Status: 📋 PLANNED — Q4 2026 first formal VAPT
→ VAPT by competent independent party
  Status: 📋 PLANNED — Evaluating Halborn / Blackpanda

**§8.3 Security Patch Management**
→ Critical patches applied within 14 days
  Status: 🔄 IN PROGRESS
→ High risk patches within 30 days
  Status: 🔄 IN PROGRESS
→ Patch compliance tracking
  Status: 📋 PLANNED

---

## §9 — Network Security

**§9.1 Network Perimeter Security**
→ WAF deployed at all internet-facing entry points
  Status: ✅ IMPLEMENTED (Cloudflare + AWS WAF)
→ OWASP CRS rules enabled
  Status: 🔄 IN PROGRESS — deploying
→ DDoS protection
  Status: ✅ IMPLEMENTED (Cloudflare basic)

**§9.2 Network Segmentation**
→ Production environment isolated from development
  Status: 🔄 IN PROGRESS
→ Internal systems not directly accessible from internet
  Status: ✅ IMPLEMENTED (post-incident hardening)
→ /internal/ namespace blocked at perimeter
  Status: ✅ IMPLEMENTED (post-incident)

**§9.3 Network Monitoring**
→ Network traffic monitoring for anomalies
  Status: 📋 PLANNED — SIEM deployment needed
→ VPC Flow Logs enabled
  Status: 🔄 IN PROGRESS

**§9.4 Change Management for Network Config**
→ Formal CAB process for all network changes
  Status: 🔄 IN PROGRESS — CAB being established
→ This was root cause of Feb 2026 incident
  Status: ❌ WAS GAP → 🔄 IN PROGRESS fixing

---

## §10 — Identity and Access Management

**§10.1 Access Control Policy**
→ Least privilege principle enforced
  Status: 🔄 IN PROGRESS
→ Separation of duties for critical functions
  Status: 🔄 IN PROGRESS

**§10.2 User Authentication**
→ MFA for all privileged access
  Status: 🔄 IN PROGRESS — rolling out
→ MFA for remote access
  Status: 🔄 IN PROGRESS

**§10.3 Privileged Access**
→ PAM solution for privileged credentials
  Status: 📋 PLANNED — HashiCorp Vault evaluation
→ Session recording for privileged sessions
  Status: 📋 PLANNED
→ Just-in-time privileged access
  Status: 📋 PLANNED

**§10.4 Access Review**
→ Quarterly access rights review
  Status: 📋 PLANNED — process to establish
→ Prompt revocation on role change / termination
  Status: 🔄 IN PROGRESS — SOP being documented

---

## §11 — IT Resilience

**§11.1 Business Continuity Planning**
→ BCP covering all critical business processes
  Status: 🔄 IN PROGRESS
→ Annual BCP review and update
  Status: 📋 PLANNED

**§11.2 Recovery Objectives**
→ Critical systems RTO ≤ 4 hours
  Status: 🔄 IN PROGRESS — pending DR test
→ Critical systems RPO ≤ 4 hours
  Status: 🔄 IN PROGRESS — backup config review needed
→ System availability ≥ 99.95%
  Status: 🔄 IN PROGRESS — SLA monitoring being set up

**§11.3 DR Testing**
→ Annual full DR drill
  Status: 📋 PLANNED — Q3 2026

---

## §12 — Security Monitoring

**§12.1 Security Event Monitoring**
→ SIEM collecting logs from all critical systems
  Status: 📋 PLANNED — highest priority deployment
→ 24×7 security monitoring
  Status: 📋 PLANNED — SOC to be established / outsourced
→ Automated alerting for security events
  Status: 🔄 IN PROGRESS — initial rules being configured

**§12.2 Threat Intelligence**
→ Threat intelligence feeds integrated
  Status: 📋 PLANNED

---

## §13 — Data Security

**§13.1 Data Classification**
→ Formal data classification policy and scheme
  Status: 🔄 IN PROGRESS
→ Data inventory maintained
  Status: 📋 PLANNED

**§13.2 Data Protection**
→ Encryption of sensitive data at rest (AES-256)
  Status: 🔄 IN PROGRESS (RDS full-disk done, field-level in progress)
→ Encryption of data in transit (TLS 1.2+)
  Status: ✅ IMPLEMENTED (external), 🔄 IN PROGRESS (internal mTLS)
→ PAN tokenization (PCI DSS)
  Status: 🔄 IN PROGRESS
→ CVV never stored
  Status: ✅ IMPLEMENTED

**§13.3 DLP**
→ DLP solution covering endpoints
  Status: ✅ IMPLEMENTED (SealSuite)
→ DLP covering network and cloud
  Status: 📋 PLANNED (Microsoft Purview evaluation)

---

## §14 — Incident Management and Reporting

**§14.1 Incident Response Capability**
→ Documented Incident Response Plan
  Status: 🔄 IN PROGRESS — SOP being created
→ Incident response team defined
  Status: 🔄 IN PROGRESS
→ Regular IR drills
  Status: 📋 PLANNED

**§14.2 MAS Reporting**
→ Process to notify MAS within 1 hour of major incident
  Status: 🔄 IN PROGRESS — SOP being documented
→ Written report to MAS within 24 hours
  Status: 🔄 IN PROGRESS — template being created
→ This was a gap in Feb 2026 incident
  Status: ❌ WAS GAP → 🔄 IN PROGRESS fixing

---

## Gap Summary

**Critical Gaps (address immediately):**
→ CISO not yet appointed
→ SIEM not yet operational
→ MAS 1-hour notification SOP not documented
→ CAB process for network changes not yet formalized

**High Priority (address within 90 days):**
→ VAPT annual schedule not established
→ Quarterly access review process not established
→ PAM solution not deployed
→ Internal mTLS (Istio) not yet implemented
→ BCP/DR not tested

**Medium Priority (address within 6 months):**
→ Network-layer DLP
→ ISO 27001 certification initiation
→ Threat intelligence integration
→ Full data classification and inventory
