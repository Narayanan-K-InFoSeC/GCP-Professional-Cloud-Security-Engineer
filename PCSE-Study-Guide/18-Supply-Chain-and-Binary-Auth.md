# 18 — Supply Chain Security & Binary Authorization

> **Domain 4 | Weight: ~3% of total exam**  
> **Time:** ~2.5 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. Software Supply Chain Threats

**Attack vectors:**
- Compromised base images in container registries
- Malicious dependencies in package managers (npm, PyPI, Maven)
- Tampered build pipelines (CI/CD compromise)
- Unsigned or unverified container images
- Leaked secrets in source code

**GCP supply chain security stack:**
```
Source Code (Git)
     ↓ scan for secrets, vulnerabilities
Cloud Build (CI/CD)
     ↓ generates provenance (SLSA)
Artifact Registry (container/package store)
     ↓ vulnerability scanning, SBOM
Binary Authorization (admission control)
     ↓ only signed images run in GKE/Cloud Run
```

---

### 2. SLSA Framework (Supply-chain Levels for Software Artifacts)

**SLSA** defines levels of supply chain security maturity:

| Level | Requirements |
|-------|-------------|
| **SLSA 1** | Provenance exists (build metadata recorded) |
| **SLSA 2** | Hosted build (CI/CD), signed provenance |
| **SLSA 3** | Hardened build (isolated, no tampering), verified provenance |
| **SLSA 4** | Hermetic build, two-party review |

**Cloud Build** produces SLSA 1/2 provenance automatically.

---

### 3. Artifact Registry

Centralized repository for:
- Container images (Docker)
- Language packages (npm, PyPI, Maven, Go, apt, yum)
- Generic artifacts (binaries, configs)

**Security features:**
- **Vulnerability scanning** — OS and language package vulnerabilities (powered by Container Analysis)
- **SBOM generation** — Software Bill of Materials
- **Remote repositories** — proxy and cache external repos (PyPI, npm, Docker Hub) with vulnerability filtering
- **Cleanup policies** — auto-delete untagged/old images

---

### 4. Container Analysis (Vulnerability Scanning)

**On-demand scanning:**
```bash
gcloud artifacts docker images scan IMAGE_URL
```

**Continuous scanning:** Automatically scans every image pushed to Artifact Registry.

**Scan result types:**
- OS package vulnerabilities (CVEs)
- Language package vulnerabilities (npm, pip, Maven, Go)

---

### 5. Binary Authorization (BinAuthz)

**Policy** defines what images can run in GKE or Cloud Run.

**Policy evaluation modes:**
| Mode | Action on Violation |
|------|-------------------|
| `ALWAYS_ALLOW` | No enforcement (disabled) |
| `ALWAYS_DENY` | Block everything |
| `REQUIRE_ATTESTATION` | Require signed attestations |

**Attestors** = trusted parties who sign images (QA team, security scanner, build system)

**Policy components:**
- **Default admission rule** — applies to all images
- **Specific admission rules** — per-Kubernetes namespace or cluster
- **Allow patterns** — exempt specific image paths (e.g., `gcr.io/google-containers/*`)

**Enforcement modes:**
- `ENFORCED_BLOCK_AND_AUDIT_LOG` — block + log
- `DRYRUN_AUDIT_LOG_ONLY` — log only (don't block)

---

### 6. Artifact Registry Remote Repositories

Pull external packages through GCP instead of directly from the internet:
- **Upstream**: PyPI, npm, Maven Central, Docker Hub
- **Filtering**: Block packages with critical vulnerabilities
- **Caching**: Store copies for resilience and speed
- **Audit**: All downloads logged

---

### 7. Assured OSS (Open Source Software)

Google-curated and scanned OSS packages:
- Tested and validated by Google
- Signed with Google's certificates
- Available for Maven (Java) and Python
- Free through regular Artifact Registry

---

### 8. Cloud Build Security

**Provenance:** Cloud Build generates signed build provenance that records:
- Source code commit hash
- Build trigger that initiated the build
- Build steps executed
- Output artifact digests
- Build environment details

**Secret management in Cloud Build:**
- Use `secretEnv` with Secret Manager integration — never hardcode secrets in cloudbuild.yaml
- Use `availableSecrets.secretManager` blocks

**Cloud Build SA permissions:**
- Default SA: `PROJECT_NUMBER@cloudbuild.gserviceaccount.com` — often over-privileged
- Create a custom SA with minimal permissions for Cloud Build

---

## gcloud Commands

### Artifact Registry
```bash
# Create an Artifact Registry Docker repository
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production container images" \
  --immutable-tags  # Prevent tag overwriting

# Create a remote repository (proxy Docker Hub)
gcloud artifacts repositories create dockerhub-proxy \
  --repository-format=docker \
  --location=us-central1 \
  --mode=REMOTE_REPOSITORY \
  --remote-repo-config-desc="Docker Hub Proxy" \
  --remote-docker-repo=DOCKER_HUB

# List repositories
gcloud artifacts repositories list --location=us-central1

# Configure Docker to authenticate to Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Push an image
docker tag my-app:latest us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:v1.0
docker push us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:v1.0

# List images in a repository
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo

# View image tags
gcloud artifacts docker tags list \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app

# Delete an image
gcloud artifacts docker images delete \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:v1.0 \
  --delete-tags

# Grant read access to a SA
gcloud artifacts repositories add-iam-policy-binding my-docker-repo \
  --location=us-central1 \
  --member="serviceAccount:gke-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

# Set up cleanup policy (delete untagged images older than 30 days)
gcloud artifacts repositories set-cleanup-policies my-docker-repo \
  --location=us-central1 \
  --policy=cleanup-policy.json
# cleanup-policy.json:
cat > cleanup-policy.json << 'EOF'
[
  {
    "name": "delete-untagged",
    "action": {"type": "Delete"},
    "condition": {
      "tagState": "UNTAGGED",
      "olderThan": "2592000s"
    }
  }
]
EOF
```

### Vulnerability Scanning
```bash
# Scan an image in Artifact Registry (on-demand)
gcloud artifacts docker images scan \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:v1.0 \
  --format=json

# List vulnerabilities for an image
gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:v1.0 \
  --format="table(vulnerability.effectiveSeverity,vulnerability.packageIssue[0].affectedPackage,vulnerability.packageIssue[0].fixedVersion)"

# Enable continuous scanning on repository
gcloud artifacts repositories update my-docker-repo \
  --location=us-central1 \
  --update-labels=vulnerability-scanning=enabled

# View scan results for all images
gcloud container images list-tags us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app \
  --show-occurrences \
  --occurrence-filter=kind=VULNERABILITY

# Check if image has critical vulnerabilities (use in CI/CD)
CRITICAL=$(gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app:latest \
  --filter="vulnerability.effectiveSeverity=CRITICAL" \
  --format="value(vulnerability.cveId)" | wc -l)

if [ "$CRITICAL" -gt 0 ]; then
  echo "FAIL: $CRITICAL critical vulnerabilities found"
  exit 1
fi
```

### Binary Authorization
```bash
# Enable Binary Authorization API
gcloud services enable binaryauthorization.googleapis.com

# View current policy
gcloud container binauthz policy export

# Set a simple "deny all" policy (everything must be attested)
cat > strict-policy.yaml << 'EOF'
admissionWhitelistPatterns:
  - namePattern: gcr.io/google-containers/*
  - namePattern: gcr.io/gke-release/*
  - namePattern: k8s.gcr.io/*
  - namePattern: gke.gcr.io/*
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/PROJECT_ID/attestors/build-attestor
globalPolicyEvaluationMode: ENABLE
EOF
gcloud container binauthz policy import strict-policy.yaml

# Create a Container Analysis Note (required for attestor)
ACCESS_TOKEN=$(gcloud auth print-access-token)
curl -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://containeranalysis.googleapis.com/v1/projects/PROJECT_ID/notes/?noteId=build-qa-note" \
  -d '{
    "attestation": {
      "hint": {"humanReadableName": "Build QA Attestation"}
    }
  }'

# Create attestor
gcloud container binauthz attestors create build-attestor \
  --description="Build and QA sign-off attestor" \
  --attestation-authority-note=projects/PROJECT_ID/notes/build-qa-note

# Create KMS key for signing
gcloud kms keyrings create binauthz-kr --location=global
gcloud kms keys create binauthz-signing-key \
  --location=global \
  --keyring=binauthz-kr \
  --purpose=asymmetric-signing \
  --default-algorithm=rsa-sign-pkcs1-4096-sha512 \
  --protection-level=software

# Add key to attestor
KEY_VERSION=$(gcloud kms keys versions list \
  --key=binauthz-signing-key \
  --keyring=binauthz-kr \
  --location=global \
  --format="value(name)" | head -1)

gcloud container binauthz attestors public-keys add \
  --attestor=build-attestor \
  --keyversion=$KEY_VERSION

# Sign and create attestation for an image
IMAGE_DIGEST="us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/my-app@sha256:ACTUAL_DIGEST"
gcloud container binauthz attestations sign-and-create \
  --artifact-url="$IMAGE_DIGEST" \
  --attestor=build-attestor \
  --attestor-project=PROJECT_ID \
  --keyversion=$KEY_VERSION

# Verify attestation exists
gcloud container binauthz attestations list \
  --artifact-url="$IMAGE_DIGEST" \
  --attestor=build-attestor \
  --attestor-project=PROJECT_ID

# Check if a deployed pod has signed image (after setting dry-run)
gcloud container binauthz policy evaluate \
  --image-url="$IMAGE_DIGEST"
```

### Secure Cloud Build Pipeline
```yaml
# cloudbuild.yaml — secure build pipeline
steps:
  # Step 1: Build the image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$COMMIT_SHA'
      - '.'

  # Step 2: Push the image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$COMMIT_SHA'

  # Step 3: Scan for vulnerabilities
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        gcloud artifacts docker images scan \
          us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$COMMIT_SHA \
          --format=json > scan-results.json
        
        CRITICAL=$(cat scan-results.json | python3 -c "
        import json, sys
        data = json.load(sys.stdin)
        vulns = data.get('response', {}).get('vulnerabilities', [])
        critical = [v for v in vulns if v.get('effectiveSeverity') == 'CRITICAL']
        print(len(critical))
        ")
        
        if [ "$CRITICAL" -gt 0 ]; then
          echo "BLOCKED: Critical vulnerabilities found"
          exit 1
        fi
        echo "Scan passed: No critical vulnerabilities"

  # Step 4: Create Binary Authorization attestation (if scan passed)
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        IMAGE_DIGEST=$(gcloud container images describe \
          us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$COMMIT_SHA \
          --format="value(image_summary.digest)")
        
        gcloud container binauthz attestations sign-and-create \
          --artifact-url="us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app@$IMAGE_DIGEST" \
          --attestor=build-attestor \
          --attestor-project=$PROJECT_ID \
          --keyversion=projects/$PROJECT_ID/locations/global/keyRings/binauthz-kr/cryptoKeys/binauthz-signing-key/cryptoKeyVersions/1

# Use a custom SA with minimal permissions
serviceAccount: 'projects/$PROJECT_ID/serviceAccounts/cloudbuild-sa@$PROJECT_ID.iam.gserviceaccount.com'

# Access secrets securely
availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/registry-token/versions/latest
      env: 'REGISTRY_TOKEN'

options:
  logging: CLOUD_LOGGING_ONLY
```

---

## Hands-On Practice

### Exercise 1: Build a Secure Pipeline

```bash
PROJECT_ID=$(gcloud config get-value project)

# 1. Create Artifact Registry repo
gcloud artifacts repositories create secure-images \
  --repository-format=docker \
  --location=us-central1

# 2. Create Cloud Build SA with minimal permissions
gcloud iam service-accounts create cloudbuild-sa \
  --display-name="Cloud Build SA"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:cloudbuild-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:cloudbuild-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/containeranalysis.notes.attacher"

# 3. Create a sample Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
EOF

echo "flask==3.0.0" > requirements.txt
echo "print('Hello, secure world!')" > app.py

# 4. Build and push via Cloud Build
gcloud builds submit . \
  --tag=us-central1-docker.pkg.dev/${PROJECT_ID}/secure-images/myapp:v1 \
  --service-account=cloudbuild-sa@${PROJECT_ID}.iam.gserviceaccount.com

# 5. Scan the image
gcloud artifacts docker images scan \
  us-central1-docker.pkg.dev/${PROJECT_ID}/secure-images/myapp:v1 \
  --format="table(vulnerability.effectiveSeverity,vulnerability.packageIssue[0].affectedPackage)"
```

---

## Review Questions

1. What is **SLSA** and what level does Cloud Build's automatic provenance satisfy?

2. Binary Authorization is in `REQUIRE_ATTESTATION` mode. A developer deploys a pod using `kubectl apply`. The image exists in Artifact Registry but has no attestation. What happens?

3. How does using **Artifact Registry remote repositories** improve supply chain security?

4. A critical CVE is found in a base image used by 20 applications in Artifact Registry. How does GCP help you identify which images are affected?

5. Why should Cloud Build pipelines use a **custom service account** instead of the default Cloud Build SA?

---

## Key Exam Points

- **Binary Authorization** enforces at **admission time** (kubectl apply, Cloud Run deploy) — not at runtime
- **Attestation is per image DIGEST** (SHA256) not per tag — tags can point to different digests
- **Cloud Build provenance** = SLSA 1/2 — generated automatically, can be referenced in BinAuthz
- **Artifact Registry replaces Container Registry (GCR)** — use AR for new projects
- **Vulnerability scanning** requires Container Analysis API and can be on-demand or continuous
- **SBOM** (Software Bill of Materials) = list of all components in an image — generated by Artifact Registry
- **Assured OSS** provides vetted open-source packages signed by Google
- **Never use Cloud Build's default SA** in production — it has broad permissions by default
