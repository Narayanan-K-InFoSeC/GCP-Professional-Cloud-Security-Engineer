# 11 — Secret Manager

> **Domain 3 | Weight: ~3% of total exam**  
> **Time:** ~2 hours concept + 1 hour hands-on

---

## Concepts

### 1. What is Secret Manager?

**Secret Manager** is GCP's managed service for storing sensitive data:
- API keys, database passwords, TLS certificates, SSH keys, tokens
- Versioned — each update creates a new version (old ones remain)
- Access controlled via IAM
- Audited via Cloud Logging data access logs
- Encrypted at rest (Google default or CMEK)

**Secret Manager vs. Cloud KMS:**

| Feature | Secret Manager | Cloud KMS |
|---------|---------------|-----------|
| Stores | Actual secret values | Encryption keys only |
| Use case | App credentials, passwords | Encrypting/decrypting data |
| Returns | The secret value | Encrypted/decrypted bytes |
| Versioning | Yes, immutable versions | Yes, key versions |

---

### 2. Secret Versions

- Each secret has **versions** (1, 2, 3…)
- Versions are **immutable** — you cannot change a version's value
- States: `ENABLED`, `DISABLED`, `DESTROYED`
- **`latest`** alias — always points to the latest enabled version
- Destroying a version deletes the secret material — data is gone

---

### 3. Secret Replication

| Type | Description | Use Case |
|------|-------------|---------|
| **Automatic** (Google-managed) | Google picks regions | Default, most services |
| **User-managed** | You specify regions | Data residency requirements |

User-managed replication: specify `us-central1` and `us-east1` → secret replicated to only those regions.

---

### 4. Secret Rotation

**Manual rotation:**
1. Add a new version (`gcloud secrets versions add`)
2. Update your application to use `latest` or the new version number
3. Disable/destroy the old version

**Automated rotation (Pub/Sub-triggered):**
- Set `rotation-period` on the secret
- Secret Manager publishes to a Pub/Sub topic on the rotation schedule
- A Cloud Function / Cloud Run service receives the event, generates a new secret, and adds a new version

---

### 5. CMEK for Secret Manager

Encrypt the secret payload with your own Cloud KMS key:
```bash
gcloud secrets create my-secret \
  --kms-key-name=projects/P/locations/L/keyRings/R/cryptoKeys/K
```

If the KMS key is deleted/disabled → the secret cannot be accessed.

---

### 6. Secret Manager IAM Roles

| Role | Capability |
|------|-----------|
| `roles/secretmanager.admin` | Full control — create, delete, manage, access secrets |
| `roles/secretmanager.secretVersionManager` | Create/destroy versions, cannot access values |
| `roles/secretmanager.viewer` | List secrets, see metadata — NOT values |
| `roles/secretmanager.secretAccessor` | Access (read) secret values — the key role for apps |

**Best practice:** Applications get only `secretAccessor` on specific secrets (not entire project).

---

### 7. Accessing Secrets in Applications

**Python (Application Default Credentials):**
```python
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = f"projects/PROJECT_ID/secrets/MY_SECRET/versions/latest"
response = client.access_secret_version(request={"name": name})
payload = response.payload.data.decode("UTF-8")
```

**Environment variable pattern (avoid plaintext env vars):**
Instead of `DB_PASSWORD=plaintext`, use:
```bash
DB_PASSWORD=$(gcloud secrets versions access latest --secret=db-password)
export DB_PASSWORD
```

---

### 8. Regional Secrets (2024+)

Store secrets in a specific region only — no replication to other regions:
- Required for data residency compliance
- Access latency is bound to that region
- Use for secrets containing data that must stay in a specific geography

---

## gcloud Commands

### Creating Secrets
```bash
# Create a secret with a value from a file
echo -n "my-super-secret-password" > secret-value.txt
gcloud secrets create db-password \
  --data-file=secret-value.txt \
  --replication-policy=automatic

# Create a secret from stdin
echo -n "api-key-value-here" | gcloud secrets create my-api-key \
  --data-file=- \
  --replication-policy=automatic

# Create a secret with user-managed replication (data residency)
gcloud secrets create us-only-secret \
  --replication-policy=user-managed \
  --locations=us-central1,us-east1 \
  --data-file=secret-value.txt

# Create a secret with CMEK
gcloud secrets create encrypted-secret \
  --replication-policy=user-managed \
  --locations=us-central1 \
  --kms-key-name=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key \
  --data-file=secret-value.txt

# Create with automatic rotation
gcloud secrets create rotating-secret \
  --replication-policy=automatic \
  --rotation-period=604800s \
  --next-rotation-time=2026-06-01T00:00:00Z \
  --topics=projects/PROJECT_ID/topics/secret-rotation

# Create a regional secret
gcloud secrets create regional-secret \
  --location=us-central1 \
  --data-file=secret-value.txt
```

### Managing Secret Versions
```bash
# Add a new version (rotate)
echo -n "new-password-value" | gcloud secrets versions add db-password \
  --data-file=-

# From a file
gcloud secrets versions add db-password \
  --data-file=new-secret-value.txt

# List versions
gcloud secrets versions list db-password

# Describe a version
gcloud secrets versions describe 1 --secret=db-password

# Access (read) a secret version
gcloud secrets versions access latest --secret=db-password

# Access a specific version
gcloud secrets versions access 2 --secret=db-password

# Access and decode (if base64 encoded)
gcloud secrets versions access latest --secret=db-password | base64 -d

# Disable a version (makes it inaccessible but recoverable)
gcloud secrets versions disable 1 --secret=db-password

# Re-enable a disabled version
gcloud secrets versions enable 1 --secret=db-password

# Destroy a version (permanent — deletes the value)
gcloud secrets versions destroy 1 --secret=db-password
```

### Listing and Describing Secrets
```bash
# List all secrets in a project
gcloud secrets list

# List secrets with creation time and replication
gcloud secrets list \
  --format="table(name,createTime,replication.automatic,replication.userManaged)"

# Describe a secret (metadata only — not value)
gcloud secrets describe db-password

# Filter secrets by label
gcloud secrets list \
  --filter="labels.env=production"
```

### IAM on Secrets
```bash
# Grant a service account access to a specific secret only
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Grant an entire team admin on a secret
gcloud secrets add-iam-policy-binding db-password \
  --member="group:dba-team@example.com" \
  --role="roles/secretmanager.secretVersionManager"

# View secret IAM policy
gcloud secrets get-iam-policy db-password

# Remove access
gcloud secrets remove-iam-policy-binding db-password \
  --member="serviceAccount:old-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### Deleting Secrets
```bash
# Delete a secret and ALL versions
gcloud secrets delete db-password

# Delete with force flag (no confirmation prompt)
gcloud secrets delete db-password --quiet
```

### Viewing Audit Logs for Secret Access
```bash
# View who accessed a secret
gcloud logging read \
  'resource.type="secretmanager.googleapis.com/Secret" AND 
   protoPayload.methodName="google.cloud.secretmanager.v1.SecretManagerService.AccessSecretVersion"' \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

# View all secret manager operations
gcloud logging read \
  'resource.type="secretmanager.googleapis.com/Secret"' \
  --limit=20 \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail)"
```

---

## Hands-On Practice

### Exercise 1: Secret Lifecycle — Create, Rotate, Destroy

```bash
# Step 1: Create initial secret
echo -n "initial-password-v1" | gcloud secrets create app-db-pass \
  --data-file=- \
  --replication-policy=automatic \
  --labels=env=production,app=myapp

# Step 2: Verify access
gcloud secrets versions access latest --secret=app-db-pass

# Step 3: Rotate — add a new version
echo -n "rotated-password-v2" | gcloud secrets versions add app-db-pass \
  --data-file=-

# Step 4: Verify both versions exist
gcloud secrets versions list app-db-pass

# Step 5: Access the new version
gcloud secrets versions access 2 --secret=app-db-pass

# Step 6: Disable old version
gcloud secrets versions disable 1 --secret=app-db-pass

# Step 7: Verify old version is inaccessible
gcloud secrets versions access 1 --secret=app-db-pass
# Expected: ERROR - version is disabled

# Step 8: After confirming app uses new version, destroy old
gcloud secrets versions destroy 1 --secret=app-db-pass
```

### Exercise 2: Least-Privilege Secret Access

```bash
# Create a secret
echo -n "prod-api-key-12345" | gcloud secrets create prod-api-key \
  --data-file=- \
  --replication-policy=automatic

# Create an app SA
gcloud iam service-accounts create prod-app-sa \
  --display-name="Production App SA"

# Grant access to ONLY this secret
gcloud secrets add-iam-policy-binding prod-api-key \
  --member="serviceAccount:prod-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Verify: app-sa CAN access prod-api-key
gcloud secrets versions access latest --secret=prod-api-key \
  --impersonate-service-account=prod-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Create a second secret the app should NOT access
echo -n "admin-secret" | gcloud secrets create admin-secret \
  --data-file=- \
  --replication-policy=automatic

# Verify: app-sa CANNOT access admin-secret
gcloud secrets versions access latest --secret=admin-secret \
  --impersonate-service-account=prod-app-sa@PROJECT_ID.iam.gserviceaccount.com
# Expected: ERROR - permission denied
```

### Exercise 3: Use Secret in Cloud Run

```bash
# Deploy Cloud Run service with secret mounted as env var
gcloud run deploy my-app \
  --image=gcr.io/PROJECT_ID/my-app \
  --set-secrets=DB_PASSWORD=db-password:latest \
  --service-account=app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --region=us-central1

# Mount secret as a file (alternative to env var)
gcloud run deploy my-app \
  --image=gcr.io/PROJECT_ID/my-app \
  --set-secrets=/secrets/db-pass=db-password:latest \
  --service-account=app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --region=us-central1
```

---

## Review Questions

1. A developer has `roles/secretmanager.viewer` on a project. Can they read the value of a secret? Can they list the secrets in the project?

2. Your application uses `latest` to access a secret. You add a new version and disable the old one. Do you need to restart the application?

3. What happens when you **destroy** a secret version vs. **disable** it?

4. A compliance requirement states that secrets must only be stored in `us-central1`. How do you configure Secret Manager for this?

5. How would you set up automatic secret rotation so that a new database password is generated every 30 days?

---

## Key Exam Points

- **`secretAccessor`** is the role for reading secret values — not `viewer` or `admin`
- **Versions are immutable** — you add new versions, you don't update existing ones
- **`latest`** always points to the latest **enabled** version (not just latest created)
- **Destroying a version permanently deletes the value** — cannot recover
- **Secret Manager vs KMS:** Secret Manager stores actual values; KMS stores encryption keys
- **CMEK on Secret Manager** means if you delete the KMS key, you lose access to ALL secrets encrypted with it
- **Data access logs for Secret Manager** must be explicitly enabled (they're off by default)
- **Rotation via Pub/Sub** — Secret Manager publishes a notification; your code does the actual rotation
