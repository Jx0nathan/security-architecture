# DCS Card Centre — 安全架构
**符合 MAS TRM 2021 | PDPA | PCI DSS v4.0**

> ⚠️ 机密文件 — 仅限内部使用，禁止对外传播。

---

## 仓库结构

```
architecture/
  overall-architecture.excalidraw   ← 整体安全架构图（用 excalidraw.com 打开）

安全技术实施指南/
  01-边界安全.md                     ← WAF、DDoS、TLS 配置规范
  02-内网安全.md                     ← VPC 结构、mTLS、路由、命名空间隔离
  03-数据安全.md                     ← 字段加密、PAN Tokenization、DLP
  04-身份与访问管理.md                ← MFA、PAM、Vault、访问审查
  05-安全运营.md                     ← SIEM 告警规则、SOC 流程、IR SOP、VAPT

mas-mapping/
  trm-control-mapping.md            ← MAS TRM §6-§14 逐条控制映射与缺口分析
```

---

## 整体架构分层（纵深防御）

```
互联网
    ↓
【边界安全】     Cloudflare WAF + DDoS + Bot 过滤
    ↓
【AWS 入口】     CloudFront + AWS WAF（OWASP CRS）+ ALB
    ↓
【API 网关】     Kong / AWS API GW — 认证 + 限速 + 路由
    ↓
【应用层】       K8s + Traefik + Istio mTLS + Vault
    ↓
【数据层】       RDS（AES-256）+ Redis（TLS）+ S3（SSE-KMS）+ 区块链（HSM）
    ↓
【安全运营】     SIEM + EDR/SealSuite + CSPM + WAF 分析
    ↓
【身份与合规】   IAM + PAM + GRC + CISO
```

---

## MAS TRM 核心合规要求

**§6 治理与监督**
→ CISO 独立向 CEO/董事会汇报
→ IT 风险管理框架正式建立并年度审查

**§8 系统安全**
→ 系统加固标准（CIS Benchmark）
→ VAPT 每年一次（Critical 漏洞 14 天内修复）
→ SDL 嵌入研发流程

**§9 网络安全**
→ WAF 边界防护 + OWASP CRS
→ 网络分区（公有 / 私有 / 数据 / 管理子网）
→ 内部接口禁止暴露公网
→ 所有网络配置变更走 CAB 审批

**§10 身份与访问管理**
→ MFA 强制（所有生产系统）
→ 最小权限原则
→ PAM 工具（HashiCorp Vault）
→ 季度访问权限审查

**§11 业务连续性**
→ 关键系统 RTO ≤ 4 小时，RPO ≤ 4 小时
→ 可用性 ≥ 99.95%
→ 每年一次完整 DR 演练

**§12 安全监控**
→ SIEM 7×24 告警
→ 异常行为实时检测
→ P1 告警 15 分钟内响应

**§13 数据安全**
→ AES-256 静态加密（含字段级加密）
→ TLS 1.2+ 传输加密
→ PAN Tokenization（PCI DSS 要求）
→ CVV 永不存储

**§14 事件管理**
→ 重大事件 1 小时内致电 MAS
→ 24 小时内提交书面报告
→ 关闭后 30 天内提交完整 RCA

---

## 版本历史

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-03-15 | v0.2 | 全部文档翻译为中文，目录重命名为安全技术实施指南 |
| 2026-03-15 | v0.1 | 初始架构草案（英文版） |
