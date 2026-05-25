# 19 — Compliance & Assured Workloads

> **Domain 5 | Weight: 8% of total exam**  
> **Time:** ~3 hours concept + 1 hour hands-on

---

## Concepts

### 1. Compliance Frameworks

| Framework | Scope | Key Requirements |
|-----------|-------|-----------------|
| **GDPR** | EU personal data | Data residency, right to erasure, DPA, breach notification |
| **HIPAA** | US healthcare (PHI) | BAA with Google, access controls, audit logs, encryption |
| **PCI DSS** | Payment card data | Network segmentation, encryption, access controls, logging |
| **SOC 2** | Service organization controls | Security, availability, confidentiality, privacy |
| **ISO 27001** | Information security management | Risk management, controls framework |
| **FedRAMP** | US federal data | Government-approved cloud services, NIST controls |
| **IL2/IL4/IL5** | US DoD data | Impact levels for controlled unclassified info |
| **ITAR** | Export-controlled data | No non-US persons access, data location controls |
| **CJIS** | Criminal justice info | FBI-controlled, background checks, encryption |
| **NIST CSF** | Security framework | Identify, Protect, Detect, Respond, Recover |
| **CIS Benchmarks** | Cloud config hardening | GCP CIS Benchmark v2.0 — specific control checks |

---

### 2. Assured Workloads

**Assured Workloads** provides compliance controls for regulated workloads on GCP.

| Compliance Package | What It Enforces |
|-------------------|-----------------|
| `FedRAMP_MODERATE` | US government services, NIST 800-53 controls |
| `FedRAMP_HIGH` | Higher-impact government data |
| `IL2` | DoD Impact Level 2 |
| `IL4` | DoD Impact Level 4 (CUI) |
| `IL5` | DoD Impact Level 5 (secret/sensitive) |
| `ITAR` | International Traffic in Arms Regulations |
| `CJIS` | Criminal Justice Information Services |
| `HIPAA` | Health data controls |
| `EU_REGIONS_AND_SUPPORT` | EU data sovereignty |
| `SOVEREIGN_CONTROLS_BY_PARTNERS` | Sovereign controls via T-Systems |

**What Assured Workloads does:**
1. Creates a **dedicated folder** with compliance controls applied
2. Enforces **org policies** for data residency, personnel controls, encryption
3. Restricts which GCP services are available in the folder
4. Maintains **compliance audit trail**
5. Applies **access transparency** to Google operator access

---

### 3. Key Compliance Controls in GCP

#### Data Residency
- **`constraints/gcp.resourceLocations`** — restrict where resources can be created
- **Dual-region buckets** — explicitly pin replicas to specific regions
- **Spanner multi-region configs** — specific multi-region configs for residency

#### Encryption Requirements
- **CMEK mandatory** — many frameworks require customer-controlled keys
- **Cloud HSM** — FIPS 140-2 Level 3 required for PCI HSM, FedRAMP High
- **Key Access Justifications** — audit/approve Google operator key usage

#### Access Controls
- **MFA enforcement** — for PCI, HIPAA, FedRAMP
- **Domain restriction** — `constraints/iam.allowedPolicyMemberDomains`
- **Privileged access management** — PAM for just-in-time admin access
- **Separation of duties** — KMS admin ≠ KMS encrypter

#### Audit Logging
- **Admin Activity logs** — always on (meets most frameworks)
- **Data Access logs** — must be enabled (required for HIPAA, PCI)
- **Log retention** — align with framework (HIPAA: 6 years, PCI: 1 year)
- **Log export** — to immutable storage (compliance archive)

---

### 4. CIS GCP Benchmark v2.0

The CIS Benchmark provides specific configuration checks:

**Section 1 — IAM:**
- 1.1: Ensure super admin is NOT a service account
- 1.2: Ensure 2FA is enforced for admin accounts
- 1.3: Ensure users are NOT using basic email for billing
- 1.4: Ensure service account keys are rotated within 90 days
- 1.5: Ensure API keys are restricted and scoped

**Section 2 — Logging:**
- 2.1: Ensure cloud audit logging is enabled
- 2.2: Ensure activity logs are retained for 1+ years
- 2.3: Ensure log sinks are configured for all log types

**Section 3 — Networking:**
- 3.1: Ensure default firewall rules block everything except health checks
- 3.2: Ensure no firewall rules allow `0.0.0.0/0` on SSH/RDP
- 3.3: Ensure all VPCs use flow logs

**Section 4 — VMs:**
- 4.1: Ensure VMs use Shielded VM features
- 4.2: Ensure VMs do NOT have external IPs unless required
- 4.3: Ensure disks are encrypted with CMEK

**Section 5 — Storage:**
- 5.1: Ensure buckets do NOT have public access
- 5.2: Ensure uniform bucket-level access is enabled
- 5.3: Ensure GCS buckets use retention policies

**Section 6 — Cloud SQL:**
- 6.1: Ensure Cloud SQL does NOT have public IP unless required
- 6.2: Ensure SSL/TLS is required for all connections
- 6.3: Ensure automated backups are enabled

---

### 5. SCC Compliance Dashboards

SCC Premium maps findings to compliance frameworks:
- PCI DSS
- CIS GCP Benchmark
- NIST SP 800-53
- ISO 27001

You can see:
- How many controls are passing vs. failing
- Which findings map to which controls
- Historical compliance trend

---

### 6. Google Cloud Compliance Reports Manager

Download official compliance reports at any time:
- SOC 2, SOC 3
- ISO 27001, 27017, 27018
- PCI DSS AOC
- FedRAMP authorizations
- HIPAA BAA

Access: **console.cloud.google.com/compliance**

---

### 7. Business Associate Agreement (BAA) — HIPAA

For HIPAA compliance, you must sign a **BAA with Google** before storing PHI on GCP.

- BAA is available for Google Workspace and GCP
- After signing, you're responsible for configuring GCP services compliantly
- Google's HIPAA coverage includes: Cloud Storage, BigQuery, Cloud SQL, GKE, App Engine, Cloud Run, Secret Manager, Cloud KMS, and more

---

### 8. Access Transparency

Logs when **Google personnel** access your project for support, maintenance, etc.:
- Available via: SCC Premium, Google Workspace Enterprise
- Provides reason code for each access
- View in Cloud Logging under `access_transparency`

**Reason codes:**
- `CUSTOMER_INITIATED_SUPPORT` — you opened a ticket
- `GOOGLE_INITIATED_SERVICE` — automated GCP operations
- `LEGAL_PROCESS` — legal/law enforcement request

---

## gcloud Commands

### Assured Workloads
```bash
# List Assured Workloads
gcloud assured workloads list \
  --location=us-central1 \
  --organization=ORG_ID

# Create an Assured Workload (FedRAMP Moderate)
gcloud assured workloads create \
  --organization=ORG_ID \
  --location=us-central1 \
  --compliance-regime=FEDRAMP_MODERATE \
  --display-name="FedRAMP Production" \
  --billing-account=billingAccounts/BILLING_ID \
  --resource-settings=CONSUMER_PROJECT_ID=assuredproject1

# Describe an Assured Workload
gcloud assured workloads describe WORKLOAD_ID \
  --organization=ORG_ID \
  --location=us-central1

# Update Assured Workload
gcloud assured workloads update WORKLOAD_ID \
  --organization=ORG_ID \
  --location=us-central1 \
  --display-name="FedRAMP Production Updated"

# Delete Assured Workload
gcloud assured workloads delete WORKLOAD_ID \
  --organization=ORG_ID \
  --location=us-central1

# Acknowledge a compliance violation
gcloud assured workloads violations acknowledge VIOLATION_ID \
  --workload=WORKLOAD_ID \
  --organization=ORG_ID \
  --location=us-central1 \
  --acknowledgement-type=EXCEPTION_ACKNOWLEDGEMENT
```

### Data Residency Controls
```bash
# Restrict resources to specific regions (EU only example)
cat > eu-residency.yaml << 'EOF'
constraint: constraints/gcp.resourceLocations
listPolicy:
  allowedValues:
    - in:eu-locations
EOF

gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  eu-residency.yaml

# Verify effective policy
gcloud resource-manager org-policies describe \
  constraints/gcp.resourceLocations \
  --organization=ORG_ID \
  --effective

# Create Cloud Storage bucket with explicit region (no auto-replication)
gcloud storage buckets create gs://eu-only-data \
  --location=EUROPE-WEST1 \
  --uniform-bucket-level-access \
  --public-access-prevention

# Create a dual-region bucket (EU residency)
gcloud storage buckets create gs://eu-dual-region-data \
  --location=EU \  # EU dual-region: europe-west1 + europe-west4
  --uniform-bucket-level-access
```

### Compliance Audit — CIS Benchmark Check Script
```bash
#!/bin/bash
# Quick CIS GCP Benchmark checks
PROJECT_ID=$(gcloud config get-value project)

echo "=== CIS GCP Benchmark Quick Audit ==="
echo "Project: $PROJECT_ID"
echo ""

# Check 1: Owner role not assigned to users
echo "--- Check 1: Owner role assignments ---"
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role=roles/owner AND bindings.members=user:*" \
  --format="table(bindings.members)"

# Check 2: Data Access logging
echo "--- Check 2: Data Access logging ---"
gcloud projects get-iam-policy $PROJECT_ID --format=json | \
  python3 -c "
import json, sys
p = json.load(sys.stdin)
audit = p.get('auditConfigs', [])
if not audit:
    print('WARNING: No Data Access logging configured')
else:
    for a in audit:
        print(f\"Service: {a['service']}: {[c['logType'] for c in a.get('auditLogConfigs', [])]}\")"

# Check 3: Public GCS buckets
echo "--- Check 3: Public GCS buckets ---"
gcloud storage buckets list --format="value(name)" | while read b; do
  PUBLIC=$(gcloud storage buckets get-iam-policy gs://$b 2>/dev/null | \
    grep -c "allUsers\|allAuthenticatedUsers" || echo 0)
  [ "$PUBLIC" -gt 0 ] && echo "PUBLIC: $b"
done

# Check 4: VMs with external IPs
echo "--- Check 4: VMs with external IPs ---"
gcloud compute instances list \
  --filter="networkInterfaces.accessConfigs.natIP:*" \
  --format="table(name,zone,networkInterfaces[0].accessConfigs[0].natIP)"

# Check 5: Firewall rules with 0.0.0.0/0 on port 22/3389
echo "--- Check 5: Open SSH/RDP firewall rules ---"
gcloud compute firewall-rules list \
  --filter="allowed.ports:(22,3389) AND sourceRanges=0.0.0.0/0 AND direction=INGRESS" \
  --format="table(name,allowed.ports,sourceRanges)"

# Check 6: Cloud SQL with public IP
echo "--- Check 6: Cloud SQL instances with public IP ---"
gcloud sql instances list \
  --format="table(name,region,settings.ipConfiguration.ipv4Enabled,settings.ipConfiguration.requireSsl)"

# Check 7: KMS key rotation
echo "--- Check 7: KMS keys not recently rotated ---"
gcloud kms keys list --location=us-central1 \
  --format="table(name,rotationPeriod,nextRotationTime)" 2>/dev/null | head -20

echo ""
echo "=== Audit complete. Review findings above. ==="
```

### Log Retention for Compliance
```bash
# Set up 7-year log retention for compliance
gcloud logging buckets create compliance-archive \
  --project=PROJECT_ID \
  --location=global \
  --retention-days=2555  # 7 years

# Create sink to archive all audit logs
gcloud logging sinks create compliance-sink \
  storage.googleapis.com/my-compliance-archive-bucket \
  --log-filter='log_name=~"cloudaudit.googleapis.com"' \
  --project=PROJECT_ID

# Set 7-year retention on archive bucket
gcloud storage buckets update gs://my-compliance-archive-bucket \
  --retention-period=7y

# Lock retention (makes it WORM — cannot be deleted)
gcloud storage buckets update gs://my-compliance-archive-bucket \
  --lock-retention-policy
```

### Access Transparency Logs
```bash
# View Access Transparency logs
gcloud logging read \
  'log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Faccess_transparency"' \
  --format="table(timestamp,jsonPayload.principalOfficeCountry,jsonPayload.justifications,jsonPayload.accesses)"

# Set up alert for Google personnel access
gcloud logging metrics create google-staff-access \
  --description="Google personnel accessed customer data" \
  --log-filter='log_name=~"access_transparency"'
```

---

## Hands-On Practice

### Exercise 1: Assess Compliance Against CIS Benchmark

```bash
# Run the CIS audit script from above and document findings
# Then fix the most critical ones:

PROJECT_ID=$(gcloud config get-value project)

# Fix 1: Enable Data Access logging for critical services
gcloud projects get-iam-policy $PROJECT_ID --format=json > current.json
# (Merge audit configs as shown in file 15)

# Fix 2: Prevent public buckets
gcloud resource-manager org-policies enable-enforce \
  constraints/storage.publicAccessPrevention \
  --project=$PROJECT_ID

# Fix 3: Require SSL on all Cloud SQL instances
gcloud sql instances list --format="value(name)" | while read inst; do
  gcloud sql instances patch $inst --require-ssl
done

echo "Basic CIS fixes applied"
```

### Exercise 2: Set Up Compliance Log Archive

```bash
PROJECT_ID=$(gcloud config get-value project)
ARCHIVE_BUCKET="compliance-archive-${PROJECT_ID}"

# Create archive bucket in US (or EU based on requirement)
gcloud storage buckets create gs://$ARCHIVE_BUCKET \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --public-access-prevention

# Set 7-year retention
gcloud storage buckets update gs://$ARCHIVE_BUCKET \
  --retention-period=7y

# Create sink for all audit logs
gcloud logging sinks create compliance-log-sink \
  storage.googleapis.com/$ARCHIVE_BUCKET \
  --log-filter='log_name=~"cloudaudit.googleapis.com"' \
  --project=$PROJECT_ID

# Grant write permission to sink SA
WRITER=$(gcloud logging sinks describe compliance-log-sink \
  --project=$PROJECT_ID --format="value(writerIdentity)")
gcloud storage buckets add-iam-policy-binding gs://$ARCHIVE_BUCKET \
  --member=$WRITER \
  --role=roles/storage.objectCreator

echo "Compliance log archive ready: gs://$ARCHIVE_BUCKET"
echo "Retention: 7 years"
```

---

## Review Questions

1. A healthcare company wants to store patient data (PHI) on GCP. What steps must they take before going live?

2. Your organization needs FedRAMP Moderate authorization. What GCP service should you use, and what does it automatically configure?

3. A PCI DSS requirement states that audit logs must be retained for at least 12 months. How do you satisfy this in GCP?

4. What is the difference between **data residency** and **data sovereignty**? Which GCP feature addresses each?

5. Your CISO asks you to prove that Google cannot access your encryption keys without your approval. What GCP feature enables this?

---

## Key Exam Points

- **HIPAA requires a BAA with Google** — not just enabling controls
- **Assured Workloads creates a folder** with enforced controls — use it for regulated workloads
- **`constraints/gcp.resourceLocations`** = data residency enforcement
- **FIPS 140-2 Level 3** = Cloud HSM — required for some FedRAMP and DoD requirements
- **Key Access Justifications** = you can deny Google operator access — strongest sovereignty control
- **Access Transparency** shows Google staff access — requires SCC Premium or Enterprise
- **PCI DSS requires** — encryption, network segmentation, access controls, logging, vulnerability scans
- **CIS Benchmark checks** map directly to SCC Security Health Analytics findings
- **Log retention for PCI** = 1 year (3 months immediately available, 9 months archived)
- **Log retention for HIPAA** = 6 years for records, audit logs follow BAA terms
