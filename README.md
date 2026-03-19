# DCS Card Centre — Security Architecture
**MAS TRM 2021 Compliant | PDPA | PCI DSS v4.0**

> ⚠️ CONFIDENTIAL — Internal use only. Do not distribute externally.

---

## Repository Structure

```
architecture/
  overall-architecture.excalidraw   ← 整体安全架构图
  network-topology.excalidraw       ← 网络拓扑图（待补充）
  data-flow.excalidraw              ← 数据流图（待补充）

modules/
  01-perimeter-security.md          ← 边界安全（WAF、DDoS、CDN）
  02-network-security.md            ← 内网安全（mTLS、VPC、路由）
  03-application-security.md        ← 应用安全（API、SDL、认证）
  04-data-security.md               ← 数据安全（加密、DLP、分类）
  05-iam.md                         ← 身份与访问管理（IAM、PAM、MFA）
  06-security-operations.md         ← 安全运营（SIEM、SOC、IR）
  07-compliance-grc.md              ← 合规治理（MAS TRM、PDPA、PCI DSS）

mas-mapping/
  trm-control-mapping.md            ← MAS TRM 逐条控制映射
  gap-analysis.md                   ← 合规缺口分析
```

---

## 整体架构分层

```
Internet
    ↓
[Edge Security]     Cloudflare WAF + DDoS + Bot Protection
    ↓
[AWS Ingress]       CloudFront + AWS WAF (OWASP CRS) + ALB
    ↓
[API Gateway]       Kong / AWS API GW — Auth + Rate Limit + Routing
    ↓
[Application]       K8s + Traefik + Istio mTLS + Vault
    ↓
[Data Layer]        RDS(AES-256) + Redis(TLS) + S3(SSE-KMS) + Blockchain(HSM)
    ↓
[Sec Operations]    SIEM + EDR/SealSuite + CSPM + WAF Analytics
    ↓
[Identity/GRC]      IAM + PAM + GRC + CISO
```

---

## MAS TRM 核心合规要求

| TRM 章节 | 要求 | 当前状态 |
|---------|------|---------|
| §6 Governance | CISO 独立汇报 CEO | 待建立 |
| §8 System Security | VAPT 每年一次 | 计划中 |
| §9 Network Security | WAF + 网络隔离 | 部分完成 |
| §10 IAM | MFA + 最小权限 | 部分完成 |
| §11 Cyber Resilience | RTO ≤ 4h, RPO ≤ 4h | 待验证 |
| §12 Monitoring | SIEM 7×24 | 待建立 |
| §13 Data Security | 加密 + 分类 | 部分完成 |
| §14 Incident Management | 1小时通知 MAS | SOP 待完善 |

---

## 版本历史

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-03-15 | v0.1 | 初始架构草案 |
