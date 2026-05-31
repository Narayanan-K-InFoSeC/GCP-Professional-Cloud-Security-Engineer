# 03 — Organization Policy & Resource Hierarchy

> **Domain 1 | Weight: ~5% of total exam**  
> **Time:** ~3 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. Resource Hierarchy

```
Organization (root — tied to your domain: example.com)
├── Folder: Production
│   ├── Folder: Finance
│   │   └── Project: finance-prod-001
│   └── Folder: Engineering
│       └── Project: eng-prod-001
├── Folder: Non-Production
│   └── Project: dev-001
└── Project: billing-management
```

**Why it matters for security:**
- IAM policies cascade down — grant at org level = access everywhere
- Org policies cascade down — restrict at org level = restrict everywhere
- Folders = environment boundaries (prod vs. dev), team boundaries, compliance boundaries
- Projects = billing unit, API enablement unit, quota unit

---

### 2. Organization Policy Service

Defines **what can be done**, not **who can do it** (that's IAM).

**Constraint types:**

| Type | Example | Values |
|------|---------|--------|
| **Boolean** | `constraints/compute.requireShieldedVm` | `enforced: true/false` |
| **List** | `constraints/gcp.resourceLocations` | `allowedValues` or `deniedValues` |

**Policy inheritance:**
- Child inherits parent's policy
- Child can **restrict further** but cannot be **more permissive** than parent (unless parent allows override)
- `restore_default` — resets to GCP default for that constraint

---

### 3. Critical Org Constraints (Exam Favorites)

| Constraint | What It Does |
|-----------|-------------|
| `constraints/compute.requireShieldedVm` | All new VMs must use Shielded VM |
| `constraints/compute.vmExternalIpAccess` | Restrict which VMs can have external IPs |
| `constraints/compute.skipDefaultNetworkCreation` | Don't create default VPC in new projects |
| `constraints/compute.restrictCloudNATUsage` | Control where Cloud NAT can be used |
| `constraints/compute.restrictLoadBalancerCreationForTypes` | Block creating specific LB types |
| `constraints/storage.uniformBucketLevelAccess` | Enforce IAM-only access on GCS |
| `constraints/storage.publicAccessPrevention` | Block `allUsers`/`allAuthenticatedUsers` on GCS |
| `constraints/iam.allowedPolicyMemberDomains` | Only allow identities from specific domains |
| `constraints/iam.disableServiceAccountKeyCreation` | Block creating SA JSON keys |
| `constraints/iam.disableServiceAccountCreation` | Block creating service accounts |
| `constraints/gcp.resourceLocations` | Restrict which regions resources can be created in |
| `constraints/cloudfunctions.allowedIngressSettings` | Control Cloud Functions ingress |
| `constraints/run.allowedIngress` | Cloud Run ingress restriction |
| `constraints/compute.disableInternetGroupManagerCreation` | Block public managed instance groups |

---

### 4. Custom Org Policy Constraints (2024+ Feature)

You can write your **own constraints** using CEL expressions on resource fields.

Use case: "No Cloud SQL instances without private IP" or "All Cloud Run services must have max instances ≤ 100"

```yaml
name: organizations/ORG_ID/customConstraints/custom.cloudSqlRequirePrivateIp
methodTypes: [CREATE, UPDATE]
condition: "resource.settings.ipConfiguration.privateNetwork != ''"
actionType: ALLOW
displayName: "Cloud SQL must use Private IP"
```

---

### 5. Policy Inheritance — Merge vs. Replace

When a parent sets a list constraint and a child also sets it:
- **Default (merge):** Child's list is combined with parent's list
- **Replace:** Child completely replaces parent's list (only if parent allows override)

For boolean constraints:
- If parent enforces `true`, child cannot set `false`
- If parent does NOT enforce, child can enforce `true`

---

## gcloud Commands

### Viewing Hierarchy
```bash
# List all organizations your account can see
gcloud organizations list

# List folders under an organization
gcloud resource-manager folders list --organization=ORG_ID

# List folders under a folder
gcloud resource-manager folders list --folder=FOLDER_ID

# List projects under an organization
gcloud projects list --filter="parent.id=ORG_ID"

# Get organization details
gcloud organizations describe ORG_ID

# Get folder details
gcloud resource-manager folders describe FOLDER_ID
```

### Creating Folders and Projects
```bash
# Create a folder under org
gcloud resource-manager folders create \
  --display-name="Production" \
  --organization=ORG_ID

# Create a folder under another folder
gcloud resource-manager folders create \
  --display-name="Finance" \
  --folder=FOLDER_ID

# Create a project under a folder
gcloud projects create finance-prod-001 \
  --folder=FOLDER_ID \
  --name="Finance Production"

# Move a project to a different folder
gcloud beta projects move PROJECT_ID \
  --folder=NEW_FOLDER_ID
```

### Viewing Org Policies
```bash
# View effective org policy on a project
gcloud resource-manager org-policies describe \
  constraints/compute.requireShieldedVm \
  --project=PROJECT_ID

# View org policy at org level
gcloud resource-manager org-policies describe \
  constraints/storage.publicAccessPrevention \
  --organization=ORG_ID

# List ALL org policies set on a project
gcloud resource-manager org-policies list --project=PROJECT_ID

# List ALL org policies set at org level
gcloud resource-manager org-policies list --organization=ORG_ID
```

### Setting Boolean Constraints
```bash
# Enforce Shielded VM at org level
gcloud resource-manager org-policies enable-enforce \
  constraints/compute.requireShieldedVm \
  --organization=ORG_ID

# Enforce at folder level
gcloud resource-manager org-policies enable-enforce \
  constraints/compute.requireShieldedVm \
  --folder=FOLDER_ID

# Disable enforcement (restore default behavior)
gcloud resource-manager org-policies disable-enforce \
  constraints/compute.requireShieldedVm \
  --project=PROJECT_ID

# Restore default (remove project-level override)
gcloud resource-manager org-policies delete \
  constraints/compute.requireShieldedVm \
  --project=PROJECT_ID
```

### Setting List Constraints
```bash
# Restrict resource locations to US regions only
cat > location-policy.yaml << 'EOF'
constraint: constraints/gcp.resourceLocations
listPolicy:
  allowedValues:
    - in:us-locations
EOF

gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  location-policy.yaml

# Allow only specific regions
cat > region-policy.yaml << 'EOF'
constraint: constraints/gcp.resourceLocations
listPolicy:
  allowedValues:
    - us-central1
    - us-east1
    - us-west1
EOF

gcloud resource-manager org-policies set-policy \
  --project=PROJECT_ID \
  region-policy.yaml

# Restrict external IPs — deny all (no VMs with external IPs)
cat > no-external-ip.yaml << 'EOF'
constraint: constraints/compute.vmExternalIpAccess
listPolicy:
  allValues: DENY
EOF

gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  no-external-ip.yaml

# Allow external IP only for specific VMs
cat > allow-specific-vms.yaml << 'EOF'
constraint: constraints/compute.vmExternalIpAccess
listPolicy:
  allowedValues:
    - projects/PROJECT_ID/zones/us-central1-a/instances/bastion-vm
EOF

gcloud resource-manager org-policies set-policy \
  --project=PROJECT_ID \
  allow-specific-vms.yaml

# Restrict member domains — only allow your company's domain
cat > domain-restrict.yaml << 'EOF'
constraint: constraints/iam.allowedPolicyMemberDomains
listPolicy:
  allowedValues:
    - C0xxxxxxx   # Your Google Workspace customer ID
EOF

gcloud resource-manager org-policies set-policy \
  --organization=ORG_ID \
  domain-restrict.yaml

# Get your Workspace customer ID
gcloud organizations describe ORG_ID --format="value(owner.directoryCustomerId)"
```

### Custom Org Policy Constraints (New)
```bash
# Create a custom constraint
cat > custom-constraint.yaml << 'EOF'
name: organizations/ORG_ID/customConstraints/custom.cloudSqlNoPublicIp
resourceTypes:
  - sqladmin.googleapis.com/Instance
methodTypes:
  - CREATE
  - UPDATE
condition: "resource.settings.ipConfiguration.ipv4Enabled == false"
actionType: ALLOW
displayName: "Cloud SQL must not have public IP"
description: "All Cloud SQL instances must use private IP only"
EOF

gcloud org-policies set-custom-constraint custom-constraint.yaml

# Now enforce it as an org policy
cat > enforce-custom.yaml << 'EOF'
name: projects/PROJECT_ID/policies/custom.cloudSqlNoPublicIp
spec:
  rules:
  - enforce: true
EOF

gcloud org-policies set-policy enforce-custom.yaml
```

### Storage-Specific Policies
```bash
# Enforce uniform bucket-level access across org
gcloud resource-manager org-policies enable-enforce \
  constraints/storage.uniformBucketLevelAccess \
  --organization=ORG_ID

# Prevent public access to GCS buckets
gcloud resource-manager org-policies enable-enforce \
  constraints/storage.publicAccessPrevention \
  --organization=ORG_ID

# Verify on a specific project
gcloud resource-manager org-policies describe \
  constraints/storage.publicAccessPrevention \
  --project=PROJECT_ID --effective
```

---

## Hands-On Practice

### Exercise 1: Secure Landing Zone Setup

**Scenario:** New org needs basic security baseline.

```bash
ORG_ID="your-org-id"

# 1. Block public GCS buckets
gcloud resource-manager org-policies enable-enforce \
  constraints/storage.publicAccessPrevention \
  --organization=$ORG_ID

# 2. Block SA key creation
gcloud resource-manager org-policies enable-enforce \
  constraints/iam.disableServiceAccountKeyCreation \
  --organization=$ORG_ID

# 3. Require Shielded VMs
gcloud resource-manager org-policies enable-enforce \
  constraints/compute.requireShieldedVm \
  --organization=$ORG_ID

# 4. Skip default VPC in new projects
gcloud resource-manager org-policies enable-enforce \
  constraints/compute.skipDefaultNetworkCreation \
  --organization=$ORG_ID

# 5. Lock resources to US regions
cat > us-only-locations.yaml << 'EOF'
constraint: constraints/gcp.resourceLocations
listPolicy:
  allowedValues:
    - in:us-locations
EOF
gcloud resource-manager org-policies set-policy \
  --organization=$ORG_ID \
  us-only-locations.yaml

# 6. Verify all policies
gcloud resource-manager org-policies list --organization=$ORG_ID
```

### Exercise 2: Folder-Level Override

**Scenario:** Prod folder needs extra restriction but Dev folder needs relaxed external IP policy.

```bash
PROD_FOLDER_ID="prod-folder-id"
DEV_FOLDER_ID="dev-folder-id"

# Prod: No external IPs at all
cat > prod-no-external-ip.yaml << 'EOF'
constraint: constraints/compute.vmExternalIpAccess
listPolicy:
  allValues: DENY
EOF
gcloud resource-manager org-policies set-policy \
  --folder=$PROD_FOLDER_ID \
  prod-no-external-ip.yaml

# Dev: Allow external IPs (org has DENY, dev overrides if inherit is not enforced)
# Note: If org policy is set to DENY with allValues, a child CANNOT override it
# Only works if parent uses allowedValues/deniedValues list, not allValues

# Check what the effective policy is on a project in prod folder
gcloud resource-manager org-policies describe \
  constraints/compute.vmExternalIpAccess \
  --project=PROJECT_IN_PROD_FOLDER \
  --effective
```

### Exercise 3: Audit Policy Violations

```bash
# Check if a GCS bucket violates the publicAccessPrevention policy
gsutil iam get gs://bucket-name | grep "allUsers\|allAuthenticatedUsers"

# Check a VM for external IP despite policy
gcloud compute instances list \
  --filter="networkInterfaces.accessConfigs.natIP:*" \
  --format="table(name,zone,networkInterfaces[0].accessConfigs[0].natIP)"

# Check for projects without org policies
gcloud projects list --format="value(projectId)" | while read proj; do
  count=$(gcloud resource-manager org-policies list --project=$proj 2>/dev/null | wc -l)
  echo "$proj: $count policies"
done
```

---

## Review Questions

1. An org policy at the Organization level sets `constraints/compute.vmExternalIpAccess` with `allValues: DENY`. A project admin tries to override it with `allowedValues` for their project. What happens?

2. What is the difference between `constraints/storage.uniformBucketLevelAccess` and `constraints/storage.publicAccessPrevention`? When would you use each?

3. You want to block all resources from being created outside of `us-central1` and `us-east1`. Which constraint do you use and what are the allowed values?

4. A new engineer creates a project under the "Engineering" folder. The org-level policy requires Shielded VMs. The Engineering folder has no policy set. Will the project inherit the org-level Shielded VM requirement?

5. What is a **custom org policy constraint** and when would you create one instead of using a predefined constraint?

---

## Key Exam Points

- **Org policies restrict configurations** — IAM restricts who can act
- **Children inherit parent policies** — cannot be more permissive (by default)
- **`constraints/iam.allowedPolicyMemberDomains`** requires the Google Workspace **Customer ID** (like `C0xxxxxxx`), not the domain string
- **`allValues: DENY`** is stronger than `deniedValues` — `allValues: DENY` with no exceptions cannot be overridden by children
- **Custom constraints** use CEL on resource metadata — not IAM
- **`--effective` flag** on org-policies commands shows the inherited+effective policy
- **Folders vs. Projects** — folders organize, projects are the billing/API boundary
- **Organization node** is automatically created when you have a Google Workspace or Cloud Identity domain
