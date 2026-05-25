# 15 — Audit Logging & Cloud Logging

> **Domain 4 | Weight: ~6% of total exam**  
> **Time:** ~4 hours concept + 2 hours hands-on

---

## Concepts

### 1. Audit Log Types

| Log Type | What It Records | On by Default | Chargeable | Can Disable? |
|----------|----------------|--------------|-----------|-------------|
| **Admin Activity** | Config/metadata changes (create VM, change IAM) | YES | NO | NO |
| **Data Access** | Read/write data (read GCS object, query BigQuery) | NO (must enable) | YES | YES |
| **System Event** | GCP-generated events (live migration, auto-scaling) | YES | NO | NO |
| **Policy Denied** | Requests denied by org policy | YES | NO | NO |

**Critical exam fact:** Admin Activity logs are **always on** and **cannot be disabled**.

---

### 2. Admin Activity Log Examples

| Service | Operation Logged |
|---------|----------------|
| IAM | `setIamPolicy`, `createServiceAccount` |
| Compute | `insert` (create VM), `delete` (delete VM) |
| Storage | `create` (create bucket), `delete` (delete bucket) |
| Cloud SQL | `create` (create instance), `patch` (modify settings) |
| GKE | `create` (create cluster), `updateMaster` |
| KMS | `createKeyRing`, `createCryptoKey`, `destroyCryptoKeyVersion` |

---

### 3. Data Access Log Subtypes

| Subtype | What It Records |
|---------|----------------|
| `DATA_READ` | Reading data (GET, LIST operations) |
| `DATA_WRITE` | Writing data (INSERT, UPDATE, DELETE) |
| `ADMIN_READ` | Reading metadata/configuration (describe, list) |

**Enable per service, per subtype, per project/folder/org.**

Common services to enable Data Access logs for:
- `storage.googleapis.com` — who read/wrote which GCS objects
- `bigquery.googleapis.com` — which queries ran, who accessed what
- `cloudkms.googleapis.com` — who used which key
- `secretmanager.googleapis.com` — who accessed which secret
- `iam.googleapis.com` — who viewed IAM policies

---

### 4. Log Buckets

Logs are stored in **log buckets** (not Cloud Storage buckets):

| Bucket | Retention | Deletable | Note |
|--------|-----------|----------|------|
| `_Required` | 400 days | NO | Admin Activity, System Event, Access Transparency |
| `_Default` | 30 days | NO (but can upgrade) | Everything else by default |
| Custom | 1–3650 days | YES | You define |

**`_Required` bucket cannot be modified or deleted** — regulatory-grade log protection.

---

### 5. Log Sinks (Export)

Export logs from Cloud Logging to external destinations:

| Destination | Use Case |
|------------|---------|
| Cloud Storage | Long-term archival, compliance |
| BigQuery | Analysis, SQL queries on logs |
| Pub/Sub | Real-time streaming to SIEM, alerts |
| Cloud Logging bucket (different project) | Centralized security log project |
| Splunk / Chronicle | via Pub/Sub or direct integration |

**Aggregated sinks** — export logs from all projects in a folder or org to a single destination.

---

### 6. Log-Based Metrics and Alerts

Create metrics from log patterns → trigger alerts:

```
Log entry matches filter
       ↓
Log-based metric incremented
       ↓
Alerting policy threshold exceeded
       ↓
Notification (email, PagerDuty, Pub/Sub)
```

**Security alerts to create:**
- Root/`roles/owner` granted to a user
- Service account key created
- Firewall rule allowing `0.0.0.0/0` created
- Org policy changed
- VPC Service Control violation
- Secret Manager secret accessed outside business hours
- IAM deny policy changed

---

### 7. Log Exclusions

Reduce log volume and cost by excluding unwanted log entries.

```bash
# Exclude health check logs
gcloud logging sinks update _Default \
  --add-exclusion='name=exclude-health-checks,filter=resource.type="http_load_balancer" AND httpRequest.requestUrl=~"^/health"'
```

---

### 8. Access Transparency Logs

Logs when **Google staff** access your customer data:
- Available in Premium/Enterprise SCC tiers
- Separate from audit logs
- Shows reason for access (support ticket, legal, maintenance)
- Role to view: `roles/axt.viewer`

---

### 9. Log Analytics

Run SQL queries on logs in Cloud Logging (without exporting to BigQuery):

```sql
-- Top IAM changes in last 7 days
SELECT
  protopayload_auditlog.authenticationInfo.principalEmail as principal,
  protopayload_auditlog.methodName as action,
  COUNT(*) as count
FROM `PROJECT_ID.global._Default._AllLogs`
WHERE
  log_id = "cloudaudit.googleapis.com/activity"
  AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY 1, 2
ORDER BY 3 DESC
LIMIT 20
```

---

## gcloud Commands

### Viewing Audit Logs
```bash
# View Admin Activity logs
gcloud logging read \
  'log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Factivity"' \
  --limit=20 \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# View IAM changes only
gcloud logging read \
  'log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Factivity" AND
   protoPayload.serviceName="iam.googleapis.com"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail)"

# View Data Access logs (must be enabled first)
gcloud logging read \
  'log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Fdata_access"' \
  --limit=20

# View GCS data access
gcloud logging read \
  'log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Fdata_access" AND
   protoPayload.serviceName="storage.googleapis.com"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# View KMS key usage
gcloud logging read \
  'log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Fdata_access" AND
   protoPayload.serviceName="cloudkms.googleapis.com"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail)"

# View firewall rule changes
gcloud logging read \
  'protoPayload.methodName=~"firewalls" AND
   log_name="projects/PROJECT_ID/logs/cloudaudit.googleapis.com%2Factivity"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail,protoPayload.request)"
```

### Enabling Data Access Logs
```bash
# Get current audit log config
gcloud projects get-iam-policy PROJECT_ID --format=json | jq '.auditConfigs'

# Enable Data Access logs for ALL services (expensive — use selectively)
cat > enable-all-data-access.yaml << 'EOF'
auditConfigs:
  - auditLogConfigs:
      - logType: DATA_READ
      - logType: DATA_WRITE
      - logType: ADMIN_READ
    service: allServices
EOF

gcloud projects set-iam-policy PROJECT_ID enable-all-data-access.yaml

# Enable only for specific services (recommended)
cat > selective-data-access.yaml << 'EOF'
auditConfigs:
  - auditLogConfigs:
      - logType: DATA_READ
      - logType: DATA_WRITE
    service: storage.googleapis.com
  - auditLogConfigs:
      - logType: DATA_READ
      - logType: DATA_WRITE
    service: bigquery.googleapis.com
  - auditLogConfigs:
      - logType: DATA_READ
    service: cloudkms.googleapis.com
  - auditLogConfigs:
      - logType: DATA_READ
    service: secretmanager.googleapis.com
EOF

gcloud projects set-iam-policy PROJECT_ID selective-data-access.yaml
```

### Log Buckets
```bash
# List log buckets in a project
gcloud logging buckets list --project=PROJECT_ID --location=global

# Create a custom log bucket (retain for 1 year)
gcloud logging buckets create security-logs \
  --project=PROJECT_ID \
  --location=global \
  --retention-days=365 \
  --description="Security audit logs"

# Create a log bucket with CMEK
gcloud logging buckets create cmek-logs \
  --project=PROJECT_ID \
  --location=us-central1 \
  --retention-days=365 \
  --kms-key-name=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key

# Update retention on existing bucket
gcloud logging buckets update security-logs \
  --project=PROJECT_ID \
  --location=global \
  --retention-days=730

# Upgrade _Default bucket to use Log Analytics (enables SQL queries)
gcloud logging buckets update _Default \
  --project=PROJECT_ID \
  --location=global \
  --enable-analytics
```

### Log Sinks
```bash
# Create a sink to export ALL admin activity to Cloud Storage
gcloud logging sinks create security-audit-archive \
  storage.googleapis.com/my-audit-archive-bucket \
  --log-filter='log_name=~"cloudaudit.googleapis.com"' \
  --project=PROJECT_ID

# Create a sink to BigQuery for analysis
gcloud logging sinks create logs-to-bigquery \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/log_analysis \
  --log-filter='resource.type="gce_instance"' \
  --project=PROJECT_ID \
  --use-partitioned-tables

# Create a sink to Pub/Sub for real-time SIEM
gcloud logging sinks create security-stream \
  pubsub.googleapis.com/projects/PROJECT_ID/topics/security-events \
  --log-filter='severity>=WARNING OR log_name=~"cloudaudit.googleapis.com"' \
  --project=PROJECT_ID

# Create an aggregated sink at the ORG level (all projects)
gcloud logging sinks create org-audit-sink \
  bigquery.googleapis.com/projects/SECURITY_PROJECT_ID/datasets/org_audit_logs \
  --organization=ORG_ID \
  --include-children \
  --log-filter='log_name=~"cloudaudit.googleapis.com"'

# List sinks
gcloud logging sinks list --project=PROJECT_ID

# Describe a sink (get writer identity for IAM grants)
gcloud logging sinks describe security-audit-archive --project=PROJECT_ID

# Grant the sink's SA write access to the destination
WRITER_SA=$(gcloud logging sinks describe security-audit-archive \
  --project=PROJECT_ID \
  --format="value(writerIdentity)")

# For Cloud Storage destination
gcloud storage buckets add-iam-policy-binding gs://my-audit-archive-bucket \
  --member=$WRITER_SA \
  --role=roles/storage.objectCreator

# For BigQuery destination
bq add-iam-policy-binding \
  --member=$WRITER_SA \
  --role=roles/bigquery.dataEditor \
  PROJECT_ID:log_analysis

# For Pub/Sub destination
gcloud pubsub topics add-iam-policy-binding security-events \
  --member=$WRITER_SA \
  --role=roles/pubsub.publisher
```

### Log Exclusions
```bash
# Add exclusion to _Default sink (health checks)
gcloud logging sinks update _Default \
  --project=PROJECT_ID \
  --add-exclusion='name=no-health-checks,filter=resource.type="http_load_balancer" AND httpRequest.requestUrl=~"/_ah/health|/health|/healthz"'

# Add exclusion for debug-level logs
gcloud logging sinks update _Default \
  --project=PROJECT_ID \
  --add-exclusion='name=no-debug,filter=severity="DEBUG"'

# List exclusions
gcloud logging sinks describe _Default --project=PROJECT_ID | grep -A 20 exclusions
```

### Log-Based Metrics and Alerts
```bash
# Create a metric: count owner role grants
gcloud logging metrics create owner-role-grants \
  --description="Count of owner role grants" \
  --log-filter='resource.type="project" AND 
    protoPayload.methodName="SetIamPolicy" AND 
    protoPayload.serviceData.policyDelta.bindingDeltas.role="roles/owner" AND
    protoPayload.serviceData.policyDelta.bindingDeltas.action="ADD"'

# Create metric: SA key created
gcloud logging metrics create sa-key-created \
  --description="Service account key creation events" \
  --log-filter='resource.type="service_account" AND 
    protoPayload.methodName="google.iam.admin.v1.CreateServiceAccountKey"'

# Create metric: firewall rule opening 0.0.0.0/0
gcloud logging metrics create open-firewall \
  --description="Firewall rules allowing 0.0.0.0/0" \
  --log-filter='protoPayload.methodName="v1.compute.firewalls.insert" AND 
    protoPayload.request.sourceRanges:"0.0.0.0/0" AND
    protoPayload.request.allowed.ports:"22"'

# Create an alerting policy via gcloud (for the metric above)
cat > alert-policy.json << 'EOF'
{
  "displayName": "Alert: Owner Role Granted",
  "conditions": [
    {
      "displayName": "Owner role granted",
      "conditionThreshold": {
        "filter": "metric.type=\"logging.googleapis.com/user/owner-role-grants\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0,
        "duration": "0s"
      }
    }
  ],
  "notificationChannels": ["projects/PROJECT_ID/notificationChannels/CHANNEL_ID"],
  "alertStrategy": {
    "autoClose": "604800s"
  }
}
EOF

gcloud alpha monitoring policies create --policy-from-file=alert-policy.json
```

### Querying Logs
```bash
# Find who deleted a GCS bucket
gcloud logging read \
  'protoPayload.methodName="storage.buckets.delete"' \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# Find all logins from outside expected IP range
gcloud logging read \
  'protoPayload.serviceName="cloudresourcemanager.googleapis.com" AND
   NOT protoPayload.requestMetadata.callerIp:"203.0.113"' \
  --limit=20

# Find Secret Manager accesses
gcloud logging read \
  'protoPayload.methodName="google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion"' \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# Find all IAM policy changes in last 24 hours
gcloud logging read \
  'protoPayload.methodName="SetIamPolicy" AND
   timestamp >= "2026-05-24T00:00:00Z"' \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"
```

---

## Hands-On Practice

### Exercise 1: Enable Critical Data Access Logging

```bash
PROJECT_ID=$(gcloud config get-value project)

# Get current IAM policy as JSON
gcloud projects get-iam-policy $PROJECT_ID --format=json > current-policy.json

# Add audit log configs for security-critical services
# (Merge with existing policy)
python3 << 'EOF'
import json

with open('current-policy.json') as f:
    policy = json.load(f)

audit_configs = policy.get('auditConfigs', [])
services_to_audit = [
    'storage.googleapis.com',
    'bigquery.googleapis.com',
    'cloudkms.googleapis.com',
    'secretmanager.googleapis.com',
    'iam.googleapis.com',
]

for service in services_to_audit:
    # Check if already exists
    existing = next((ac for ac in audit_configs if ac.get('service') == service), None)
    if not existing:
        audit_configs.append({
            'service': service,
            'auditLogConfigs': [
                {'logType': 'DATA_READ'},
                {'logType': 'DATA_WRITE'},
            ]
        })

policy['auditConfigs'] = audit_configs
with open('updated-policy.json', 'w') as f:
    json.dump(policy, f, indent=2)
print("Updated policy written to updated-policy.json")
EOF

gcloud projects set-iam-policy $PROJECT_ID updated-policy.json
echo "Data access logging enabled for critical services"
```

### Exercise 2: Create Centralized Security Log Sink

```bash
PROJECT_ID=$(gcloud config get-value project)
SECURITY_BUCKET="security-logs-${PROJECT_ID}"

# Create archive bucket with 7-year retention
gcloud storage buckets create gs://$SECURITY_BUCKET \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --public-access-prevention

gcloud storage buckets update gs://$SECURITY_BUCKET \
  --retention-period=7y

# Create sink for all audit logs
gcloud logging sinks create centralized-audit \
  storage.googleapis.com/$SECURITY_BUCKET \
  --log-filter='log_name=~"cloudaudit.googleapis.com"' \
  --project=$PROJECT_ID

# Get writer identity and grant access
WRITER=$(gcloud logging sinks describe centralized-audit \
  --project=$PROJECT_ID --format="value(writerIdentity)")

gcloud storage buckets add-iam-policy-binding gs://$SECURITY_BUCKET \
  --member=$WRITER \
  --role=roles/storage.objectCreator

echo "Audit logs will be archived to gs://$SECURITY_BUCKET"
```

---

## Review Questions

1. A security auditor asks for all API calls made in your project for the last 6 months. Admin Activity logs are retained for 400 days. Data Access logs were never enabled. What can you provide and what's missing?

2. You want to export logs from ALL projects in your organization to a central BigQuery dataset. What type of log sink do you create?

3. What is the difference between the `_Required` and `_Default` log buckets? Which one retains Admin Activity logs?

4. A developer claims they never deleted a GCS bucket. How do you prove or disprove this using Cloud Logging?

5. Your log sink's service account needs permission to write to a BigQuery dataset. What IAM role do you grant to the sink's writer identity on the BigQuery dataset?

---

## Key Exam Points

- **Admin Activity logs = always on, cannot disable, free** — always available for forensics
- **Data Access logs = off by default, you pay for them** — enable selectively for critical services
- **`_Required` bucket has 400-day retention** and cannot be modified — use for compliance
- **Aggregated log sinks** use `--include-children` at org or folder level — captures all child projects
- **Log sink writer identity needs IAM on destination** — always run this after creating a sink
- **Log-based metrics + alerting** = real-time security monitoring without SIEM
- **Access Transparency** logs are in `_Required` bucket — not affected by your log configs
- **Policy Denied logs** = when org policy DENIES a request — useful for VPC-SC violation detection
