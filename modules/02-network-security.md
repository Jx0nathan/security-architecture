# Module 02: Network Security
**MAS TRM §9 — Network Security Controls**
Status: Partially Implemented

---

## 1. VPC Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  AWS VPC  (10.0.0.0/16)                                      │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  Public Subnet   │  │  Public Subnet   │  (Multi-AZ)     │
│  │  10.0.1.0/24     │  │  10.0.2.0/24     │                 │
│  │  ALB / NAT GW    │  │  ALB / NAT GW    │                 │
│  └────────┬─────────┘  └────────┬─────────┘                 │
│           │                     │                           │
│  ┌────────▼─────────────────────▼─────────┐                 │
│  │  Private Subnet  10.0.10.0/24           │                 │
│  │  K8s Worker Nodes / Microservices       │                 │
│  │  API Gateway / Traefik Ingress          │                 │
│  └────────────────────┬───────────────────┘                 │
│                       │                                     │
│  ┌────────────────────▼───────────────────┐                 │
│  │  Data Subnet  10.0.20.0/24             │                 │
│  │  RDS / ElastiCache / Kafka             │                 │
│  │  No direct internet access             │                 │
│  └────────────────────┬───────────────────┘                 │
│                       │                                     │
│  ┌────────────────────▼───────────────────┐                 │
│  │  Management Subnet  10.0.30.0/24       │                 │
│  │  SIEM / Vault / Bastion Host           │                 │
│  │  VPN termination / Jump Server         │                 │
│  └────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

**Rules:**
→ Public subnet: ALB only, no application servers
→ Private subnet: no direct internet access, outbound via NAT GW only
→ Data subnet: no internet access at all, only accepts connections from private subnet
→ Management subnet: accessible only via VPN or dedicated bastion host with MFA

---

## 2. Security Groups (Key Rules)

**ALB Security Group**
```
Inbound:  443 (HTTPS) from 0.0.0.0/0
          80  (HTTP)  from 0.0.0.0/0  → redirect to 443
Outbound: 8080 to Private Subnet SG only
```

**API Gateway / Traefik Security Group**
```
Inbound:  8080 from ALB SG only
          8443 from ALB SG only
Outbound: All to Private App SG
```

**Application Services Security Group**
```
Inbound:  App ports (8080-9090) from API GW SG only
          15000-15010 (Istio sidecar) from within Private Subnet
Outbound: 5432 (PostgreSQL) to Data SG
          6379 (Redis) to Data SG
          9092 (Kafka) to Data SG
```

**Data Subnet Security Group**
```
Inbound:  5432 from App SG only  (PostgreSQL)
          6379 from App SG only  (Redis)
          9092 from App SG only  (Kafka)
          NO inbound from 0.0.0.0/0
Outbound: Restricted, no internet
```

**Management Security Group**
```
Inbound:  22 (SSH) from VPN CIDR only  (10.0.31.0/24)
          443 from VPN CIDR only
          DENY all from 0.0.0.0/0
Outbound: All (monitoring needs broad outbound)
```

---

## 3. API Namespace Segregation [IMPLEMENTED - post-incident]

Critical post-incident fix: `/internal/` namespace MUST NOT be reachable from public internet.

**Traefik IngressRoute configuration:**
```yaml
# PUBLIC routes - exposed via CloudFront → ALB → API GW
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: public-routes
spec:
  entryPoints:
    - websecure
  routes:
    - match: PathPrefix(`/user/public/`) || PathPrefix(`/payment/public/`)
      kind: Rule
      services:
        - name: api-gateway-service
          port: 8080

---
# INTERNAL routes - only accessible from internal pods
# NOT exposed via external ingress
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: internal-routes
  namespace: internal
spec:
  entryPoints:
    - internal-only   # bound to 10.0.10.x only, NOT 0.0.0.0
  routes:
    - match: PathPrefix(`/internal/`)
      kind: Rule
      middlewares:
        - name: internal-auth   # requires X-Internal-Token header
      services:
        - name: internal-service
          port: 8081
```

---

## 4. Service-to-Service Communication: mTLS via Istio [PLANNED]

All internal microservice communication must use mutual TLS.

**Istio PeerAuthentication (enforce mTLS cluster-wide):**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT  # reject all non-mTLS traffic
```

**Istio AuthorizationPolicy (service-to-service allowlist):**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: card-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: card-service
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/subscription-service"
              - "cluster.local/ns/production/sa/kyc-service"
      to:
        - operation:
            methods: ["POST", "GET"]
            paths: ["/v1/cards/*"]
```

Benefits:
→ Services prove identity via certificate, not just IP
→ Compromised service cannot impersonate another service
→ All traffic encrypted in transit within cluster
→ Audit trail of inter-service calls

---

## 5. Network Routing Structure

```
Client Request Flow:
──────────────────
Internet
  ↓  HTTPS
Cloudflare WAF (DDoS filter, Bot block, OWASP rules)
  ↓  HTTPS (Cloudflare → AWS origin)
AWS CloudFront (CDN, Cache, TLS termination)
  ↓  HTTPS
AWS WAF (OWASP CRS, Rate limiting, IP reputation)
  ↓  HTTP (internal, SSL already terminated)
Application Load Balancer (health check, routing by hostname)
  ↓  HTTP/2
Traefik Ingress Controller (K8s, route by path prefix)
  ↓  gRPC / HTTP (mTLS via Istio sidecar)
Target Microservice
  ↓  TLS
Data Layer (RDS / Redis / Kafka)

Outbound Flow (microservice → external):
────────────────────────────────────────
Microservice → NAT Gateway → Internet
(For: Sumsub API, Blockchain RPC, SWIFT/payment rails)
All outbound via fixed NAT IP → whitelist at external providers
```

---

## 6. Office / Remote Access

**VPN (Zero Trust replacement target):**
→ Current: OpenVPN / WireGuard to Management Subnet
→ Target: SealSuite ZTNA replacing VPN
→ MFA required for all VPN/ZTNA connections
→ No split tunneling — all traffic routes through corporate gateway

**Bastion Host (for emergency direct access):**
→ Located in Management Subnet
→ No persistent SSH keys — session-based credentials via Vault
→ All sessions recorded (terminal recording to S3)
→ Auto-terminated after 8 hours of inactivity

---

## 7. MAS TRM §9 Compliance Checklist

→ Network perimeter controls (WAF, firewall): [IMPLEMENTED]
→ Internal network segmentation: [IN PROGRESS]
→ No unnecessary open ports: [IN PROGRESS - audit needed]
→ Encrypted data in transit: [IMPLEMENTED for external, IN PROGRESS for internal mTLS]
→ Monitoring of network traffic: [PLANNED - SIEM integration]
→ Change management for network config: [IN PROGRESS - CAB process being established]
→ Regular network vulnerability scans: [PLANNED - quarterly schedule]
