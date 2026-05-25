# 16 — Security Command Center (SCC)

> **Domain 4 | Weight: ~6% of total exam**  
> **Time:** ~4 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. SCC Overview and Tiers

| Tier | Key Features | Cost |
|------|-------------|------|
| **Standard** | Asset inventory, Security Health Analytics (basic), basic vulnerability scanning | Free |
| **Premium** | Full SHA, Event Threat Detection, Container Threat Detection, VM Threat Detection, Web Security Scanner, Rapid Vulnerability Detection, compliance dashboards | Paid |
| **Enterprise** | Chronicle SIEM, SOAR, case management, ticketing, multi-cloud threat detection | Higher tier |

---

### 2. Finding Types

| Category | Description |
|----------|-------------|
| **VULNERABILITY** | Security weaknesses (open port, unencrypted disk, public bucket) |
| **MISCONFIGURATION** | Improper settings that violate best practices |
| **THREAT** | Active attacks or suspicious activity |
| **POSTURE_VIOLATION** | Drift from desired security posture |
| **OBSERVATION** | Informational — doesn't require action |

---

### 3. Built-in Sources

#### Security Health Analytics (SHA)

Detects **misconfigurations** across GCP services. Runs daily and on config changes.

**High-value findings:**
| Finding Name | What It Means |
|------------|---------------|
| `OPEN_FIREWALL` | Firewall allows `0.0.0.0/0` ingress on sensitive ports |
| `PUBLIC_BUCKET_ACL` | GCS bucket is publicly accessible |
| `AUDIT_LOGGING_DISABLED` | Project has Data Access logs disabled |
| `LEGACY_AUTHORIZATION_ENABLED` | ABAC enabled on GKE (use RBAC instead) |
| `MASTER_AUTHORIZED_NETWORKS_DISABLED` | GKE API server exposed without IP filtering |
| `CLUSTER_SHIELDED_NODES_DISABLED` | GKE nodes don't use Shielded VM |
| `CLUSTER_PRIVATE_GOOGLE_ACCESS_DISABLED` | Nodes can't access APIs privately |
| `KMS_KEY_NOT_ROTATED` | KMS key not rotated in 90+ days |
| `PRIMITIVE_ROLES_USED` | Basic roles (Owner/Editor) granted to users |
| `OVER_PRIVILEGED_SERVICE_ACCOUNT` | SA has broader permissions than needed |
| `SERVICE_ACCOUNT_KEY_NOT_ROTATED` | SA key older than 90 days |
| `MFA_NOT_ENFORCED` | Users without 2FA enrolled |
| `DEFAULT_SERVICE_ACCOUNT_USED` | Default compute SA used (has editor role) |

#### Event Threat Detection (ETD)

Detects **active threats** by analyzing Admin Activity and Data Access logs in real-time.

| Finding | Description |
|---------|-------------|
| `Malware: Bad Domain` | VM communicated with known malware domain |
| `Malware: Cryptomining` | Cryptocurrency mining activity detected |
| `Exfiltration: BigQuery Data Exfil` | Large BigQuery export to external destination |
| `Exfiltration: CloudSQL Data Exfil` | Cloud SQL data exported externally |
| `Persistence: IAM Anomalous Grant` | Unusual IAM grant pattern |
| `Persistence: New API Method` | Account calling API methods never used before |
| `Initial Access: Leaked Credentials` | Credentials found in public code repos |
| `Brute Force: SSH` | Repeated SSH login failures |
| `Defense Evasion: Delete Activity Logs` | Attempt to delete audit logs |
| `Privilege Escalation: AlloyDB SA Impersonation` | SA used for privilege escalation |

#### Container Threat Detection (CTD)

Detects **runtime container attacks** in GKE clusters.

| Finding | Description |
|---------|-------------|
| `Added Library Loaded` | Unexpected library loaded into container |
| `Reverse Shell` | Container opens outbound shell connection |
| `Unexpected Child Shell` | Shell process spawned from non-shell parent |
| `Malicious Script Executed` | Known malicious script executed |
| `Container Running as Root` | Container runs as UID 0 |

#### VM Threat Detection (VMTD)

Detects **threats at VM memory level** (not just network/log-based).

| Finding | Description |
|---------|-------------|
| `Execution: Cryptocurrency Mining - YARA` | Mining code in VM memory |
| `Defense Evasion: Rootkit - YARA` | Rootkit signatures in memory |
| `Execution: Malware - Bad IP` | Communication with known malware IP |

#### Web Security Scanner

DAST (Dynamic Application Security Testing) for public web apps:
- SQL Injection
- XSS (Cross-Site Scripting)
- Mixed content
- Outdated libraries
- Insecure form actions
- Clear-text passwords

#### Rapid Vulnerability Detection (RVD)

Network-based vulnerability scanner:
- Scans external-facing resources (without agents)
- Detects exposed services, weak credentials, software vulnerabilities

---

### 4. Security Posture Management

**Security Posture service** (SCC Enterprise):
- Define the desired security state as **posture templates**
- Deploy postures to projects/folders/orgs
- **Detect drift** from desired state
- Findings for violations appear in SCC

---

### 5. SCC Assets

SCC maintains an **asset inventory** of all GCP resources:
- Resources are discovered automatically
- Historical snapshots for change tracking
- Query and filter assets across projects

---

### 6. SCC Notifications

**Pub/Sub notifications** — real-time finding delivery:
- Configure notification channel pointing to a Pub/Sub topic
- Subscribe downstream (Cloud Functions, Dataflow, SIEM)
- Filter by severity, finding type, asset type

---

### 7. Muting Findings

Suppress findings you've acknowledged or accepted as acceptable risk:
- **Mute** — suppresses the finding (but it's still created internally)
- **Mute configs** — rule-based automatic muting of specific patterns
- Use case: suppress `OPEN_FIREWALL` for known bastion hosts

---

## gcloud Commands

### SCC Organization Setup
```bash
# Enable SCC on your organization
gcloud services enable securitycenter.googleapis.com

# Get SCC settings for org
gcloud scc settings describe --organization=ORG_ID

# Enable SCC services
gcloud scc settings update \
  --organization=ORG_ID \
  --enable-asset-discovery

# List SCC sources
gcloud scc sources list --organization=ORG_ID
```

### Viewing Findings
```bash
# List all active findings for an organization
gcloud scc findings list ORG_ID \
  --filter="state=ACTIVE" \
  --format="table(finding.name,finding.category,finding.severity,finding.createTime)"

# List CRITICAL and HIGH findings only
gcloud scc findings list ORG_ID \
  --filter='state=ACTIVE AND (severity="CRITICAL" OR severity="HIGH")' \
  --format="table(finding.category,finding.severity,finding.resourceName,finding.createTime)"

# List findings for a specific project
gcloud scc findings list ORG_ID \
  --filter='state=ACTIVE AND resource.projectDisplayName="my-project"' \
  --format="table(finding.category,finding.severity,finding.createTime)"

# List specific finding type
gcloud scc findings list ORG_ID \
  --filter='state=ACTIVE AND finding.category="PUBLIC_BUCKET_ACL"' \
  --format="table(finding.name,finding.resourceName,finding.createTime)"

# Get details of a specific finding
gcloud scc findings list ORG_ID \
  --filter='finding.name="organizations/ORG_ID/sources/SOURCE_ID/findings/FINDING_ID"' \
  --format=yaml

# List findings by source (ETD, SHA, CTD)
gcloud scc sources list ORG_ID \
  --format="table(name,displayName)"
# Get SOURCE_ID from above, then:
gcloud scc findings list ORG_ID \
  --source=SOURCE_ID \
  --filter="state=ACTIVE" \
  --limit=20
```

### Managing Finding States
```bash
# Mark a finding as INACTIVE (resolved)
gcloud scc findings update FINDING_NAME \
  --organization=ORG_ID \
  --source=SOURCE_ID \
  --state=INACTIVE

# Mute a finding
gcloud scc findings update FINDING_NAME \
  --organization=ORG_ID \
  --source=SOURCE_ID \
  --mute=MUTED

# Bulk mute with a mute config
gcloud scc mute-configs create suppress-dev-firewall \
  --organization=ORG_ID \
  --filter='resource.projectDisplayName="dev-project" AND finding.category="OPEN_FIREWALL"' \
  --description="Suppress open firewall findings in dev project"

# List mute configs
gcloud scc mute-configs list --organization=ORG_ID
```

### SCC Pub/Sub Notifications
```bash
# Create a Pub/Sub topic for SCC notifications
gcloud pubsub topics create scc-findings-stream

# Create notification config (all CRITICAL findings)
gcloud scc notifications create critical-findings-notify \
  --organization=ORG_ID \
  --pubsub-topic=projects/PROJECT_ID/topics/scc-findings-stream \
  --filter='severity="CRITICAL" AND state="ACTIVE"' \
  --description="Stream critical findings to SIEM"

# Create notification for all findings (no filter)
gcloud scc notifications create all-findings \
  --organization=ORG_ID \
  --pubsub-topic=projects/PROJECT_ID/topics/scc-findings-stream

# List notification configs
gcloud scc notifications list --organization=ORG_ID

# Grant SCC service account publish permission to topic
# Get SCC SA:
gcloud scc settings describe --organization=ORG_ID \
  --format="value(name)"
# Grant publish role to the SCC SA
gcloud pubsub topics add-iam-policy-binding scc-findings-stream \
  --member="serviceAccount:service-ORG_NUMBER@gcp-sa-scc-notification.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

### SCC Asset Inventory
```bash
# List all assets in a project
gcloud scc assets list ORG_ID \
  --filter='resourceProperties.projectId="my-project"' \
  --format="table(asset.resourceProperties.name,asset.resourceProperties.resourceType)"

# List GCS buckets
gcloud scc assets list ORG_ID \
  --filter='asset.resourceProperties.resourceType="storage.googleapis.com/Bucket"' \
  --format="table(asset.resourceProperties.name)"

# List public GCS buckets (via asset properties)
gcloud scc assets list ORG_ID \
  --filter='asset.resourceProperties.resourceType="storage.googleapis.com/Bucket" AND 
            asset.resourceProperties.iamPolicy.bindings.members:"allUsers"'

# Get security marks on an asset
gcloud scc assets list ORG_ID \
  --filter='asset.resourceProperties.name="//storage.googleapis.com/my-bucket"' \
  --format="yaml(asset.securityMarks)"

# Update security marks on an asset
gcloud scc assets update-marks ORG_ID ASSET_ID \
  --source=SOURCE_ID \
  --security-marks=reviewed=true,owner=security-team
```

### Exporting SCC Findings to BigQuery
```bash
# Set up continuous export of findings to BigQuery
gcloud scc big-query-exports create scc-bq-export \
  --organization=ORG_ID \
  --dataset=projects/PROJECT_ID/datasets/scc_findings \
  --filter='severity="HIGH" OR severity="CRITICAL"' \
  --description="Export high severity findings to BigQuery"

# List exports
gcloud scc big-query-exports list --organization=ORG_ID

# Query exported findings in BigQuery
bq query --use_legacy_sql=false \
'SELECT
  finding.category,
  finding.severity,
  finding.resource_name,
  finding.create_time
FROM `PROJECT_ID.scc_findings.findings`
WHERE finding.state = "ACTIVE"
  AND finding.severity IN ("HIGH", "CRITICAL")
ORDER BY finding.create_time DESC
LIMIT 50'
```

---

## Hands-On Practice

### Exercise 1: Run a Security Health Check

```bash
ORG_ID="your-org-id"

# Check for critical misconfigurations
echo "=== PUBLIC BUCKETS ==="
gcloud scc findings list $ORG_ID \
  --filter='state=ACTIVE AND finding.category="PUBLIC_BUCKET_ACL"' \
  --format="table(finding.resourceName,finding.severity,finding.createTime)"

echo "=== OPEN FIREWALLS ==="
gcloud scc findings list $ORG_ID \
  --filter='state=ACTIVE AND finding.category="OPEN_FIREWALL"' \
  --format="table(finding.resourceName,finding.severity,finding.createTime)"

echo "=== PRIMITIVE ROLES ==="
gcloud scc findings list $ORG_ID \
  --filter='state=ACTIVE AND finding.category="PRIMITIVE_ROLES_USED"' \
  --format="table(finding.resourceName,finding.createTime)"

echo "=== SA KEY NOT ROTATED ==="
gcloud scc findings list $ORG_ID \
  --filter='state=ACTIVE AND finding.category="SERVICE_ACCOUNT_KEY_NOT_ROTATED"' \
  --format="table(finding.resourceName,finding.createTime)"
```

### Exercise 2: Create Finding Alert Pipeline

```bash
PROJECT_ID=$(gcloud config get-value project)
ORG_ID="your-org-id"

# 1. Create Pub/Sub topic
gcloud pubsub topics create scc-critical-alerts

# 2. Create subscription
gcloud pubsub subscriptions create scc-alerts-sub \
  --topic=scc-critical-alerts

# 3. Create SCC notification
gcloud scc notifications create critical-only \
  --organization=$ORG_ID \
  --pubsub-topic=projects/${PROJECT_ID}/topics/scc-critical-alerts \
  --filter='severity="CRITICAL" AND state="ACTIVE"'

# 4. Test: Pull messages from subscription
gcloud pubsub subscriptions pull scc-alerts-sub --auto-ack --limit=5

echo "Pipeline ready. Critical findings stream to scc-critical-alerts topic."
```

---

## Review Questions

1. What is the difference between **Security Health Analytics** and **Event Threat Detection**? Give one finding example from each.

2. SCC finds a `PUBLIC_BUCKET_ACL` finding for a known public website bucket that intentionally should be public. How do you suppress this finding?

3. Your security team wants real-time alerts when SCC detects `CRYPTOMINING` in any VM. What is the architecture to achieve this?

4. Container Threat Detection finds `REVERSE_SHELL` in a production GKE pod. Walk through your response.

5. What SCC tier is required to see Event Threat Detection findings?

---

## Key Exam Points

- **SHA = misconfigurations** (static analysis of resource config)
- **ETD = active threats** (real-time log analysis)
- **CTD = container runtime** (GKE pod-level behavioral analysis)
- **VMTD = VM memory** (memory scanning, no agent needed)
- **Web Security Scanner** does DAST — requires app to be accessible
- **SCC Standard is free** — Premium/Enterprise are paid
- **Mute** suppresses finding display, not finding creation
- **SCC notifications** require the SCC SA to have `roles/pubsub.publisher` on the topic
- **BigQuery export** of SCC findings = historical compliance analysis
- **Finding states:** ACTIVE (open), INACTIVE (resolved), MUTED (suppressed)
