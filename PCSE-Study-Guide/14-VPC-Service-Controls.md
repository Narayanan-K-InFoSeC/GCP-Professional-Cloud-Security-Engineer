# 14 — VPC Service Controls (VPC-SC)

> **Domain 3 | Weight: ~5% of total exam**  
> **Time:** ~4 hours concept + 2 hours hands-on

---

## Concepts

### 1. What is VPC Service Controls?

**VPC Service Controls** creates an **API-level security perimeter** around GCP resources.

**Problem it solves:** IAM controls WHO can access resources. VPC-SC controls FROM WHERE they can access them. It prevents data exfiltration even by authenticated users.

```
Without VPC-SC:
  Employee with BigQuery access → copies data to personal GCS bucket
  → Data leaves the org! (authenticated, but unauthorized location)

With VPC-SC:
  Employee with BigQuery access → tries to copy to personal GCS bucket
  → BLOCKED! (destination GCS bucket is outside the perimeter)
```

---

### 2. Service Perimeters

A **service perimeter** is a logical boundary that:
- Protects a set of GCP **services** (BigQuery, GCS, Spanner, KMS, etc.)
- Around a set of GCP **projects/VPC networks**
- Enforces that requests to those services come from within the perimeter

**Two perimeter types:**
| Type | Description |
|------|-------------|
| **Regular perimeter** | Enforces access restrictions |
| **Bridge perimeter** | Allows two regular perimeters to share specific resources |

---

### 3. Perimeter Components

```
Service Perimeter
├── Restricted services: [BigQuery, Storage, KMS, Spanner...]
├── Projects: [project-A, project-B]
├── VPC Networks: [my-vpc]
├── Access levels: (conditions for ingress from outside)
├── Ingress rules: (allow specific external principals/sources in)
└── Egress rules: (allow specific internal resources/destinations out)
```

---

### 4. Ingress and Egress Rules

**Ingress rules** define what external traffic can enter the perimeter:
- `from`: external identity (user, SA, access level)
- `to`: protected resources inside the perimeter

**Egress rules** define what internal traffic can leave the perimeter:
- `from`: source inside the perimeter (project, SA)
- `to`: external destination (project, service)

**Without ingress/egress rules:** Only requests that originate AND terminate inside the perimeter are allowed.

---

### 5. Access Levels in VPC-SC

Access levels from Access Context Manager can be referenced in VPC-SC ingress rules to allow external users that meet context requirements (corporate network, managed device) to enter the perimeter.

---

### 6. Dry-Run Mode (Preview)

**Dry-run mode** evaluates the perimeter but does NOT enforce it:
- Access is allowed even if it would be blocked in enforced mode
- Violations are logged with `dryRunViolation: true`
- Use dry-run before enforcing to avoid breaking production

**Always start with dry-run and monitor logs for at least 24-48 hours before enforcing.**

---

### 7. VPC Accessible Services

Within a perimeter, you can restrict which APIs a VM on the VPC can call:
- By default, VMs in the perimeter can call any Google API
- With `vpcAccessibleServices`, restrict to only listed APIs

---

### 8. Restricted vs Private googleapis.com

For VMs to access protected APIs while inside a VPC-SC perimeter:
- Use `restricted.googleapis.com` (199.36.153.4/30) instead of `private.googleapis.com`
- This ensures API requests are routed through VPC-SC enforcement
- Configure DNS to resolve `*.googleapis.com` → `restricted.googleapis.com`

---

## gcloud Commands

### Access Policy Setup
```bash
# Create an access policy (one per org)
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="My Corp Policy"

# Get policy name
POLICY=$(gcloud access-context-manager policies list \
  --organization=ORG_ID \
  --format="value(name)" | head -1)
echo "Policy: $POLICY"

# Create an access level (for ingress rules)
cat > corp-access-level.yaml << 'EOF'
conditions:
  - ipSubnetworks:
      - 203.0.113.0/24
    members:
      - user:admin@example.com
combiningFunction: OR
EOF

gcloud access-context-manager levels create corp-network \
  --policy=$POLICY \
  --title="Corporate Network" \
  --basic-level-spec=corp-access-level.yaml
```

### Creating a Service Perimeter
```bash
# Create a perimeter in DRY-RUN mode first (always!)
gcloud access-context-manager perimeters dry-run create my-perimeter \
  --policy=$POLICY \
  --title="Data Protection Perimeter" \
  --type=REGULAR \
  --resources=projects/PROJECT_NUMBER_A,projects/PROJECT_NUMBER_B \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com,spanner.googleapis.com,cloudkms.googleapis.com

# Describe the dry-run perimeter
gcloud access-context-manager perimeters describe my-perimeter \
  --policy=$POLICY

# List perimeters
gcloud access-context-manager perimeters list --policy=$POLICY

# View dry-run violations in Cloud Logging
gcloud logging read \
  'resource.type="audited_resource" AND 
   protoPayload.status.code=403 AND 
   protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"' \
  --format="json" \
  --limit=10

# After reviewing dry-run violations, enforce the perimeter
gcloud access-context-manager perimeters dry-run enforce my-perimeter \
  --policy=$POLICY
```

### Enforced Perimeter Setup (Direct)
```bash
# Create an enforced perimeter directly (only if you know what you're doing)
gcloud access-context-manager perimeters create production-perimeter \
  --policy=$POLICY \
  --title="Production Data Perimeter" \
  --type=REGULAR \
  --resources=projects/PROD_PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com,bigquery.googleapis.com

# Update perimeter to add more projects
gcloud access-context-manager perimeters update production-perimeter \
  --policy=$POLICY \
  --add-resources=projects/NEW_PROJECT_NUMBER

# Add more restricted services
gcloud access-context-manager perimeters update production-perimeter \
  --policy=$POLICY \
  --add-restricted-services=spanner.googleapis.com,cloudkms.googleapis.com
```

### Ingress and Egress Rules
```bash
# Add ingress rule: allow users from corporate network to access BigQuery
# Create ingress policy file
cat > ingress-policy.yaml << 'EOF'
- ingressFrom:
    identityType: USER_SERVICE_ACCOUNT
    identities:
      - user:analyst@example.com
      - group:data-team@example.com
    sources:
      - accessLevel: accessPolicies/POLICY_ID/accessLevels/corp-network
  ingressTo:
    operations:
      - serviceName: bigquery.googleapis.com
        methodSelectors:
          - method: "*"
    resources:
      - projects/PROTECTED_PROJECT_NUMBER
EOF

gcloud access-context-manager perimeters update production-perimeter \
  --policy=$POLICY \
  --set-ingress-policies=ingress-policy.yaml

# Add egress rule: allow exporting BigQuery data to a specific external project
cat > egress-policy.yaml << 'EOF'
- egressFrom:
    identityType: USER_SERVICE_ACCOUNT
    identities:
      - serviceAccount:bq-export-sa@PROJECT_ID.iam.gserviceaccount.com
  egressTo:
    operations:
      - serviceName: storage.googleapis.com
        methodSelectors:
          - method: "*"
    resources:
      - projects/EXPORT_DESTINATION_PROJECT_NUMBER
EOF

gcloud access-context-manager perimeters update production-perimeter \
  --policy=$POLICY \
  --set-egress-policies=egress-policy.yaml

# View current ingress/egress rules
gcloud access-context-manager perimeters describe production-perimeter \
  --policy=$POLICY \
  --format="yaml(status.ingressPolicies,status.egressPolicies)"
```

### VPC Accessible Services
```bash
# Restrict which APIs VMs in the VPC can call (within perimeter)
gcloud access-context-manager perimeters update production-perimeter \
  --policy=$POLICY \
  --enable-vpc-accessible-services \
  --add-vpc-allowed-services=RESTRICTED-SERVICES

# This restricts VMs inside the perimeter to only use
# the explicitly listed services via VPC
```

### Perimeter Bridge
```bash
# Create a bridge between two perimeters (allow them to share data)
gcloud access-context-manager perimeters create bridge-perimeter \
  --policy=$POLICY \
  --title="Bridge between prod and analytics" \
  --type=BRIDGE \
  --resources=projects/PROD_PROJECT_NUMBER,projects/ANALYTICS_PROJECT_NUMBER
```

### Deleting Perimeters
```bash
# Delete a perimeter (data becomes accessible again!)
gcloud access-context-manager perimeters delete my-perimeter \
  --policy=$POLICY

# Delete dry-run only (keeps enforced perimeter)
gcloud access-context-manager perimeters dry-run delete my-perimeter \
  --policy=$POLICY
```

### Monitoring VPC-SC Violations
```bash
# All VPC-SC denials
gcloud logging read \
  'protoPayload.status.code=403 AND 
   protoPayload.metadata."@type":"VpcServiceControlAuditMetadata"' \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName,protoPayload.status.message)"

# Dry-run violations only
gcloud logging read \
  'protoPayload.metadata.vpcServiceControlsUniqueId!="" AND 
   protoPayload.metadata.dryRun=true' \
  --limit=20

# Create a log-based metric for VPC-SC violations
gcloud logging metrics create vpsc-violations \
  --description="Count of VPC-SC access denied events" \
  --log-filter='protoPayload.status.code=403 AND protoPayload.metadata."@type":"VpcServiceControlAuditMetadata"'
```

### Configure DNS for `restricted.googleapis.com`
```bash
# Create a Cloud DNS response policy to route APIs to restricted endpoint
gcloud dns response-policies create restricted-apis-policy \
  --description="Route Google APIs to restricted.googleapis.com" \
  --networks=my-vpc

# Add rules for googleapis.com
gcloud dns response-policies rules create googleapis \
  --response-policy=restricted-apis-policy \
  --dns-name="*.googleapis.com." \
  --local-data=name="*.googleapis.com.",type=CNAME,ttl=300,rrdatas=restricted.googleapis.com.

# Add A records for restricted.googleapis.com
gcloud dns response-policies rules create restricted-a-record \
  --response-policy=restricted-apis-policy \
  --dns-name="restricted.googleapis.com." \
  --local-data=name="restricted.googleapis.com.",type=A,ttl=300,rrdatas="199.36.153.4;199.36.153.5;199.36.153.6;199.36.153.7"
```

---

## Hands-On Practice

### Exercise 1: Full VPC-SC Setup — Dry Run to Enforce

```bash
ORG_ID="your-org-id"
PROJECT_NUMBER=$(gcloud projects describe PROJECT_ID --format="value(projectNumber)")

# Step 1: Get or create access policy
POLICY=$(gcloud access-context-manager policies list \
  --organization=$ORG_ID --format="value(name)" | head -1)
[ -z "$POLICY" ] && POLICY=$(gcloud access-context-manager policies create \
  --organization=$ORG_ID --title="Corp Policy" --format="value(name)")
echo "Policy: $POLICY"

# Step 2: Create access level for your current IP
MY_IP=$(curl -s https://api.ipify.org)
cat > my-level.yaml << EOF
conditions:
  - ipSubnetworks:
      - ${MY_IP}/32
EOF
gcloud access-context-manager levels create my-corp-access \
  --policy=$POLICY \
  --title="My Corp Access" \
  --basic-level-spec=my-level.yaml

# Step 3: Create perimeter in DRY-RUN mode
gcloud access-context-manager perimeters dry-run create test-perimeter \
  --policy=$POLICY \
  --title="Test Perimeter" \
  --type=REGULAR \
  --resources="projects/$PROJECT_NUMBER" \
  --restricted-services="storage.googleapis.com,bigquery.googleapis.com"

# Step 4: Monitor violations for 24 hours before enforcing
echo "Perimeter in dry-run. Monitor logs before enforcing."
echo "Check violations:"
echo "gcloud logging read 'protoPayload.metadata.dryRun=true' --limit=20"

# Step 5 (after review): Enforce
# gcloud access-context-manager perimeters dry-run enforce test-perimeter --policy=$POLICY
```

### Exercise 2: Test Data Exfiltration Prevention

```bash
# With perimeter protecting project A's GCS buckets:
# Try to copy from protected bucket to external bucket (should be blocked)
gcloud storage cp gs://protected-bucket/sensitive-data.csv \
  gs://external-project-bucket/stolen-data.csv
# Expected: AccessDenied (VPC-SC violation)

# Authorized copy (ingress rule allows this SA + origin)
gcloud storage cp gs://protected-bucket/report.csv \
  gs://authorized-export-bucket/report.csv
# Expected: SUCCESS (matches egress rule)
```

---

## Review Questions

1. Explain the difference between **IAM** and **VPC Service Controls**. Can they be used together? Are they redundant?

2. A user with `roles/storage.admin` on a GCS bucket tries to delete the bucket from outside the VPC perimeter. The perimeter protects `storage.googleapis.com`. What happens?

3. What is the difference between a **regular perimeter** and a **bridge perimeter**?

4. Your team sets up VPC-SC and immediately enforces it. 30 minutes later, your Cloud Build jobs start failing. What likely happened? How do you fix it?

5. Why should you use `restricted.googleapis.com` instead of `private.googleapis.com` when inside a VPC-SC perimeter?

---

## Key Exam Points

- **VPC-SC is not IAM** — it's a network-level API boundary; both are needed together
- **Dry-run FIRST** — always. Enforcing immediately without testing breaks production
- **Perimeter protects services, not projects** — you list both the services AND the projects
- **Ingress = external to internal, Egress = internal to external**
- **Service account identity matters** — if Cloud Build's SA isn't in ingress rules, Cloud Build fails
- **`restricted.googleapis.com`** routes via VPC-SC enforcement path — required for VMs inside perimeter
- **Bridge perimeter** allows two separate perimeters to exchange data — not a bypass of both perimeters
- **VPC-SC violations = HTTP 403** — log `protoPayload.status.code=403` with VpcServiceControlAuditMetadata
- **Access Transparency** does NOT bypass VPC-SC — Google operators are also subject to perimeters
