# 06 — Cloud Armor & Web Application Firewall (WAF)

> **Domain 2 | Weight: ~5% of total exam**  
> **Time:** ~3 hours concept + 2 hours hands-on

---

## Concepts

### 1. Cloud Armor Overview

**Cloud Armor** is GCP's DDoS protection and WAF service that integrates with the **Global HTTPS Load Balancer**.

```
Internet
   ↓
Cloud Armor (edge security policy evaluated here)
   ↓
Global HTTPS Load Balancer
   ↓
Backend Services (GCE, GKE, Cloud Run, Cloud Storage)
```

Cloud Armor operates at the **edge** — rules are evaluated before traffic reaches your backends, globally distributed.

---

### 2. Security Policy Types

| Type | Scope | Attach To |
|------|-------|-----------|
| **Backend security policy** | Per backend service | Backend services on global LB |
| **Edge security policy** | Cache level | Cloud CDN backends |
| **Network edge security policy** | External passthrough LB | Network LB backends |

---

### 3. Security Policy Rules

Each rule has:
- **Priority:** 0–2147483646 (lower = higher priority)
- **Match condition:** IP ranges, HTTP headers, request properties, CEL expressions
- **Action:** `allow`, `deny(403)`, `deny(404)`, `deny(502)`, `redirect`, `throttle`, `rate_based_ban`

**Default rule (priority 2147483647):** Either allow or deny — must always exist.

---

### 4. Match Conditions

| Condition Type | Example |
|---------------|---------|
| IP/CIDR ranges | `inIpRange(origin.ip, '203.0.113.0/24')` |
| HTTP headers | `request.headers['x-forwarded-for'].contains('10.0.0.1')` |
| Request path | `request.path.startsWith('/admin')` |
| User-Agent | `request.headers['user-agent'].contains('Googlebot')` |
| Named IP lists | `evaluatePreconfiguredExpr('sourceiplist-fastly')` |
| Geo-based | `origin.region_code == 'CN'` |
| WAF rules | `evaluatePreconfiguredExpr('xss-stable')` |

---

### 5. Preconfigured WAF Rules (OWASP Top 10)

Google maintains managed rule sets you can reference:

| Expression | Protects Against |
|-----------|-----------------|
| `sqli-stable` | SQL Injection |
| `xss-stable` | Cross-Site Scripting |
| `lfi-stable` | Local File Inclusion |
| `rfi-stable` | Remote File Inclusion |
| `rce-stable` | Remote Code Execution |
| `methodenforcement-stable` | Abnormal HTTP methods |
| `scannerdetection-stable` | Security scanner fingerprints |
| `protocolattack-stable` | HTTP protocol violations |
| `php-stable` | PHP injection attacks |
| `sessionfixation-stable` | Session fixation attacks |
| `java-stable` | Java attack signatures |
| `nodejs-stable` | Node.js attack signatures |
| `cve-canary` | CVE-specific rules |

**Sensitivity levels:** Each rule has sensitivity 0–4. Higher = more sensitive = more false positives.

---

### 6. Adaptive Protection

ML-based DDoS detection built into Cloud Armor.

- Learns normal traffic patterns automatically
- Detects anomalous traffic spikes
- **Auto-generates suggested rules** when an attack is detected
- You can apply suggested rules immediately or set them to preview

**How to enable:**
```bash
--enable-layer7-ddos-defense
```

---

### 7. Rate Limiting Actions

| Action | Behavior |
|--------|----------|
| `throttle` | Limit requests per client per interval — excess returns 429 |
| `rate_based_ban` | Ban IP temporarily when threshold exceeded |

**Throttle threshold example:**
- 100 requests per 60 seconds per IP
- Excess requests get a `429 Too Many Requests` response

---

### 8. Bot Management (reCAPTCHA Enterprise Integration)

Cloud Armor can validate **reCAPTCHA tokens** embedded in requests:
- `allow` requests with valid tokens
- `deny` or `redirect` requests without valid tokens
- Redirect suspicious requests to a reCAPTCHA challenge page

---

### 9. Preview Mode

Rules can be set to **preview** — they log hits but don't enforce. Use to:
- Test new rules before blocking
- Validate WAF rules don't block legitimate traffic
- Assess Adaptive Protection suggestions

Logs appear in `jsonPayload.enforcedSecurityPolicy.outcome == "PREVIEW"`.

---

### 10. Named IP Lists

Google-curated IP lists you can reference in rules:

| List | Contents |
|------|---------|
| `sourceiplist-fastly` | Fastly CDN IPs |
| `sourceiplist-cloudflare` | Cloudflare IPs |
| `sourceiplist-imperva` | Imperva CDN IPs |
| `sourceiplist-tor` | Tor exit nodes |
| `sourceiplist-aws-amazonaws` | AWS IP ranges |
| `sourceiplist-azure-cloud` | Azure IP ranges |

---

## gcloud Commands

### Create and Manage Security Policies
```bash
# Create a basic security policy
gcloud compute security-policies create my-security-policy \
  --description="Web application security policy"

# Create with Adaptive Protection enabled
gcloud compute security-policies create my-adaptive-policy \
  --description="With DDoS protection" \
  --enable-layer7-ddos-defense \
  --layer7-ddos-defense-rule-visibility=STANDARD

# List security policies
gcloud compute security-policies list

# Describe a policy (see all rules)
gcloud compute security-policies describe my-security-policy

# Delete a policy
gcloud compute security-policies delete my-security-policy
```

### Adding Rules
```bash
# Allow only specific IP range (e.g., office network)
gcloud compute security-policies rules create 100 \
  --security-policy=my-security-policy \
  --description="Allow office IPs" \
  --src-ip-ranges=203.0.113.0/24 \
  --action=allow

# Block a specific IP
gcloud compute security-policies rules create 200 \
  --security-policy=my-security-policy \
  --description="Block bad actor IP" \
  --src-ip-ranges=198.51.100.1/32 \
  --action=deny-403

# Block a country (geo-based) using expression
gcloud compute security-policies rules create 300 \
  --security-policy=my-security-policy \
  --description="Block specific region" \
  --expression="origin.region_code == 'CN'" \
  --action=deny-403

# Block Tor exit nodes (named IP list)
gcloud compute security-policies rules create 400 \
  --security-policy=my-security-policy \
  --description="Block Tor nodes" \
  --expression="evaluatePreconfiguredExpr('sourceiplist-tor')" \
  --action=deny-403

# Add WAF rule — block XSS (sensitivity 1)
gcloud compute security-policies rules create 1000 \
  --security-policy=my-security-policy \
  --description="Block XSS attacks" \
  --expression="evaluatePreconfiguredExpr('xss-stable', {'sensitivity': 1})" \
  --action=deny-403

# Add WAF rule — block SQL injection
gcloud compute security-policies rules create 1001 \
  --security-policy=my-security-policy \
  --description="Block SQL injection" \
  --expression="evaluatePreconfiguredExpr('sqli-stable', {'sensitivity': 2})" \
  --action=deny-403

# Add WAF rule in PREVIEW mode (log only, don't block)
gcloud compute security-policies rules create 1002 \
  --security-policy=my-security-policy \
  --description="Preview RCE rule" \
  --expression="evaluatePreconfiguredExpr('rce-stable')" \
  --action=deny-403 \
  --preview

# Default rule — allow all (change to deny for allowlist-only setup)
gcloud compute security-policies rules update 2147483647 \
  --security-policy=my-security-policy \
  --action=allow

# Default deny (only allow explicitly whitelisted IPs)
gcloud compute security-policies rules update 2147483647 \
  --security-policy=my-security-policy \
  --action=deny-403
```

### Rate Limiting
```bash
# Add throttle rule (100 req/min per IP)
gcloud compute security-policies rules create 500 \
  --security-policy=my-security-policy \
  --description="Rate limit per IP" \
  --src-ip-ranges=0.0.0.0/0 \
  --action=throttle \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP

# Add rate-based ban (ban after 1000 req/min)
gcloud compute security-policies rules create 501 \
  --security-policy=my-security-policy \
  --description="Ban high-rate IPs" \
  --src-ip-ranges=0.0.0.0/0 \
  --action=rate_based_ban \
  --rate-limit-threshold-count=1000 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=3600 \
  --conform-action=allow \
  --exceed-action=deny-403 \
  --enforce-on-key=IP

# Rate limit on a specific path (API endpoint)
gcloud compute security-policies rules create 600 \
  --security-policy=my-security-policy \
  --description="Rate limit /api/login" \
  --expression="request.path == '/api/login'" \
  --action=throttle \
  --rate-limit-threshold-count=10 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP
```

### Attach Policy to Backend Service
```bash
# Attach security policy to a backend service
gcloud compute backend-services update my-backend \
  --security-policy=my-security-policy \
  --global

# Verify the attachment
gcloud compute backend-services describe my-backend \
  --global \
  --format="get(securityPolicy)"

# Remove security policy from backend service
gcloud compute backend-services update my-backend \
  --no-security-policy \
  --global
```

### Edge Security Policy (for Cloud CDN)
```bash
# Create an edge security policy
gcloud compute security-policies create my-edge-policy \
  --type=CLOUD_ARMOR_EDGE

# Add a rule (only IP-based for edge policies)
gcloud compute security-policies rules create 100 \
  --security-policy=my-edge-policy \
  --src-ip-ranges=203.0.113.0/24 \
  --action=allow

# Attach to CDN backend
gcloud compute backend-buckets update my-backend-bucket \
  --edge-security-policy=my-edge-policy
```

### Monitoring and Logs
```bash
# View security policy request logs
gcloud logging read \
  'resource.type="http_load_balancer" AND jsonPayload.enforcedSecurityPolicy.name="my-security-policy"' \
  --format="table(timestamp,jsonPayload.enforcedSecurityPolicy.outcome,jsonPayload.statusDetails,httpRequest.remoteIp)"

# View blocked requests
gcloud logging read \
  'resource.type="http_load_balancer" AND jsonPayload.enforcedSecurityPolicy.outcome="DENY"' \
  --limit=50 \
  --format="table(timestamp,httpRequest.remoteIp,httpRequest.requestUrl,jsonPayload.enforcedSecurityPolicy.priority)"

# View preview rule hits
gcloud logging read \
  'resource.type="http_load_balancer" AND jsonPayload.previewSecurityPolicy.outcome="DENY"' \
  --limit=20
```

---

## Hands-On Practice

### Exercise 1: Build a WAF for a Web App

```bash
POLICY="webapp-waf"
BACKEND="my-web-backend"

# Create policy
gcloud compute security-policies create $POLICY \
  --description="Web app WAF policy" \
  --enable-layer7-ddos-defense

# Rule 1: Allow Google health checker IPs
gcloud compute security-policies rules create 100 \
  --security-policy=$POLICY \
  --src-ip-ranges=35.191.0.0/16,130.211.0.0/22 \
  --action=allow \
  --description="LB health checkers"

# Rule 2: Block Tor
gcloud compute security-policies rules create 200 \
  --security-policy=$POLICY \
  --expression="evaluatePreconfiguredExpr('sourceiplist-tor')" \
  --action=deny-403 \
  --description="Block Tor"

# Rule 3: WAF — XSS (preview first)
gcloud compute security-policies rules create 1000 \
  --security-policy=$POLICY \
  --expression="evaluatePreconfiguredExpr('xss-stable')" \
  --action=deny-403 \
  --preview \
  --description="XSS protection (preview)"

# Rule 4: WAF — SQLi (preview first)
gcloud compute security-policies rules create 1001 \
  --security-policy=$POLICY \
  --expression="evaluatePreconfiguredExpr('sqli-stable')" \
  --action=deny-403 \
  --preview \
  --description="SQL injection protection (preview)"

# Rule 5: Rate limit login endpoint
gcloud compute security-policies rules create 2000 \
  --security-policy=$POLICY \
  --expression="request.path.matches('/login')" \
  --action=throttle \
  --rate-limit-threshold-count=5 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP \
  --description="Brute-force protection for /login"

# Default: allow
gcloud compute security-policies rules update 2147483647 \
  --security-policy=$POLICY \
  --action=allow

# Attach to backend
gcloud compute backend-services update $BACKEND \
  --global \
  --security-policy=$POLICY

# Describe full policy
gcloud compute security-policies describe $POLICY
```

### Exercise 2: Enable Rules After Preview Validation

```bash
# After monitoring for 24-48 hours and confirming no false positives:

# Promote XSS rule from preview to enforce
gcloud compute security-policies rules update 1000 \
  --security-policy=$POLICY \
  --no-preview

# Promote SQLi rule from preview to enforce
gcloud compute security-policies rules update 1001 \
  --security-policy=$POLICY \
  --no-preview
```

---

## Review Questions

1. Cloud Armor is attached to your HTTPS load balancer. An attacker launches a Layer 3 volumetric DDoS attack. Does Cloud Armor protect you? What does it protect against?

2. You want to block all traffic from a country except for your known IP range (corporate VPN). Write the rule priority order and expressions.

3. What is **Adaptive Protection** and how does it generate rules? What must you do with the suggested rules?

4. What is the difference between `throttle` and `rate_based_ban` actions?

5. Your WAF rule for SQLi is causing too many false positives for your legacy app. Without disabling the rule, what can you do?

---

## Key Exam Points

- **Cloud Armor is ONLY for Global HTTP(S) LB and External Passthrough NLB** — does NOT protect internal LBs or direct VM access
- **Lower priority number = evaluated first** — same as firewall rules
- **Default rule is always `2147483647`** — must exist, can be allow or deny
- **Preview mode logs but does NOT block** — use for testing WAF rules
- **Adaptive Protection** is automatic but does NOT auto-apply rules — you must review and apply
- **Named IP lists** (like `sourceiplist-tor`) are maintained by Google — you don't manage the IPs
- **Edge security policy** = Cloud CDN edge enforcement — rule types are more limited (IP-only)
- **Throttle** = slow down, **rate_based_ban** = temporary ban after threshold
