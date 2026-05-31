# 05 — VPC & Firewall Security

> **Domain 2 | Weight: ~8% of total exam**  
> **Time:** ~4 hours concept + 2 hours hands-on

---

## Concepts

### 1. VPC Architecture

A **VPC (Virtual Private Cloud)** is a global, private network in GCP.

```
VPC Network (global)
├── Subnet: us-central1 (10.0.0.0/24)
├── Subnet: us-east1    (10.1.0.0/24)
└── Subnet: europe-west1 (10.2.0.0/24)
```

**Key characteristics:**
- VPCs are **global** — subnets are **regional**
- VMs in different regions of the same VPC can communicate via internal IPs
- Each VPC has its own routing table (with implied local routes for subnets)

**Default VPC:** Created automatically in each new project — insecure (pre-created firewall rules allow SSH/RDP). Block with `constraints/compute.skipDefaultNetworkCreation`.

---

### 2. Subnet Modes

| Mode | Description | Security Consideration |
|------|-------------|----------------------|
| **Auto mode** | Creates one subnet per region automatically | Not recommended for production |
| **Custom mode** | You define subnets | Preferred — full control over IP ranges |

---

### 3. Shared VPC

**Shared VPC** centralizes networking in a **host project** while allowing **service projects** to use the shared network.

```
Host Project (owns VPC, subnets, firewall rules)
├── Service Project A (deploys VMs into host project's subnet)
├── Service Project B (deploys Cloud Run into host project's subnet)
└── Service Project C (deploys GKE into host project's subnet)
```

**Benefits:**
- Centralized network management
- Network team controls firewall rules, the app teams deploy apps
- Billing stays in service projects

**Key IAM roles for Shared VPC:**
- `roles/compute.xpnAdmin` — set up Shared VPC (on host project)
- `roles/compute.networkUser` — use subnets in the host VPC (granted on the subnet or host project)
- `roles/compute.networkViewer` — view network resources

---

### 4. VPC Peering

Connects two VPCs privately (within GCP or across organizations).

- **Not transitive** — A↔B and B↔C does NOT mean A↔C
- No overlapping CIDRs allowed
- Routes are automatically exchanged
- Works across projects and organizations

---

### 5. Firewall Rules

**Direction:** `INGRESS` (inbound) or `EGRESS` (outbound)  
**Action:** `ALLOW` or `DENY`  
**Priority:** 0–65534 (lower = higher priority)

**Implied rules (cannot be deleted):**
- Priority 65535: `DENY all INGRESS`
- Priority 65534: `ALLOW all EGRESS`

**Target specification:**
| Method | Example | Security Level |
|--------|---------|---------------|
| All instances in VPC | No target tag | Broad — avoid |
| Target tags | `--target-tags=web-server` | Good |
| Target service accounts | `--target-service-accounts=sa@...` | Best — tags can be self-applied |

**Firewall rule logging:**
```bash
--enable-logging --logging-metadata=include-all
```
Logs go to Cloud Logging → can alert on rule hits.

---

### 6. Hierarchical Firewall Policies

Policies defined at **Organization or Folder level** that apply before VPC-level rules.

**Actions in hierarchical policies:**
- `allow` — permit traffic
- `deny` — block traffic  
- `goto_next` — pass evaluation to next level (VPC rules)
- `apply_security_profile_group` — send to Cloud NGFW (advanced)

```
Organization Policy (priority: evaluated first)
  └── Folder Policy
        └── VPC Firewall Rules (only reached if parent says goto_next)
              └── Network Firewall Policy (optional, regional/global)
```

---

### 7. Network Firewall Policies (VPC-Level)

Newer replacement for individual VPC firewall rules:
- **Global network firewall policies** — apply across all regions in a VPC
- **Regional network firewall policies** — apply to specific regions

Can be associated with multiple VPCs — reusable policies.

---

### 8. Firewall Insights

Built into **Network Intelligence Center**:
- **Shadow rules** — rules never matched because a higher-priority rule always matches first
- **Overly permissive rules** — rules that allow more than needed based on actual traffic
- **Deny rules with hits** — blocked traffic patterns
- **Unused rules** — rules with zero traffic in 90+ days

---

### 9. VPC Flow Logs

Captures network flow records for VMs, GKE nodes, and Shared VPC.

**Fields captured:** src/dst IP, src/dst port, protocol, bytes, packets, VM metadata

**Use cases:**
- Network forensics
- Anomaly detection
- Cost monitoring
- Compliance

**Sampling rate:** Default 50% — can set 1%–100%

---

### 10. Private Google Access

Allows VMs **without external IPs** to reach Google APIs (GCS, BigQuery, etc.) via internal routing.

Enable per subnet:
```bash
gcloud compute networks subnets update SUBNET \
  --region=REGION \
  --enable-private-ip-google-access
```

**`restricted.googleapis.com`** (199.36.153.4/30) — use with VPC Service Controls for API access.  
**`private.googleapis.com`** (199.36.153.8/30) — standard private API access.

---

## gcloud Commands

### VPC and Subnets
```bash
# Create a custom VPC
gcloud compute networks create my-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Create a subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24 \
  --enable-private-ip-google-access \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=0.5 \
  --logging-metadata=include-all

# List VPCs
gcloud compute networks list

# List subnets
gcloud compute networks subnets list --network=my-vpc

# Delete a VPC (must delete subnets first)
gcloud compute networks subnets delete my-subnet --region=us-central1
gcloud compute networks delete my-vpc
```

### Firewall Rules
```bash
# List all firewall rules in a project
gcloud compute firewall-rules list

# List firewall rules for a specific VPC
gcloud compute firewall-rules list \
  --filter="network:my-vpc" \
  --format="table(name,direction,priority,sourceRanges,targetTags,allowed)"

# Allow SSH only from IAP (no external SSH)
gcloud compute firewall-rules create allow-ssh-iap \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=ssh-allowed \
  --priority=1000 \
  --enable-logging

# Allow internal traffic only
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=all \
  --source-ranges=10.0.0.0/8 \
  --priority=1000

# Allow HTTP/HTTPS from internet to web servers
gcloud compute firewall-rules create allow-http-https \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --priority=1000

# Deny all egress (then selectively allow)
gcloud compute firewall-rules create deny-all-egress \
  --network=my-vpc \
  --direction=EGRESS \
  --action=DENY \
  --rules=all \
  --destination-ranges=0.0.0.0/0 \
  --priority=65000

# Allow egress to specific services only
gcloud compute firewall-rules create allow-egress-google-apis \
  --network=my-vpc \
  --direction=EGRESS \
  --action=ALLOW \
  --rules=tcp:443 \
  --destination-ranges=199.36.153.4/30,199.36.153.8/30 \
  --priority=1000

# SA-based firewall rule (best practice)
gcloud compute firewall-rules create allow-db-access \
  --network=my-vpc \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:5432 \
  --source-service-accounts=app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --target-service-accounts=db-sa@PROJECT_ID.iam.gserviceaccount.com \
  --priority=1000

# Delete a firewall rule
gcloud compute firewall-rules delete allow-ssh-iap

# Update firewall rule (add logging to existing rule)
gcloud compute firewall-rules update allow-http-https \
  --enable-logging
```

### Hierarchical Firewall Policies
```bash
# Create a hierarchical firewall policy at org level
gcloud compute firewall-policies create \
  --short-name=org-baseline-policy \
  --description="Organization baseline security rules" \
  --organization=ORG_ID

# Add a rule to the policy — block all ingress on 3389 (RDP)
gcloud compute firewall-policies rules create 1000 \
  --firewall-policy=org-baseline-policy \
  --organization=ORG_ID \
  --direction=INGRESS \
  --action=deny \
  --layer4-configs=tcp:3389 \
  --src-ip-ranges=0.0.0.0/0 \
  --description="Block RDP from internet"

# Add a goto_next rule (pass to VPC rules)
gcloud compute firewall-policies rules create 65534 \
  --firewall-policy=org-baseline-policy \
  --organization=ORG_ID \
  --direction=INGRESS \
  --action=goto_next \
  --layer4-configs=all \
  --src-ip-ranges=0.0.0.0/0

# Associate the policy with the org
gcloud compute firewall-policies associations create \
  --firewall-policy=org-baseline-policy \
  --organization=ORG_ID

# Associate with a folder
gcloud compute firewall-policies associations create \
  --firewall-policy=POLICY_ID \
  --folder=FOLDER_ID

# List policies
gcloud compute firewall-policies list --organization=ORG_ID

# Describe a policy and its rules
gcloud compute firewall-policies describe org-baseline-policy \
  --organization=ORG_ID
```

### Shared VPC
```bash
# Enable Shared VPC on the host project
gcloud compute shared-vpc enable HOST_PROJECT_ID

# Associate a service project with the host project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT_ID \
  --host-project=HOST_PROJECT_ID

# Grant a SA in the service project access to a specific subnet
gcloud compute networks subnets add-iam-policy-binding my-subnet \
  --region=us-central1 \
  --project=HOST_PROJECT_ID \
  --member="serviceAccount:sa@SERVICE_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"

# List associated service projects
gcloud compute shared-vpc list-associated-resources HOST_PROJECT_ID

# Disable Shared VPC
gcloud compute shared-vpc disable HOST_PROJECT_ID
```

### VPC Flow Logs
```bash
# Enable flow logs on a subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-flow-sampling=1.0 \
  --logging-aggregation-interval=interval-5-sec \
  --logging-metadata=include-all

# Disable flow logs
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --no-enable-flow-logs

# Query flow logs in BigQuery (after exporting via log sink)
# SELECT
#   jsonPayload.connection.src_ip,
#   jsonPayload.connection.dest_ip,
#   jsonPayload.connection.dest_port,
#   jsonPayload.bytes_sent,
#   jsonPayload.packets_sent
# FROM `project.dataset.compute_googleapis_com_vpc_flows_*`
# WHERE jsonPayload.connection.dest_port = 3306
# ORDER BY jsonPayload.bytes_sent DESC
# LIMIT 100
```

### Cloud NAT (Outbound Internet for Private VMs)
```bash
# Create a Cloud Router (required for NAT)
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

# Create a NAT gateway
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --enable-logging

# List NAT gateways
gcloud compute routers nats list --router=my-router --region=us-central1
```

---

## Hands-On Practice

### Exercise 1: Build a Secure 3-Tier Network

```bash
VPC="secure-vpc"
PROJECT=$(gcloud config get-value project)

# Create VPC
gcloud compute networks create $VPC --subnet-mode=custom

# Web tier subnet
gcloud compute networks subnets create web-subnet \
  --network=$VPC --region=us-central1 --range=10.0.1.0/24

# App tier subnet
gcloud compute networks subnets create app-subnet \
  --network=$VPC --region=us-central1 --range=10.0.2.0/24 \
  --enable-private-ip-google-access

# DB tier subnet
gcloud compute networks subnets create db-subnet \
  --network=$VPC --region=us-central1 --range=10.0.3.0/24 \
  --enable-private-ip-google-access

# Firewall: Allow internet → web (port 443 only)
gcloud compute firewall-rules create allow-web-ingress \
  --network=$VPC --direction=INGRESS --action=ALLOW \
  --rules=tcp:443 --source-ranges=0.0.0.0/0 \
  --target-tags=web-tier

# Firewall: Allow web → app (port 8080)
gcloud compute firewall-rules create allow-web-to-app \
  --network=$VPC --direction=INGRESS --action=ALLOW \
  --rules=tcp:8080 --source-tags=web-tier \
  --target-tags=app-tier

# Firewall: Allow app → db (port 5432)
gcloud compute firewall-rules create allow-app-to-db \
  --network=$VPC --direction=INGRESS --action=ALLOW \
  --rules=tcp:5432 --source-tags=app-tier \
  --target-tags=db-tier

# Firewall: Block all other ingress
gcloud compute firewall-rules create deny-all-ingress \
  --network=$VPC --direction=INGRESS --action=DENY \
  --rules=all --source-ranges=0.0.0.0/0 --priority=65000

# Verify rules
gcloud compute firewall-rules list --filter="network:$VPC" \
  --format="table(name,direction,action,priority,sourceRanges,sourceTags,targetTags)"
```

### Exercise 2: Investigate a Firewall with Firewall Insights

```bash
# Check firewall rule with most hits
gcloud compute firewall-rules list \
  --format="table(name,disabled,logConfig.enable)"

# Check for overly permissive rules (port 22 from 0.0.0.0/0)
gcloud compute firewall-rules list \
  --filter="allowed[].ports[]:22 AND sourceRanges[]:0.0.0.0/0" \
  --format="table(name,sourceRanges,targetTags,priority)"
```

---

## Review Questions

1. You have a VPC with VMs in `us-central1` and `europe-west1`. A VM in `us-central1` needs to talk to a VM in `europe-west1`. Is this possible without any additional configuration? What about external IPs?

2. You have three VPCs: A, B, and C. A is peered with B, and B is peered with C. Can a VM in VPC-A reach a VM in VPC-C via B? Why?

3. What is the difference between targeting firewall rules with **tags** vs **service accounts**? Which is more secure and why?

4. Your org requires NO VM to have an external IP. Which org policy constraint do you set? What happens to existing VMs with external IPs?

5. Explain the `goto_next` action in a hierarchical firewall policy. What happens if no lower-level rule matches after `goto_next`?

---

## Key Exam Points

- **Default VPC is insecure** — has SSH/RDP rules open to `0.0.0.0/0` by default
- **Hierarchical firewall evaluated BEFORE VPC rules** — org policy wins
- **SA-based firewall targeting** is more secure than tag-based (users can self-assign tags)
- **Shared VPC billing stays in service projects** — only networking is shared
- **VPC peering is non-transitive** — frequently tested
- **`goto_next` in hierarchical policies** allows VPC rules to apply
- **Flow Logs sampling** — default 50%, set higher for security monitoring
- **Private Google Access** enables API calls without external IPs — enable per subnet
