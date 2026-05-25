# 08 — GKE Security

> **Domain 2 | Weight: ~5% of total exam**  
> **Time:** ~4 hours concept + 2 hours hands-on

---

## Concepts

### 1. GKE Security Layers

```
Cluster level:    Private cluster, authorized networks, Shielded Nodes
Node level:       Shielded VMs, GKE Sandbox (gVisor), OS hardening
Network level:    Network Policies, Dataplane V2
Pod/Workload:     Workload Identity, Security Contexts, PodSecurity
Image supply:     Binary Authorization, Artifact Registry scanning
Runtime:          Container Threat Detection, VMTD
```

---

### 2. Cluster Types and API Server Access

| Type | Node External IPs | API Server | Use Case |
|------|------------------|------------|---------|
| Public cluster | Yes | Public | Dev/test |
| Public cluster + authorized networks | Yes | Public (filtered) | Better |
| Private cluster | No | Private + optional public | Production |
| Private cluster + authorized networks | No | Private (no public endpoint) | Most secure |

**Private cluster:** Nodes have only internal IPs. API server has a private endpoint. Optional public endpoint with authorized networks.

**Authorized networks:** IP allowlist for the Kubernetes API server. Even if the endpoint is public, only listed CIDRs can reach it.

---

### 3. Workload Identity (GKE)

Replaces service account key files for pods to authenticate to GCP.

**Setup:**
1. Enable workload identity on cluster: `--workload-pool=PROJECT_ID.svc.id.goog`
2. Create GCP SA + grant it needed permissions
3. Create Kubernetes SA in the pod's namespace
4. Bind Kubernetes SA → GCP SA (IAM binding)
5. Annotate Kubernetes SA with GCP SA email
6. Pod uses Kubernetes SA → metadata server gives GCP token automatically

**No JSON key files needed in pods!**

---

### 4. Network Policies

Kubernetes **NetworkPolicy** controls pod-to-pod and pod-to-external traffic.

**Requires:** A CNI plugin that enforces network policies.
- **Dataplane V2 (eBPF)** — built into GKE, no extra install needed
- **Calico** — alternative CNI

```yaml
# Example: Deny all ingress to a namespace except from same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```yaml
# Allow only specific pods to reach the database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: database
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: app
      ports:
        - port: 5432
```

---

### 5. Binary Authorization

Enforces that only **cryptographically signed container images** run in your GKE cluster.

**Components:**
- **Attestors** — trusted entities that sign images (e.g., "QA team", "Security scanner")
- **Attestations** — cryptographic signatures that an attestor has approved an image
- **Policy** — defines which attestations are required for which images
- **Binary Authorization admission controller** — enforces policy at deploy time

**Policy modes:**
- `ALWAYS_ALLOW` — allow everything (equivalent to disabled)
- `ALWAYS_DENY` — block everything
- `REQUIRE_ATTESTATION` — require specific attestor signatures
- Whitelist patterns — allow specific registries/images without attestation

**Dry-run mode:** Policy is evaluated but not enforced — logs violations.

---

### 6. GKE Sandbox (gVisor)

Provides an additional **kernel isolation layer** between the container and host OS.

- Uses **gVisor** (user-space kernel written by Google)
- Intercepts system calls before they reach the host kernel
- Ideal for: running untrusted code, multi-tenant clusters, third-party workloads
- Trade-off: ~10-15% performance overhead

**Enable per node pool** with:
```bash
--sandbox type=gvisor
```

---

### 7. Shielded GKE Nodes

Protects the node's boot process:
- **Secure Boot** — verifies OS/software is signed by Google
- **vTPM (virtual Trusted Platform Module)** — provides attestation
- **Integrity Monitoring** — detects changes to node firmware/OS

Enable with `--shielded-secure-boot --shielded-integrity-monitoring`.

---

### 8. Confidential GKE Nodes

Nodes backed by **Confidential VMs** (AMD SEV):
- RAM is encrypted in-use
- Even Google cannot read VM memory
- Use for sensitive workloads

---

### 9. Pod Security

**Pod Security Standards (PSS)** — Kubernetes-native (replaced PodSecurityPolicy in K8s 1.25):
- `privileged` — no restrictions
- `baseline` — prevent known privilege escalations
- `restricted` — hardened, follow security best practices

**Pod Security Admission (PSA)** — enforcer of PSS:
- `enforce` — reject violating pods
- `warn` — allow but warn
- `audit` — allow but log

**Security Contexts** — per-pod/container security settings:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

---

### 10. Container Threat Detection (CTD)

Part of **Security Command Center Premium**. Detects runtime attacks in GKE:

| Finding | Description |
|---------|-------------|
| `Added Library Loaded` | Unexpected library loaded in container |
| `Reverse Shell` | Container opening a shell to external host |
| `Unexpected Child Shell` | Shell spawned from non-shell process |
| `Malicious Script Executed` | Known malicious script signature |
| `Privilege Escalation` | Container gaining elevated privileges |

---

## gcloud Commands

### Cluster Creation
```bash
# Create a secure private cluster
gcloud container clusters create secure-cluster \
  --region=us-central1 \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-ip-alias \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --enable-shielded-nodes \
  --shielded-secure-boot \
  --shielded-integrity-monitoring \
  --enable-dataplane-v2 \
  --enable-network-policy \
  --master-authorized-networks=203.0.113.0/24 \
  --enable-master-authorized-networks \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-autorepair \
  --enable-autoupgrade

# Create with Confidential Nodes
gcloud container clusters create confidential-cluster \
  --region=us-central1 \
  --machine-type=n2d-standard-4 \
  --enable-confidential-nodes \
  --workload-pool=PROJECT_ID.svc.id.goog

# Get credentials for kubectl
gcloud container clusters get-credentials secure-cluster \
  --region=us-central1
```

### GKE Workload Identity
```bash
PROJECT_ID=$(gcloud config get-value project)
NAMESPACE="production"
KSA_NAME="app-ksa"
GSA_NAME="gke-app-gsa"

# Create GCP Service Account
gcloud iam service-accounts create $GSA_NAME \
  --display-name="GKE App GSA"

# Grant permissions to the GCP SA
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Allow Kubernetes SA to impersonate GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  "${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]" \
  --role="roles/iam.workloadIdentityUser"

# Create Kubernetes namespace and SA (kubectl commands)
# kubectl create namespace $NAMESPACE
# kubectl create serviceaccount $KSA_NAME -n $NAMESPACE

# Annotate the Kubernetes SA (kubectl command)
# kubectl annotate serviceaccount $KSA_NAME \
#   iam.gke.io/gcp-service-account=${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
#   -n $NAMESPACE

# Verify Workload Identity is enabled
gcloud container clusters describe secure-cluster \
  --region=us-central1 \
  --format="get(workloadIdentityConfig)"
```

### Binary Authorization
```bash
# Enable Binary Authorization on a cluster
gcloud container clusters update secure-cluster \
  --region=us-central1 \
  --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE

# Get the Binary Authorization policy
gcloud container binauthz policy export > binauthz-policy.yaml

# View current policy
cat binauthz-policy.yaml

# Create an attestor
gcloud container binauthz attestors create qa-attestor \
  --description="QA team attestor" \
  --attestation-authority-note=projects/PROJECT_ID/notes/qa-note

# Create a note (requires Container Analysis API)
curl -vvv -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://containeranalysis.googleapis.com/v1/projects/PROJECT_ID/notes/?noteId=qa-note" \
  --data-binary @- << 'EOF'
{
  "attestation": {
    "hint": {
      "humanReadableName": "QA Attestation Note"
    }
  }
}
EOF

# Create a KMS key for signing
gcloud kms keyrings create binauthz-keyring \
  --location=global

gcloud kms keys create qa-signing-key \
  --location=global \
  --keyring=binauthz-keyring \
  --purpose=asymmetric-signing \
  --default-algorithm=ec-sign-p256-sha256

# Add the key to the attestor
KEY_VERSION=$(gcloud kms keys versions list \
  --key=qa-signing-key \
  --keyring=binauthz-keyring \
  --location=global \
  --format="value(name)" | head -1)

gcloud container binauthz attestors public-keys add \
  --attestor=qa-attestor \
  --keyversion=$KEY_VERSION

# Export public key (for verification)
gcloud container binauthz attestors describe qa-attestor

# Set policy to require attestation
cat > binauthz-policy.yaml << 'EOF'
admissionWhitelistPatterns:
  - namePattern: gcr.io/google-containers/*
  - namePattern: gcr.io/gke-release/*
  - namePattern: k8s.gcr.io/*
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
    - projects/PROJECT_ID/attestors/qa-attestor
globalPolicyEvaluationMode: ENABLE
EOF

gcloud container binauthz policy import binauthz-policy.yaml

# Sign an image (create attestation)
IMAGE_URL="us-central1-docker.pkg.dev/PROJECT_ID/my-repo/my-app@sha256:DIGEST"
gcloud container binauthz attestations sign-and-create \
  --artifact-url=$IMAGE_URL \
  --attestor=qa-attestor \
  --attestor-project=PROJECT_ID \
  --keyversion=$KEY_VERSION

# Verify an image has attestation
gcloud container binauthz attestations list \
  --artifact-url=$IMAGE_URL \
  --attestor=qa-attestor \
  --attestor-project=PROJECT_ID
```

### Network Policies (kubectl)
```bash
# Apply a default deny-all ingress policy to a namespace
# Save to file and apply:
cat > deny-all.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF
# kubectl apply -f deny-all.yaml

# Allow ingress from specific pods
cat > allow-app.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
EOF
# kubectl apply -f allow-app.yaml
```

### GKE Sandbox (gVisor)
```bash
# Create a node pool with gVisor sandbox
gcloud container node-pools create sandbox-pool \
  --cluster=secure-cluster \
  --region=us-central1 \
  --machine-type=e2-standard-4 \
  --num-nodes=2 \
  --sandbox type=gvisor

# Verify node pool sandbox
gcloud container node-pools describe sandbox-pool \
  --cluster=secure-cluster \
  --region=us-central1 \
  --format="get(config.sandboxConfig)"

# Run a pod on the gVisor node pool (kubectl)
# Add runtimeClassName to pod spec:
cat > sandboxed-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-app
spec:
  runtimeClassName: gvisor
  containers:
    - name: app
      image: nginx:latest
  nodeSelector:
    cloud.google.com/gke-sandbox-on: "true"
EOF
# kubectl apply -f sandboxed-pod.yaml
```

### Checking Cluster Security Posture
```bash
# View cluster security posture
gcloud container clusters describe secure-cluster \
  --region=us-central1 \
  --format="yaml(securityPostureConfig)"

# Enable workload vulnerability scanning
gcloud container clusters update secure-cluster \
  --region=us-central1 \
  --workload-vulnerability-scanning=STANDARD

# View node pool configuration
gcloud container node-pools describe default-pool \
  --cluster=secure-cluster \
  --region=us-central1 \
  --format="yaml(config.shieldedInstanceConfig)"
```

---

## Hands-On Practice

### Exercise 1: Enable Workload Identity on Existing Cluster

```bash
PROJECT_ID=$(gcloud config get-value project)

# Enable on existing cluster
gcloud container clusters update my-cluster \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --region=us-central1

# Update existing node pools (required after enabling on cluster)
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --region=us-central1 \
  --workload-metadata=GKE_METADATA

echo "Workload Identity enabled. Now annotate your Kubernetes SAs."
```

### Exercise 2: Set Binary Authorization to Dry-Run First

```bash
# Set policy to DRYRUN_AUDIT_LOG_ONLY — logs violations, doesn't block
cat > binauthz-dryrun.yaml << 'EOF'
defaultAdmissionRule:
  evaluationMode: ALWAYS_ALLOW
  enforcementMode: DRYRUN_AUDIT_LOG_ONLY
globalPolicyEvaluationMode: ENABLE
EOF

gcloud container binauthz policy import binauthz-dryrun.yaml

echo "Policy in dry-run. Monitor SCC and Cloud Logging for violations before enforcing."
```

---

## Review Questions

1. A pod needs to access Cloud Storage. What is the recommended authentication approach for GKE? Walk through the setup steps.

2. What is the difference between **GKE Sandbox (gVisor)** and **Confidential GKE Nodes**? What threat does each protect against?

3. Binary Authorization is set to `REQUIRE_ATTESTATION` for the `qa-attestor`. A developer deploys a new image that hasn't been attested. What happens?

4. Your cluster has Dataplane V2 enabled. You apply a NetworkPolicy to deny all ingress to the `payments` namespace. Does this affect existing established connections?

5. What is the difference between **authorized networks** and a **private cluster**? Can you have both?

---

## Key Exam Points

- **Workload Identity requires** `--workload-pool` on the cluster AND `GKE_METADATA` on node pools AND annotation on Kubernetes SA
- **Binary Authorization is per-cluster** — policies are project-wide but enforcement is cluster-level
- **gVisor sandboxes system calls** — doesn't encrypt memory (that's Confidential Nodes)
- **Confidential GKE Nodes** = encrypted RAM via AMD SEV — requires `n2d` machine types
- **Shielded GKE Nodes** = Secure Boot + vTPM + Integrity Monitoring — different from Confidential
- **Dataplane V2 (eBPF)** must be enabled at cluster creation — cannot be enabled on existing cluster
- **NetworkPolicy** only works if the CNI supports it — Dataplane V2 and Calico both do
- **CTD findings** appear in **Security Command Center** — requires SCC Premium
