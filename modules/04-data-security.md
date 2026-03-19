# Module 04: Data Security
**MAS TRM §13 | PDPA | PCI DSS v4.0**
Status: Partially Implemented

---

## 1. Data Classification

**Tier 1 — Restricted (最高敏感)**
→ 卡号 (PAN)、CVV、PIN — PCI DSS scope
→ 身份证号、护照号
→ 银行账号、IBAN
→ 私钥、HSM seed
→ 处理方式：字段级加密 + 访问需审批 + 完整审计日志

**Tier 2 — Confidential (机密)**
→ 姓名 + 出生日期 + 地址组合 (PII联合)
→ 手机号、邮箱
→ KYC 文件（护照照片、身份证照片）
→ 交易记录、VA余额
→ 处理方式：传输加密 + 存储加密 + 访问角色控制

**Tier 3 — Internal (内部)**
→ 用户 ID、账户 ID（非敏感标识符）
→ 脱敏后的交易摘要
→ 系统日志（不含 PII）
→ 处理方式：传输加密，存储可不加密

**Tier 4 — Public (公开)**
→ 产品介绍、公开 API 文档
→ 汇率信息、费率表

---

## 2. 数据库加密规范

### 2.1 RDS PostgreSQL — 必须加密的字段

**存储层加密 (Encryption at Rest)**
→ RDS 开启 AWS KMS AES-256 全盘加密 [IMPLEMENTED]
→ 所有快照也自动加密

**字段级加密 (Field-Level Encryption) — 必须项**

```sql
-- 以下字段必须在应用层加密后再写入数据库
-- 使用 AES-256-GCM，密钥由 AWS KMS 管理

-- users 表
id_number         TEXT  -- 加密存储：身份证/护照号
date_of_birth     TEXT  -- 加密存储
phone_number      TEXT  -- 加密存储
email             TEXT  -- 加密存储（搜索需用 HMAC hash索引）
full_address      TEXT  -- 加密存储

-- cards 表
pan_encrypted     TEXT  -- 加密存储，不可明文
pan_last4         CHAR(4)  -- 允许明文（用于显示）
pan_token         TEXT     -- 支付 token（替代明文 PAN）
-- CVV: 永不存储，任何表都不允许存 CVV 明文或加密值

-- kyc_verifications 表
sumsub_applicant_id  TEXT  -- 加密存储
raw_response         TEXT  -- 加密存储（含完整 KYC 结果）
```

**字段加密实现（Java 示例）：**
```java
// 使用 AWS KMS Data Key 做信封加密
public class FieldEncryptionService {

    private final AWSKMS kmsClient;
    private final String keyId; // from config, never hardcoded

    public String encrypt(String plaintext) {
        // 1. 向 KMS 请求 Data Key
        GenerateDataKeyResult dataKey = kmsClient.generateDataKey(
            new GenerateDataKeyRequest()
                .withKeyId(keyId)
                .withKeySpec(DataKeySpec.AES_256)
        );
        // 2. 用 Data Key 做 AES-256-GCM 加密
        // 3. 存储：base64(encrypted_data_key) + "." + base64(ciphertext)
        // 4. 明文 Data Key 立即从内存清除
    }

    public String decrypt(String ciphertext) {
        // 1. 拆分 encrypted_data_key 和 ciphertext
        // 2. 调用 KMS 解密 Data Key
        // 3. 用 Data Key 解密 ciphertext
        // 4. 明文 Data Key 立即从内存清除
    }
}
```

**PAN Tokenization (PCI DSS §3):**
```
真实 PAN: 4111 1111 1111 1111
Token:    7823 4521 9876 0001  (不可逆 token，用于系统内流转)
显示:     **** **** **** 1111  (用户界面只显示后4位)

Token Vault: 独立的 tokenization 服务 / Basis Theory / VGS
不允许在主数据库存储明文 PAN
```

---

### 2.2 Redis / ElastiCache

**强制规则：**
→ 开启 in-transit TLS 1.2+ [IMPLEMENTED]
→ 开启 at-rest encryption (ElastiCache)
→ Redis 中不允许存储完整 PAN、CVV、ID 号
→ Session token 存储 TTL 最大 24 小时
→ Cache key 不包含可识别的用户 ID（使用 hash）

```
❌ 错误：SET user:12345:profile {"pan":"4111...","cvv":"123"}
✅ 正确：SET session:sha256(token):{} TTL=3600
```

---

### 2.3 S3 — KYC 文件存储

**配置要求：**
```
Bucket Policy:
  → Block all public access: ENABLED
  → Server-Side Encryption: SSE-KMS (AES-256)
  → Versioning: ENABLED
  → MFA Delete: ENABLED (防止意外删除)
  → Access Logging: ENABLED → 写入独立 audit bucket

访问控制:
  → 只允许 KYC 服务的 IAM Role 读写
  → 用户前端不直接访问 S3
  → 生成 Presigned URL，有效期最长 15 分钟
  → URL 使用后立即失效（通过 CloudFront + Lambda@Edge 实现单次使用）

保留策略:
  → KYC 文件保留 5 年（MAS/PDPA 要求）
  → 5 年后自动归档到 Glacier → 7 年后删除
```

---

## 3. 传输加密规范

### 外部 API (Client ↔ Server)
```
协议:   TLS 1.2 最低，推荐 TLS 1.3
禁用:   TLS 1.0, TLS 1.1, SSL 3.0
加密套件 (允许列表):
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256
  TLS_AES_128_GCM_SHA256
HSTS: max-age=63072000; includeSubDomains; preload
证书: AWS ACM 管理，自动续期
```

### 内部服务间 (Service ↔ Service)
```
方案: Istio mTLS (STRICT mode)
证书: Istio CA 自动签发，90天轮换
验证: 双向证书验证，无法伪造服务身份
```

### 数据库连接
```
PostgreSQL: sslmode=verify-full (客户端必须验证服务器证书)
Redis: TLS=true, cert-reqs=required
Kafka: security.protocol=SSL + SASL_SSL
```

---

## 4. 数据保留与销毁 (PDPA Compliance)

```
KYC 文档:     保留 5 年（账户注销后）→ 然后安全删除
交易记录:     保留 5 年（MAS PSA 要求）
应用日志:     保留 1 年（含 PII 的日志脱敏后保留）
安全审计日志: 保留 3 年（MAS TRM 要求）
STR 报告:     保留 5 年（MAS Notice PSN01）
```

**数据销毁要求：**
→ S3 数据：AWS 安全删除（覆盖写入）
→ RDS 数据：应用层先覆盖字段值，再 DELETE
→ 硬件销毁：需要 DoD 5220.22-M 标准证书

---

## 5. DLP 覆盖 (SealSuite + 计划中)

**当前状态 [SealSuite 端点 DLP]:**
→ USB 复制管控 ✅
→ 截图限制（敏感界面） ✅
→ 应用外传监控 ✅
→ 设备合规检查 ✅

**待补充 [IN PROGRESS]:**
→ 邮件 DLP：检测 PII 通过邮件外发 → Microsoft Purview
→ 云存储 DLP：阻止 PII 上传至个人 Google Drive / Dropbox
→ 数据库批量导出监控：超过阈值的 SELECT 触发告警

---

## 6. 密钥管理架构

```
Layer 1: AWS KMS (Master Key)
  → 从不离开 AWS HSM
  → 用于加密 Data Keys

Layer 2: AWS KMS Data Keys
  → 每个加密操作生成独立 Data Key
  → 明文 Data Key 使用后立即销毁

Layer 3: HashiCorp Vault
  → 存储应用配置密钥、API Keys、DB 密码
  → 动态 credential（数据库密码每次请求都不同）
  → 审计所有密钥访问

Layer 4: 区块链私钥
  → 存储在 Fireblocks HSM
  → MPC (Multi-Party Computation) 签名
  → 无单点持有
```

---

## 7. PDPA 合规要求对照

→ 数据最小化：只收集业务必要的数据 [IN PROGRESS]
→ 同意管理：用户明确同意数据使用目的 [IMPLEMENTED]
→ 数据访问请求：30天内响应 [SOP 待建立]
→ 数据泄露通知：3个工作日内通知 PDPC [SOP 待建立]
→ 数据出境：确认数据不出新加坡，或满足豁免条件 [待审查]
→ DPO 任命：已要求，待正式任命 [IN PROGRESS]
