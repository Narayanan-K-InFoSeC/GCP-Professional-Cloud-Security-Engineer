# 01 — IAM Fundamentals

> **Domain 1 | Weight: 27%**  
> **Time:** ~4 hours concept + 2 hours hands-on

---

## Concepts

### 1. IAM Policy Model

Every GCP resource has an IAM policy. A policy is a list of **bindings**.  
Each binding = `role` + `members` + optional `condition`.

```
Policy
└── Binding
    ├── role: roles/storage.objectViewer
    ├── members:
    │   ├── user:alice@example.com
    │   └── group:devs@example.com
    └── condition: (optional, attribute-based)
```

**Policy hierarchy — inheritance flows DOWN:**
```
Organization  (bindings here apply to everything below)
  └── Folder
        └── Project
              └── Resource  (most specific)
```
- A more permissive parent binding CANNOT be restricted by a child
- Exception: **IAM deny policies** can override allows at any level

---

### 2. Role Types

| Type | Example | Use |
|------|---------|-----|
| **Basic** | `roles/owner`, `roles/editor`, `roles/viewer` | Never use in prod — too broad |
| **Predefined** | `roles/storage.objectViewer` | Service-specific, least privilege |
| **Custom** | `projects/my-proj/roles/myRole` | When predefined is too broad or too narrow |

**Custom role stages:**
- `ALPHA` — testing, not visible to all users
- `BETA` — stable, not GA
- `GA` — generally available
- `DISABLED` — blocked, cannot be granted

---

### 3. IAM Conditions

Conditions restrict when a binding is active. They use **CEL (Common Expression Language)**.

```yaml
condition:
  title: "Weekday access only"
  expression: >
    request.time.getDayOfWeek("America/New_York") >= 1 &&
    request.time.getDayOfWeek("America/New_York") <= 5
```

**Common condition attributes:**
- `request.time` — time/date of the request
- `resource.name` — specific resource path
- `resource.type` — resource type (e.g., `storage.googleapis.com/Bucket`)
- `resource.service` — GCP service
- Tag-based: `resource.matchTag("env", "prod")`

---

### 4. IAM Deny Policies

Deny policies are **separate from allow policies** and always take precedence.

- Defined at: Organization, Folder, or Project level
- Deny bindings have: `deniedPrincipals` + `deniedPermissions` + optional `exceptionPrincipals`
- Use case: Block `setIamPolicy` across entire org except security team

```
Allow policy says: user X has roles/editor
Deny policy says:  user X is denied storage.buckets.delete
Result:            user X CANNOT delete buckets, even with editor role
```

---

### 5. allUsers vs allAuthenticatedUsers

| Member | Who | Risk |
|--------|-----|------|
| `allUsers` | Anyone on the internet | Maximum risk — public access |
| `allAuthenticatedUsers` | Any Google account | High risk — any Gmail account |

Blocked by `constraints/iam.allowedPolicyMemberDomains` org policy.

---

### 6. Policy Analyzer

Answers questions like:
- "Who has access to this Cloud Storage bucket?"
- "What resources can user X access?"
- "Which principals have `storage.objects.delete` on project Y?"

---

### 7. IAM Recommender

Automatically analyzes actual usage and suggests:
- Remove unused roles
- Downgrade over-permissive roles
- Identify service accounts not used in 90 days

---

## gcloud Commands

### Setup
```bash
# Set your project
gcloud config set project PROJECT_ID

# View current project and account
gcloud config list

# List all IAM roles available in GCP
gcloud iam roles list --format="table(name,title)"

# List predefined roles for a specific service
gcloud iam roles list --filter="name:roles/storage"
```

### Viewing IAM Policies
```bash
# View project-level IAM policy
gcloud projects get-iam-policy PROJECT_ID

# View project IAM policy in YAML format
gcloud projects get-iam-policy PROJECT_ID --format=yaml

# View org-level IAM policy
gcloud organizations get-iam-policy ORG_ID

# View folder-level IAM policy
gcloud resource-manager folders get-iam-policy FOLDER_ID

# View IAM policy for a specific GCS bucket
gcloud storage buckets get-iam-policy gs://BUCKET_NAME
```

### Granting and Revoking Roles
```bash
# Grant a predefined role to a user on a project
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Grant a role to a service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"

# Grant a role to a Google Group
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="group:devs@example.com" \
  --role="roles/logging.viewer"

# Revoke a role from a user
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Grant with a condition (time-based)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:contractor@example.com" \
  --role="roles/viewer" \
  --condition='expression=request.time < timestamp("2026-12-31T00:00:00Z"),title=temp-access'
```

### Creating Custom Roles
```bash
# List permissions for a service (e.g., storage)
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/PROJECT_ID \
  --filter="name:storage"

# Create a custom role from a YAML file
cat > custom-role.yaml << 'EOF'
title: "Custom Storage Auditor"
description: "Read-only access to storage buckets and objects"
stage: "GA"
includedPermissions:
  - storage.buckets.get
  - storage.buckets.list
  - storage.objects.get
  - storage.objects.list
EOF

gcloud iam roles create customStorageAuditor \
  --project=PROJECT_ID \
  --file=custom-role.yaml

# List custom roles in a project
gcloud iam roles list --project=PROJECT_ID

# View a custom role's permissions
gcloud iam roles describe customStorageAuditor --project=PROJECT_ID

# Update a custom role (add permissions)
gcloud iam roles update customStorageAuditor \
  --project=PROJECT_ID \
  --add-permissions=storage.buckets.getIamPolicy

# Disable a custom role
gcloud iam roles update customStorageAuditor \
  --project=PROJECT_ID \
  --stage=DISABLED

# Delete a custom role
gcloud iam roles delete customStorageAuditor --project=PROJECT_ID
```

### IAM Deny Policies
```bash
# Create a deny policy (using REST via curl — gcloud support is limited)
# First, get your org number
gcloud organizations list

# Create deny policy JSON
cat > deny-policy.json << 'EOF'
{
  "displayName": "Block bucket deletion",
  "rules": [
    {
      "denyRule": {
        "deniedPrincipals": ["principalSet://goog/public:all"],
        "deniedPermissions": ["storage.googleapis.com/buckets.delete"],
        "exceptionPrincipals": [
          "principal://goog/subject/security-admin@example.com"
        ]
      }
    }
  ]
}
EOF

# Apply via gcloud (beta)
gcloud iam policies create deny-policy-id \
  --attachment-point="cloudresourcemanager.googleapis.com/projects/PROJECT_ID" \
  --policy-file=deny-policy.json

# List deny policies on a project
gcloud iam policies list \
  --attachment-point="cloudresourcemanager.googleapis.com/projects/PROJECT_ID"
```

### Policy Analyzer
```bash
# Who has access to a specific resource?
gcloud policy-intelligence query-activity \
  --project=PROJECT_ID \
  --activity-type=serviceAccountLastAuthentication

# Test IAM permissions for a resource
gcloud storage buckets test-iam-permissions gs://BUCKET_NAME \
  storage.buckets.get storage.objects.list \
  --impersonate-service-account=SA_EMAIL
```

### Checking Effective Permissions
```bash
# Check what permissions a principal has (Policy Troubleshooter)
gcloud policy-intelligence troubleshoot-policy \
  --project=PROJECT_ID \
  --principal-email=user@example.com \
  --permission=storage.objects.get \
  --resource="//storage.googleapis.com/projects/_/buckets/BUCKET_NAME"
```

---

## Hands-On Practice

### Exercise 1: Least Privilege Role Assignment

**Goal:** Give a dev team read-only access to logs but nothing else.

```bash
# Step 1: Create a test user group (or use an existing one)
# Step 2: Find the minimal role needed
gcloud iam roles describe roles/logging.viewer

# Step 3: Bind the role to the group at PROJECT level
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="group:dev-team@example.com" \
  --role="roles/logging.viewer"

# Step 4: Verify the binding
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/logging.viewer" \
  --format="table(bindings.members)"
```

### Exercise 2: Custom Role for Audit Use Case

**Scenario:** You need a role that can list and view Cloud SQL instances but cannot modify them.

```bash
# Step 1: Find relevant permissions
gcloud iam list-testable-permissions \
  //cloudresourcemanager.googleapis.com/projects/PROJECT_ID \
  --filter="name:cloudsql" \
  --format="table(name,stage)"

# Step 2: Create the custom role
cat > sql-auditor.yaml << 'EOF'
title: "Cloud SQL Auditor"
description: "Read-only Cloud SQL access for auditors"
stage: "GA"
includedPermissions:
  - cloudsql.databases.get
  - cloudsql.databases.list
  - cloudsql.instances.get
  - cloudsql.instances.list
  - cloudsql.users.list
EOF

gcloud iam roles create sqlAuditor \
  --project=PROJECT_ID \
  --file=sql-auditor.yaml

# Step 3: Assign to a user
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:auditor@example.com" \
  --role="projects/PROJECT_ID/roles/sqlAuditor"
```

### Exercise 3: Time-Bounded Access for Contractors

**Scenario:** Contractor needs BigQuery read access until 2026-12-31.

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:contractor@example.com" \
  --role="roles/bigquery.dataViewer" \
  --condition='expression=request.time < timestamp("2026-12-31T23:59:59Z"),title=contractor-temp-access,description=Expires end of 2026'

# Verify the condition was applied
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:contractor@example.com" \
  --format="yaml"
```

### Exercise 4: Audit Who Has Editor Role

```bash
# List all principals with editor role on a project
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role=roles/editor" \
  --format="table(bindings.members)"

# List ALL role bindings in the project
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --format="table(bindings.role,bindings.members)"
```

---

## Review Questions

1. A developer needs to read objects from a GCS bucket but should never be able to delete them. An IAM deny policy denies `storage.objects.delete` for all users except the security team. The developer has `roles/storage.objectAdmin`. Can the developer delete objects? **Why?**

2. You grant `roles/viewer` at the Organization level to a service account. A project admin applies a deny policy on their project blocking `resourcemanager.projects.get` for that service account. Can the service account call `GetProject` on that project?

3. What is the difference between `allUsers` and `allAuthenticatedUsers`? Which org policy constraint blocks both?

4. You want to give a contractor access only during business hours (9am–5pm UTC, Monday–Friday). Write the CEL condition expression.

5. What stage should a custom role be in before it's safe to use in production?

---

## Key Exam Points

- **IAM deny policies override allow policies** — always
- **Policy is inherited downward** — org → folder → project → resource
- **Custom roles can only be at project or org level** — not folder level (common trick question)
- **Basic roles (Owner/Editor/Viewer) are not recommended** — use predefined or custom
- **IAM conditions require `roles/iam.conditionedBinding`** at certain levels — know the limitation
- **Policy Analyzer vs. Policy Troubleshooter** — Analyzer finds "who has access", Troubleshooter explains "why access was denied"
