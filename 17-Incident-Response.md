# 17 — Incident Response

> **Domain 4 | Weight: ~4% of total exam**  
> **Time:** ~3 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. Incident Response Phases

```
DETECT → ANALYZE → CONTAIN → ERADICATE → RECOVER → LESSONS LEARNED
```

| Phase | GCP Actions |
|-------|------------|
| **Detect** | SCC alerts, ETD findings, log-based alerts, monitoring |
| **Analyze** | Query audit logs, check IAM policy, review flow logs, examine disk |
| **Contain** | Disable SA, quarantine VM, add Cloud Armor rule, remove IAM bindings |
| **Eradicate** | Delete malicious resources, patch vulnerabilities, reset credentials |
| **Recover** | Restore from backup, redeploy clean image, rebuild from IaC |
| **Lessons Learned** | Update detection rules, improve monitoring, fix root cause |

---

### 2. Detecting Compromised Resources

**Signs of a compromised service account:**
- Activity from unexpected IP addresses
- API calls at unusual times
- API calls to services the SA never used before
- IAM changes (granting itself more permissions)
- Large data transfers from BigQuery/GCS

**Signs of a compromised VM:**
- High CPU (cryptomining)
- Unexpected outbound connections (C2, data exfiltration)
- New processes running
- New cron jobs
- Modified files in `/etc/`, `/tmp/`, `/var/`
- VMTD finding in SCC

**Signs of a compromised GCS bucket:**
- Unexpected objects appearing
- Objects disappearing
- External access from unknown IPs in flow logs
- SCC `PUBLIC_BUCKET_ACL` finding appears suddenly

---

### 3. Containing a Compromised Service Account

```bash
# Step 1: Disable the SA immediately (blocks new authentications)
gcloud iam service-accounts disable compromised-sa@PROJECT_ID.iam.gserviceaccount.com

# Step 2: Delete all SA keys
gcloud iam service-accounts keys list --iam-account=compromised-sa@PROJECT_ID.iam.gserviceaccount.com \
  --managed-by=user --format="value(name)" | xargs -I{} gcloud iam service-accounts keys delete {} \
  --iam-account=compromised-sa@PROJECT_ID.iam.gserviceaccount.com --quiet

# Step 3: Remove all IAM bindings for this SA across the project
gcloud projects get-iam-policy PROJECT_ID --format=json | \
  python3 -c "
import json, sys
policy = json.load(sys.stdin)
sa = 'serviceAccount:compromised-sa@PROJECT_ID.iam.gserviceaccount.com'
for b in policy.get('bindings', []):
    if sa in b.get('members', []):
        b['members'].remove(sa)
policy['bindings'] = [b for b in policy['bindings'] if b.get('members')]
print(json.dumps(policy))
" > cleaned-policy.json
gcloud projects set-iam-policy PROJECT_ID cleaned-policy.json
```

---

### 4. Containing a Compromised VM

```bash
# Step 1: Isolate the VM with a quarantine firewall rule
# Block all egress (stop data exfiltration)
gcloud compute firewall-rules create quarantine-vm-egress \
  --network=default \
  --direction=EGRESS \
  --action=DENY \
  --rules=all \
  --destination-ranges=0.0.0.0/0 \
  --target-tags=quarantined \
  --priority=100

# Block all ingress
gcloud compute firewall-rules create quarantine-vm-ingress \
  --network=default \
  --direction=INGRESS \
  --action=DENY \
  --rules=all \
  --source-ranges=0.0.0.0/0 \
  --target-tags=quarantined \
  --priority=100

# Step 2: Add the quarantine tag to the compromised VM
gcloud compute instances add-tags compromised-vm \
  --tags=quarantined \
  --zone=us-central1-a

# Step 3: Take a disk snapshot for forensics BEFORE any changes
gcloud compute disks snapshot compromised-vm \
  --zone=us-central1-a \
  --snapshot-names=forensic-snapshot-$(date +%Y%m%d-%H%M%S) \
  --description="Forensic snapshot of compromised VM"

# Step 4: Stop the VM (optional — stops active attack but loses memory)
# gcloud compute instances stop compromised-vm --zone=us-central1-a

# Step 5: Remove external IP to prevent phone-home
gcloud compute instances delete-access-config compromised-vm \
  --access-config-name="External NAT" \
  --zone=us-central1-a
```

---

### 5. Forensic Analysis

```bash
# Create a forensic analysis VM (in isolated project/network)
gcloud compute instances create forensic-vm \
  --zone=us-central1-a \
  --machine-type=e2-standard-8 \
  --network=forensics-vpc \
  --no-address  # No external IP

# Attach the snapshot as a disk for analysis
gcloud compute disks create forensic-disk \
  --zone=us-central1-a \
  --source-snapshot=forensic-snapshot-20260525-120000

gcloud compute instances attach-disk forensic-vm \
  --disk=forensic-disk \
  --zone=us-central1-a \
  --mode=ro  # Read-only — preserve evidence

# SSH into forensic VM and mount the disk
gcloud compute ssh forensic-vm --zone=us-central1-a \
  --command="sudo mkdir /mnt/evidence && sudo mount -o ro /dev/sdb1 /mnt/evidence"
```

---

### 6. Investigating via Audit Logs

```bash
# Timeline: What did compromised SA do?
gcloud logging read \
  'protoPayload.authenticationInfo.principalEmail="compromised-sa@PROJECT_ID.iam.gserviceaccount.com" AND
   timestamp >= "2026-05-24T00:00:00Z"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.resourceName,protoPayload.requestMetadata.callerIp)" \
  --order=asc

# Find initial compromise — first unusual activity
gcloud logging read \
  'protoPayload.authenticationInfo.principalEmail="compromised-sa@PROJECT_ID.iam.gserviceaccount.com"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.requestMetadata.callerIp)" \
  --order=asc \
  --limit=50

# Find all resources the attacker accessed
gcloud logging read \
  'protoPayload.authenticationInfo.principalEmail="compromised-sa@PROJECT_ID.iam.gserviceaccount.com" AND
   log_name=~"data_access"' \
  --format="table(timestamp,protoPayload.resourceName,protoPayload.methodName)"

# Find if attacker created new resources (persistence)
gcloud logging read \
  'protoPayload.authenticationInfo.principalEmail="compromised-sa@PROJECT_ID.iam.gserviceaccount.com" AND
   protoPayload.methodName=~"create|insert"' \
  --format="table(timestamp,protoPayload.methodName,protoPayload.resourceName)"
```

---

### 7. Credential Revocation

```bash
# Revoke all OAuth tokens for a user (if Google Account compromised)
# This is done via the Admin Console (Google Workspace)
# gcloud has no direct command for this

# Revoke a specific token
TOKEN="ya29.xxxxx"
curl -d "token=$TOKEN" https://oauth2.googleapis.com/revoke

# Reset user password (forces re-login, revokes sessions)
# Via gcloud (if you're a Workspace admin using gcloud directory API)
gcloud alpha identity users security-events list --user=user@example.com

# Remove all direct IAM grants for a compromised user
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:compromised@example.com" \
  --format="value(bindings.role)" | while read role; do
    gcloud projects remove-iam-policy-binding PROJECT_ID \
      --member="user:compromised@example.com" \
      --role="$role"
    echo "Removed role: $role"
done
```

---

### 8. Incident Response for Data Exfiltration

```bash
# Check BigQuery job history for large exports
bq ls --jobs --all --format=prettyjson \
  --filter='configuration.jobType=EXTRACT' | \
  python3 -c "
import json, sys
jobs = json.load(sys.stdin)
for job in jobs:
    if job.get('configuration', {}).get('jobType') == 'EXTRACT':
        print(job['configuration']['extract']['destinationUris'])
        print('User:', job['user_email'])
        print('Time:', job['statistics']['creationTime'])
        print()
"

# Check GCS object access logs for unexpected IPs
gcloud logging read \
  'resource.type="gcs_bucket" AND 
   resource.labels.bucket_name="sensitive-bucket" AND
   protoPayload.methodName="storage.objects.get" AND
   timestamp >= "2026-05-24T00:00:00Z"' \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.requestMetadata.callerIp)"
```

---

### 9. Post-Incident Hardening

```bash
# After incident: review and tighten permissions
# Use IAM Recommender to find over-privileged bindings
gcloud recommender recommendations list \
  --project=PROJECT_ID \
  --location=global \
  --recommender=google.iam.policy.Recommender

# Enable VPC Service Controls to prevent future exfiltration
# (See file 14)

# Enable all Data Access logging
# (See file 15)

# Create log-based alerts for suspicious patterns
# (See file 15)

# Enable SCC Premium for ETD and VMTD
# (See file 16)
```

---

## gcloud Commands — Full Incident Runbook

```bash
#!/bin/bash
# INCIDENT RESPONSE RUNBOOK — Compromised Service Account
# Usage: ./incident-response.sh PROJECT_ID SA_EMAIL INCIDENT_ID

PROJECT_ID=$1
SA_EMAIL=$2
INCIDENT_ID=$3
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

echo "=== INCIDENT $INCIDENT_ID: Responding to compromised SA: $SA_EMAIL ==="

# Step 1: Document state before changes
echo "--- Documenting current SA IAM bindings ---"
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:$SA_EMAIL" \
  --format="table(bindings.role)" > "incident-${INCIDENT_ID}-sa-roles-${TIMESTAMP}.txt"
cat "incident-${INCIDENT_ID}-sa-roles-${TIMESTAMP}.txt"

# Step 2: List SA keys
echo "--- Listing SA keys ---"
gcloud iam service-accounts keys list \
  --iam-account=$SA_EMAIL \
  --managed-by=user \
  --format="table(name,validAfterTime,validBeforeTime)" > "incident-${INCIDENT_ID}-keys-${TIMESTAMP}.txt"
cat "incident-${INCIDENT_ID}-keys-${TIMESTAMP}.txt"

# Step 3: Disable SA
echo "--- Disabling SA ---"
gcloud iam service-accounts disable $SA_EMAIL
echo "SA disabled: $SA_EMAIL"

# Step 4: Delete all user-managed keys
echo "--- Deleting SA keys ---"
gcloud iam service-accounts keys list \
  --iam-account=$SA_EMAIL \
  --managed-by=user \
  --format="value(name)" | while read key; do
    gcloud iam service-accounts keys delete $key \
      --iam-account=$SA_EMAIL --quiet
    echo "Deleted key: $key"
done

# Step 5: Get recent activity
echo "--- Fetching recent SA activity (last 24h) ---"
gcloud logging read \
  "protoPayload.authenticationInfo.principalEmail=\"$SA_EMAIL\" AND
   timestamp >= \"$(date -u -v-24H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u --date='24 hours ago' +%Y-%m-%dT%H:%M:%SZ)\"" \
  --format="table(timestamp,protoPayload.methodName,protoPayload.resourceName,protoPayload.requestMetadata.callerIp)" \
  --order=asc > "incident-${INCIDENT_ID}-activity-${TIMESTAMP}.txt"
cat "incident-${INCIDENT_ID}-activity-${TIMESTAMP}.txt"

echo "=== CONTAINMENT COMPLETE ==="
echo "Evidence files:"
ls incident-${INCIDENT_ID}-*
echo ""
echo "NEXT STEPS:"
echo "1. Review activity log for scope of compromise"
echo "2. Check if attacker created new SAs or users"  
echo "3. Review all resources SA had access to"
echo "4. Scan those resources for modifications"
echo "5. Remove SA IAM bindings if not needed"
echo "6. Create new SA with least-privilege for legitimate use"
```

---

## Hands-On Practice

### Exercise 1: Practice Containment (Use a Test SA)

```bash
PROJECT_ID=$(gcloud config get-value project)

# Create a test SA to practice on
gcloud iam service-accounts create test-compromised-sa \
  --display-name="Test Compromised SA"

# Grant it some roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Create a test key
gcloud iam service-accounts keys create test-key.json \
  --iam-account=test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com

# Now practice incident response:
echo "--- INCIDENT RESPONSE START ---"

# 1. Disable SA
gcloud iam service-accounts disable \
  test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com
echo "Step 1 done: SA disabled"

# 2. Delete keys
gcloud iam service-accounts keys delete \
  $(gcloud iam service-accounts keys list \
    --iam-account=test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com \
    --managed-by=user --format="value(name)" | head -1) \
  --iam-account=test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com --quiet
echo "Step 2 done: Keys deleted"

# 3. Verify SA is disabled and no keys exist
gcloud iam service-accounts describe \
  test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --format="get(disabled)"

gcloud iam service-accounts keys list \
  --iam-account=test-compromised-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --managed-by=user

echo "--- INCIDENT RESPONSE COMPLETE ---"
rm -f test-key.json
```

### Exercise 2: Build a Detection Alert

```bash
# Create alert: SA disabled (indicator of incident response by someone else)
gcloud logging metrics create sa-disabled-events \
  --description="Service account disabled events" \
  --log-filter='protoPayload.methodName="google.iam.admin.v1.DisableServiceAccount"'

echo "Log-based metric created: sa-disabled-events"
echo "Add an alerting policy on this metric in Cloud Monitoring"
```

---

## Review Questions

1. You discover that a service account's JSON key was committed to a public GitHub repo 3 hours ago. Walk through the complete incident response steps in order.

2. Why should you take a **disk snapshot BEFORE stopping** a compromised VM?

3. What is the difference between **disabling** a service account vs. **deleting** it? When would you use each?

4. An attacker used a compromised SA to create a new SA with admin permissions. How do you detect this? How do you contain it?

5. After containing a compromised VM, you need to determine the full blast radius of the incident. What GCP tools do you use and what do you look for?

---

## Key Exam Points

- **Disable SA immediately** — don't just delete keys first (deleting keys alone may not block existing tokens)
- **Take disk snapshot BEFORE stopping VM** — stopped VM loses some state; snapshot preserves evidence
- **Audit logs are your primary forensic source** — must be retained before incident (enable Data Access logs proactively)
- **Quarantine VM with firewall rules** — add a deny tag, don't delete the VM (need evidence)
- **Remove external IP** — stops phone-home while preserving VM for analysis
- **`_Required` log bucket** cannot be deleted by anyone — protects your audit trail
- **Admin Activity logs are always on** — you ALWAYS have who made API calls
- **ETD findings in SCC** are often the first detection signal — don't wait for user reports
