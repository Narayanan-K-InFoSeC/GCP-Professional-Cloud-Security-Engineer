# 02 — Service Accounts & Workload Identity Federation

> **Domain 1 | Weight: ~8% of total exam**  
> **Time:** ~3 hours concept + 2 hours hands-on

---

## Concepts

### 1. Service Account Types

| Type | Created By | Purpose | Key Format |
|------|-----------|---------|-----------|
| **User-managed** | You | App identity, custom workloads | JSON or P12 |
| **Default** | GCP automatically | App Engine, Compute Engine default SA | Not recommended |
| **Google-managed** | GCP internally | GCP services (Cloud Build SA, etc.) | Managed by Google |

**Default SA risk:** `PROJECT_NUMBER-compute@developer.gserviceaccount.com` gets `roles/editor` by default — always remove this or disable default SA.

---

### 2. Service Account Keys — The Problem

A JSON key file = permanent credential. If leaked:
- Attacker has access until key is manually deleted
- Keys don't expire automatically
- Hard to rotate at scale

**Best practice: Eliminate keys with Workload Identity Federation or short-lived tokens.**

**Org constraint to block key creation:**
```
constraints/iam.disableServiceAccountKeyCreation = true
```

---

### 3. Short-Lived Credentials (Keyless Pattern)

Instead of long-lived JSON keys, use:

**Option A — Attached service account (on GCE/GKE/Cloud Run):**
- VM/pod uses the metadata server to get tokens automatically
- No key file needed

**Option B — Service account impersonation:**
- A principal impersonates a SA to get short-lived tokens
- Requires `roles/iam.serviceAccountTokenCreator`

**Option C — Workload Identity Federation (external workloads):**
- GitHub Actions, Jenkins, AWS, Azure, on-prem → get GCP tokens without keys

---

### 4. Workload Identity Federation (WIF)

Allows **external workloads** to authenticate to GCP using their native identity — no GCP service account keys needed.

**How it works:**
```
External Workload          GCP
(GitHub Actions)    →   WIF Pool & Provider
  JWT/OIDC token    →   STS exchanges for GCP token
                    →   Token can impersonate a SA
                    →   SA permissions apply
```

**Key components:**
- **Workload Identity Pool** — logical grouping of external identities
- **Provider** — OIDC or SAML connection to the external IdP
- **Attribute mapping** — map external claims to Google attributes
- **Attribute conditions** — restrict which external identities can authenticate

**Supported external IdPs:**
- GitHub Actions (OIDC)
- GitLab CI (OIDC)
- AWS (AWS STS)
- Azure AD (OIDC)
- Kubernetes clusters (OIDC)
- HashiCorp Vault
- Any OIDC-compliant IdP

---

### 5. Workload Identity for GKE (Kubernetes)

Different from federation — this is for pods running **inside GKE** to authenticate to GCP APIs.

```
Pod in GKE Namespace     →    Kubernetes SA
Kubernetes SA            →    bound to GCP SA
Pod requests GCP API     →    uses GCP SA permissions
No key file needed!
```

**Annotation on Kubernetes SA:**
```yaml
annotations:
  iam.gke.io/gcp-service-account: gcp-sa@PROJECT_ID.iam.gserviceaccount.com
```

---

### 6. Service Account Impersonation

**Impersonation chain:**  
`User → SA A → SA B → GCP Resource`

Useful for:
- Testing what a SA can do without running code as that SA
- Least-privilege pipelines where humans briefly act as a SA

**Roles involved:**
- `roles/iam.serviceAccountTokenCreator` — create tokens as a SA
- `roles/iam.serviceAccountUser` — attach SA to a VM / Cloud Run

---

### 7. Service Account Best Practices

- One SA per application / microservice
- Never reuse SAs across workloads
- Never download JSON keys unless absolutely required
- Audit unused SAs with **IAM Recommender**
- Disable SAs that haven't been used in 90+ days
- Rotate keys regularly if keys must exist
- Use `constraints/iam.disableServiceAccountCreation` to lock org

---

## gcloud Commands

### Creating and Managing Service Accounts
```bash
# Create a service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My Application Service Account" \
  --description="Used by the payment service"

# List service accounts in a project
gcloud iam service-accounts list

# Describe a service account
gcloud iam service-accounts describe my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Disable a service account (leaves it but blocks usage)
gcloud iam service-accounts disable my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Enable a service account
gcloud iam service-accounts enable my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Delete a service account
gcloud iam service-accounts delete my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

### Granting Roles to Service Accounts
```bash
# Grant GCS read access to a service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Grant resource-level access (on a specific bucket)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

### Service Account Keys (avoid in production)
```bash
# Create a key (AVOID — use WIF or impersonation instead)
gcloud iam service-accounts keys create key.json \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# List keys for a service account
gcloud iam service-accounts keys list \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Delete a specific key
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Delete ALL user-managed keys (cleanup)
for key in $(gcloud iam service-accounts keys list \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --managed-by=user --format="value(name)"); do
  gcloud iam service-accounts keys delete $key \
    --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com --quiet
done
```

### Service Account Impersonation
```bash
# Allow a user to impersonate a SA
gcloud iam service-accounts add-iam-policy-binding my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --member="user:admin@example.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# Generate a short-lived access token as a SA
gcloud auth print-access-token \
  --impersonate-service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Run a gcloud command AS a service account
gcloud storage ls gs://my-bucket \
  --impersonate-service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Generate an ID token (for calling Cloud Run / App Engine)
gcloud auth print-identity-token \
  --impersonate-service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --audiences=https://my-cloud-run-url.run.app
```

### Workload Identity Federation — OIDC Setup (GitHub Actions)
```bash
# Step 1: Create a WIF pool
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --display-name="GitHub Actions Pool" \
  --description="For GitHub Actions CI/CD"

# Step 2: Create an OIDC provider for GitHub
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --display-name="GitHub Actions Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.actor=assertion.actor" \
  --attribute-condition="assertion.repository_owner == 'your-github-org'"

# Step 3: Get the pool full resource name
gcloud iam workload-identity-pools describe github-pool \
  --location=global \
  --format="value(name)"
# Output: projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool

# Step 4: Allow GitHub repo to impersonate a SA
gcloud iam service-accounts add-iam-policy-binding deploy-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/your-github-org/your-repo"

# Step 5: List WIF pools
gcloud iam workload-identity-pools list --location=global

# Step 6: Describe a provider
gcloud iam workload-identity-pools providers describe github-provider \
  --location=global \
  --workload-identity-pool=github-pool
```

### GKE Workload Identity
```bash
# Step 1: Enable Workload Identity on a new cluster
gcloud container clusters create my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --region=us-central1

# Step 2: Enable on an existing cluster
gcloud container clusters update my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --region=us-central1

# Step 3: Create a GCP service account
gcloud iam service-accounts create gke-app-sa \
  --display-name="GKE App Service Account"

# Step 4: Grant the GCP SA the needed permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:gke-app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Step 5: Allow the Kubernetes SA to impersonate the GCP SA
gcloud iam service-accounts add-iam-policy-binding gke-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"

# Step 6: Annotate the Kubernetes SA (done in kubectl, not gcloud)
# kubectl annotate serviceaccount KSA_NAME \
#   iam.gke.io/gcp-service-account=gke-app-sa@PROJECT_ID.iam.gserviceaccount.com \
#   --namespace=NAMESPACE
```

### Auditing Service Accounts
```bash
# Find service accounts with no recent activity (use IAM Recommender)
gcloud recommender recommendations list \
  --project=PROJECT_ID \
  --location=global \
  --recommender=google.iam.policy.Recommender \
  --format="table(name,description,stateInfo.state)"

# List all SA keys older than 90 days (custom check)
gcloud iam service-accounts list --format="value(email)" | while read SA; do
  echo "=== $SA ==="
  gcloud iam service-accounts keys list \
    --iam-account="$SA" \
    --managed-by=user \
    --format="table(name,validAfterTime,validBeforeTime)"
done

# Check SA permissions using Policy Analyzer
gcloud asset search-all-iam-policies \
  --scope=projects/PROJECT_ID \
  --query="policy:serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com"
```

---

## Hands-On Practice

### Exercise 1: Eliminate Keys with Impersonation

**Scenario:** A developer has a JSON key for a SA. Replace it with short-lived tokens.

```bash
# Step 1: Create a SA for the app
gcloud iam service-accounts create app-runner \
  --display-name="App Runner SA"

# Step 2: Give it GCS read access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:app-runner@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Step 3: Allow the developer to impersonate the SA
gcloud iam service-accounts add-iam-policy-binding \
  app-runner@PROJECT_ID.iam.gserviceaccount.com \
  --member="user:developer@example.com" \
  --role="roles/iam.serviceAccountTokenCreator"

# Step 4: Developer can now use the SA without a key file
gcloud storage ls gs://my-bucket \
  --impersonate-service-account=app-runner@PROJECT_ID.iam.gserviceaccount.com
```

### Exercise 2: GitHub Actions Keyless Auth (Full Setup)

```bash
# Run these in sequence
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
GITHUB_ORG="your-org"
GITHUB_REPO="your-repo"

# Create WIF pool
gcloud iam workload-identity-pools create "github-wif-pool" \
  --project=$PROJECT_ID \
  --location="global" \
  --display-name="GitHub WIF Pool"

# Create provider
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project=$PROJECT_ID \
  --location="global" \
  --workload-identity-pool="github-wif-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Create deploy SA
gcloud iam service-accounts create "github-deploy-sa" \
  --project=$PROJECT_ID \
  --display-name="GitHub Deploy SA"

# Allow the specific repo to impersonate the SA
gcloud iam service-accounts add-iam-policy-binding "github-deploy-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project=$PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-wif-pool/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}"

echo "WIF Provider: projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-wif-pool/providers/github-provider"
echo "SA: github-deploy-sa@${PROJECT_ID}.iam.gserviceaccount.com"
```

**GitHub Actions workflow snippet (paste in `.github/workflows/deploy.yml`):**
```yaml
jobs:
  deploy:
    permissions:
      contents: read
      id-token: write    # Required for WIF
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-wif-pool/providers/github-provider'
          service_account: 'github-deploy-sa@PROJECT_ID.iam.gserviceaccount.com'
```

### Exercise 3: Org Policy — Disable SA Key Creation

```bash
# Apply org policy to prevent key creation
gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  - << 'EOF'
constraint: constraints/iam.disableServiceAccountKeyCreation
booleanPolicy:
  enforced: true
EOF

# Verify: Try to create a key — it should fail
gcloud iam service-accounts keys create test-key.json \
  --iam-account=any-sa@PROJECT_ID.iam.gserviceaccount.com
# Expected: ERROR - policy violation
```

---

## Review Questions

1. Your application runs on a GCE VM and needs to read from BigQuery. What is the recommended authentication approach and why?

2. A security audit finds 15 service accounts with JSON keys that haven't been used in 6 months. What steps should you take?

3. Explain the difference between `roles/iam.serviceAccountUser` and `roles/iam.serviceAccountTokenCreator`.

4. A GitHub Actions pipeline needs to deploy to Cloud Run in GCP. How would you set this up without using a service account JSON key?

5. What is the difference between **GKE Workload Identity** and **Workload Identity Federation**?

---

## Key Exam Points

- **Prefer WIF over SA keys** for all external workloads
- **`roles/iam.workloadIdentityUser`** must be granted on the **GCP SA** to the external identity — not the other way around
- **GKE Workload Identity requires** cluster flag `--workload-pool` AND Kubernetes SA annotation
- **Default compute SA has `roles/editor`** — always audit and consider replacing with minimal-privilege SA
- **`constraints/iam.disableServiceAccountKeyCreation`** does NOT delete existing keys — must rotate separately
- **SA impersonation audit trail** appears in **Admin Activity logs** as `GenerateAccessToken`
