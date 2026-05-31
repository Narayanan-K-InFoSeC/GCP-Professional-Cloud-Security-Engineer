# 04 — Identity-Aware Proxy (IAP) & BeyondCorp Zero Trust

> **Domain 1 | Weight: ~4% of total exam**  
> **Time:** ~2.5 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. What is IAP?

**Cloud Identity-Aware Proxy (IAP)** provides **application-level access control** without a VPN.

Traditional model: User → VPN → Internal network → App  
BeyondCorp model: User → IAP (identity + context verified) → App

IAP sits in front of your application and:
1. Verifies the user's **Google identity**
2. Checks **context** (device, IP, group membership)
3. Enforces **access levels** from Access Context Manager
4. Only then forwards the request to the app

---

### 2. How IAP Works

```
Browser/Client
      ↓
Google Front End (HTTPS LB)
      ↓
IAP (checks: Who are you? Are you allowed? Context OK?)
      ↓  (passes signed X-Goog-IAP-JWT-Assertion header)
Backend (App Engine / GCE backend service / Cloud Run / GKE)
```

**IAP adds headers to forwarded requests:**
- `X-Goog-Authenticated-User-Email` — verified email
- `X-Goog-IAP-JWT-Assertion` — signed JWT for backend verification

---

### 3. IAP-Secured Resources

| Resource | How IAP Protects |
|----------|-----------------|
| **App Engine** | Native integration — enable per service |
| **GCE Backend Services** | Via HTTPS Load Balancer |
| **GKE Ingress** | Via BackendConfig annotation |
| **Cloud Run** | Via IAP-secured backend service on LB |
| **On-premises apps** | Via IAP connector (TCP tunneling) |

**Critical:** Even with IAP enabled, you must ensure:
- Firewall rules **only allow traffic from the LB's IP range** — not direct VM access
- Firewall rule: `allow 130.211.0.0/22 35.191.0.0/16` for health checks + IAP traffic

---

### 4. Access Control for IAP

IAP uses IAM to control access:
- `roles/iap.httpsResourceAccessor` — grants access to HTTPS resources
- `roles/iap.tunnelResourceAccessor` — grants SSH/TCP tunneling access

These roles can be granted with **IAM conditions** for fine-grained control.

---

### 5. Access Context Manager

Defines **access levels** — conditions that must be met to be granted access.

**Access Level types:**
- **Basic** — IP ranges, device policy, regions
- **Custom** — CEL expressions for complex logic

**Device policy conditions (require Chrome Browser managed device):**
- Require screen lock
- Require OS version minimum
- Require device management (enterprise-managed)
- Require keystore present

**Access levels can be bound to:**
- IAP resources (via IAP settings)
- VPC Service Control perimeters
- IAM conditions

---

### 6. BeyondCorp Enterprise

BeyondCorp Enterprise = **Access Context Manager + IAP + Chrome Browser Cloud Management**

Zero-trust principles:
1. No trusted network — verify every request
2. Strong user identity — Google Identity + MFA
3. Device health — managed device required
4. Least-privilege access — per-app, not per-network

---

### 7. IAP TCP Tunneling (SSH without Bastion)

IAP can tunnel TCP connections (SSH, RDP) to VMs **without external IPs or bastion hosts**.

```
gcloud compute ssh my-vm --tunnel-through-iap
```

This creates an encrypted WebSocket tunnel through IAP to the VM.
Requires: `roles/iap.tunnelResourceAccessor` on the VM resource.

---

### 8. Privileged Access Manager (PAM)

**PAM** provides **just-in-time** elevated access for privileged operations.

Key concepts:
- **Entitlements** — define who can request what role, for how long, with what justification
- **Grants** — approved, time-bounded role activations
- **Approvals** — manual or automatic
- **Audit trail** — full log of requests and grants

Use case: Security engineer needs `roles/owner` for 2 hours to respond to an incident. PAM grants it, logs everything, auto-revokes after 2 hours.

---

## gcloud Commands

### IAP for App Engine
```bash
# Enable IAP on App Engine (requires LB setup first)
gcloud iap web enable \
  --resource-type=app-engine \
  --versions=default \
  --service=default

# Grant IAP access to a user
gcloud iap web add-iam-policy-binding \
  --resource-type=app-engine \
  --member="user:alice@example.com" \
  --role="roles/iap.httpsResourceAccessor"

# Grant access to a Google Group
gcloud iap web add-iam-policy-binding \
  --resource-type=app-engine \
  --member="group:team@example.com" \
  --role="roles/iap.httpsResourceAccessor"

# View IAP policy for App Engine
gcloud iap web get-iam-policy --resource-type=app-engine
```

### IAP for Backend Services (HTTPS LB)
```bash
# Enable IAP for a backend service
gcloud compute backend-services update my-backend-service \
  --global \
  --iap=enabled,oauth2-client-id=CLIENT_ID,oauth2-client-secret=CLIENT_SECRET

# Grant IAP access to backend service
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=my-backend-service \
  --member="user:alice@example.com" \
  --role="roles/iap.httpsResourceAccessor"

# Disable IAP on a backend service
gcloud compute backend-services update my-backend-service \
  --global \
  --iap=disabled
```

### IAP TCP Tunneling (SSH to VMs)
```bash
# Grant tunnel access to a user for a specific VM
gcloud compute instances add-iam-policy-binding my-vm \
  --zone=us-central1-a \
  --member="user:alice@example.com" \
  --role="roles/iap.tunnelResourceAccessor"

# SSH to VM via IAP tunnel (no external IP required)
gcloud compute ssh my-vm \
  --zone=us-central1-a \
  --tunnel-through-iap

# SSH to VM via IAP with a specific port
gcloud compute ssh my-vm \
  --zone=us-central1-a \
  --tunnel-through-iap \
  --ssh-flag="-N -L 3306:localhost:3306"  # Forward MySQL port

# Start IAP tunnel manually
gcloud compute start-iap-tunnel my-vm 22 \
  --local-host-port=localhost:2222 \
  --zone=us-central1-a
# Then: ssh -p 2222 localhost

# SCP via IAP tunnel
gcloud compute scp --tunnel-through-iap \
  my-vm:/remote/path ./local-path \
  --zone=us-central1-a
```

### Firewall Rules for IAP
```bash
# Allow IAP TCP traffic to VMs (REQUIRED for IAP tunneling to work)
gcloud compute firewall-rules create allow-iap-ssh \
  --network=default \
  --action=allow \
  --direction=ingress \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=iap-accessible

# Add the tag to a VM
gcloud compute instances add-tags my-vm \
  --tags=iap-accessible \
  --zone=us-central1-a

# Allow IAP for HTTPS load balancer health checks
gcloud compute firewall-rules create allow-lb-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --rules=tcp:80,tcp:443 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16
```

### Access Context Manager
```bash
# Create an access policy (one per org)
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="Corp Access Policy"

# List access policies
gcloud access-context-manager policies list \
  --organization=ORG_ID

# Get the policy name (needed for other commands)
POLICY_NAME=$(gcloud access-context-manager policies list \
  --organization=ORG_ID \
  --format="value(name)")

# Create a basic access level (IP-based)
gcloud access-context-manager levels create corp-ip-only \
  --policy=$POLICY_NAME \
  --title="Corporate IP Only" \
  --basic-level-spec=ip-level.yaml

# ip-level.yaml content:
cat > ip-level.yaml << 'EOF'
conditions:
  - ipSubnetworks:
      - 203.0.113.0/24
      - 198.51.100.0/24
EOF

# Create a custom access level using CEL
cat > custom-level.yaml << 'EOF'
conditions:
  - ipSubnetworks:
      - 203.0.113.0/24
  - members:
      - user:admin@example.com
combiningFunction: OR
EOF

gcloud access-context-manager levels create corp-access \
  --policy=$POLICY_NAME \
  --title="Corp Network Access" \
  --basic-level-spec=custom-level.yaml

# List access levels
gcloud access-context-manager levels list --policy=$POLICY_NAME

# Describe an access level
gcloud access-context-manager levels describe corp-ip-only \
  --policy=$POLICY_NAME

# Delete an access level
gcloud access-context-manager levels delete corp-ip-only \
  --policy=$POLICY_NAME
```

### Binding Access Levels to IAP
```bash
# Set IAP access level requirement via IAM condition
gcloud iap web add-iam-policy-binding \
  --resource-type=backend-services \
  --service=my-backend-service \
  --member="group:all-employees@example.com" \
  --role="roles/iap.httpsResourceAccessor" \
  --condition="expression=accessPolicies/POLICY_ID/accessLevels/corp-ip-only in request.auth.access_levels,title=corp-network-required"
```

### Privileged Access Manager (PAM)
```bash
# List PAM entitlements
gcloud pam entitlements list --project=PROJECT_ID --location=global

# Create a PAM entitlement
cat > entitlement.yaml << 'EOF'
privilegedAccess:
  gcpIamAccess:
    roleBindings:
      - role: roles/bigquery.admin
        conditionExpression: ""
    resource: //cloudresourcemanager.googleapis.com/projects/PROJECT_ID
    resourceType: cloudresourcemanager.googleapis.com/Project
eligibleUsers:
  - principals:
      - user:security-team@example.com
maxRequestDuration: 3600s
approvalWorkflow:
  manualApprovals:
    steps:
      - approvers:
          - principals:
              - user:manager@example.com
        approvalsNeeded: 1
EOF

gcloud pam entitlements create my-bq-admin-access \
  --project=PROJECT_ID \
  --location=global \
  --entitlement-file=entitlement.yaml

# Request a grant (as the eligible user)
gcloud pam grants create \
  --entitlement=my-bq-admin-access \
  --project=PROJECT_ID \
  --location=global \
  --requested-duration=1800s \
  --justification="Investigating data anomaly in production"

# List grants
gcloud pam grants list \
  --entitlement=my-bq-admin-access \
  --project=PROJECT_ID \
  --location=global
```

---

## Hands-On Practice

### Exercise 1: IAP-Protect a VM (No Bastion, No External IP)

```bash
PROJECT_ID=$(gcloud config get-value project)

# Step 1: Create a VM with NO external IP
gcloud compute instances create private-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --network-interface=no-address \
  --tags=iap-ssh-target

# Step 2: Create firewall rule to allow IAP tunnel source range
gcloud compute firewall-rules create allow-iap-ingress \
  --network=default \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=iap-ssh-target

# Step 3: Grant yourself IAP tunnel access
gcloud compute instances add-iam-policy-binding private-vm \
  --zone=us-central1-a \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/iap.tunnelResourceAccessor"

# Step 4: SSH through IAP
gcloud compute ssh private-vm \
  --zone=us-central1-a \
  --tunnel-through-iap

# Step 5: Verify no external IP
gcloud compute instances describe private-vm \
  --zone=us-central1-a \
  --format="get(networkInterfaces[0].accessConfigs)"
```

### Exercise 2: Create Corp IP Access Level

```bash
POLICY=$(gcloud access-context-manager policies list \
  --organization=ORG_ID --format="value(name)" | head -1)

# Get your current public IP
MY_IP=$(curl -s https://api.ipify.org)
echo "Your IP: $MY_IP"

cat > my-ip-level.yaml << EOF
conditions:
  - ipSubnetworks:
      - ${MY_IP}/32
EOF

gcloud access-context-manager levels create my-ip-level \
  --policy=$POLICY \
  --title="My IP Only" \
  --basic-level-spec=my-ip-level.yaml

echo "Access level created: accessPolicies/$(basename $POLICY)/accessLevels/my-ip-level"
```

---

## Review Questions

1. A user tries to access an IAP-protected App Engine application from their home IP. They are authenticated via Google but not in the access level (which requires a corporate IP). What happens?

2. Your VM has no external IP. How does a developer SSH into it? What firewall rule must exist?

3. What header does IAP add to forwarded requests that allows your backend to verify the user's identity?

4. What is the difference between `roles/iap.httpsResourceAccessor` and `roles/iap.tunnelResourceAccessor`?

5. PAM is configured to grant `roles/bigquery.admin` for a maximum of 1 hour. A security engineer has an ongoing incident that requires 3 hours of access. What options do they have?

---

## Key Exam Points

- **IAP requires an HTTPS Load Balancer** for GCE and GKE — cannot IAP-protect a VM directly
- **Firewall must block direct VM access** — IAP is bypassed if the VM has port 22 open to `0.0.0.0/0`
- **IAP TCP tunnel source range is `35.235.240.0/20`** — memorize this
- **Access levels are defined in Access Context Manager** — they're referenced by IAP and VPC Service Controls
- **PAM grants are time-bounded** — automatic revocation is the key feature
- **`X-Goog-IAP-JWT-Assertion`** — backend should verify this to prevent spoofing
- **IAP does NOT protect gRPC** by default — requires additional config
