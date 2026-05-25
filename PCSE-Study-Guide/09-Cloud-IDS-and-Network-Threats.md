# 09 — Cloud IDS & Network Threat Detection

> **Domain 2 | Weight: ~3% of total exam**  
> **Time:** ~2 hours concept + 1 hour hands-on

---

## Concepts

### 1. Cloud IDS Overview

**Cloud IDS (Intrusion Detection System)** provides network-based threat detection powered by **Palo Alto Networks** threat intelligence.

```
Your VPC Traffic
      ↓
Packet Mirroring (copies traffic)
      ↓
Cloud IDS (Palo Alto engine analyzes mirrored packets)
      ↓
Findings → Cloud Logging → Security Command Center
```

**Key facts:**
- Cloud IDS **detects** threats — it does NOT block them
- Works by **mirroring** traffic to the IDS endpoint
- Uses Palo Alto Networks signatures (updated automatically)
- Managed service — no infrastructure to maintain

---

### 2. Threat Severity Levels

| Severity | Examples |
|----------|---------|
| **Critical** | Active exploit, worm, ransomware C2 |
| **High** | Known malware, network scanning tools |
| **Medium** | Policy violations, suspicious activity |
| **Low** | Informational, port scans |
| **Informational** | Normal traffic matching a signature |

---

### 3. Cloud IDS Endpoints

An **IDS Endpoint** is the Palo Alto analysis engine deployed in your VPC zone.

- Created per **zone** (not region)
- Traffic is mirrored from subnets to the endpoint
- One endpoint can handle traffic from multiple subnets/VPCs
- Minimum 1 Gbps, scales automatically

---

### 4. Packet Mirroring

**Packet Mirroring** copies traffic from specified GCE instances/subnets and sends it to a collector (Cloud IDS endpoint, or your own SIEM).

**Mirroring policy defines:**
- **Source:** Subnet, instance tags, or specific instances
- **Traffic filter:** All, ingress only, egress only
- **Destination:** Internal load balancer pointing to the IDS endpoint

---

### 5. VPC Flow Logs vs. Cloud IDS

| Feature | VPC Flow Logs | Cloud IDS |
|---------|--------------|-----------|
| Data | Metadata (src/dst IP, port, bytes) | Full packet analysis |
| Threat detection | No | Yes |
| Protocol analysis | No | Deep packet inspection |
| Signatures | No | Palo Alto threat signatures |
| Cost | Per GB logged | Per endpoint-hour + per GB inspected |
| Use case | Traffic auditing, anomaly detection | Threat detection, IDS/IPS pattern matching |

---

### 6. Response to IDS Findings

Since Cloud IDS only detects (doesn't block), you need a response workflow:

```
Cloud IDS Finding
      ↓
Cloud Logging
      ↓
Log-based Alert → Pub/Sub
      ↓
Cloud Functions / Workflows
      ↓
Automatically add IP to Cloud Armor deny rule
OR
Create firewall rule to block traffic
OR
Page on-call team via PagerDuty
```

---

### 7. Firewall Rules for IDS Architecture

For Cloud IDS to receive mirrored traffic, the VPC must allow:
- Traffic from the subnet being mirrored to the IDS endpoint
- Internal load balancer to receive mirrored packets

---

## gcloud Commands

### Setting Up Cloud IDS
```bash
# Step 1: Enable required APIs
gcloud services enable ids.googleapis.com
gcloud services enable servicedirectory.googleapis.com
gcloud services enable networksecurity.googleapis.com

# Step 2: Create an IDS endpoint
gcloud ids endpoints create my-ids-endpoint \
  --network=my-vpc \
  --zone=us-central1-a \
  --severity=MEDIUM \
  --description="IDS endpoint for production VPC"
# Note: This takes 10-20 minutes to provision

# List IDS endpoints
gcloud ids endpoints list --zone=us-central1-a

# Describe endpoint (get the endpoint forwarding rule for packet mirroring)
gcloud ids endpoints describe my-ids-endpoint \
  --zone=us-central1-a

# Get the endpoint URL (needed for packet mirroring)
ENDPOINT_FWD_RULE=$(gcloud ids endpoints describe my-ids-endpoint \
  --zone=us-central1-a \
  --format="value(endpointForwardingRule)")
echo "Endpoint forwarding rule: $ENDPOINT_FWD_RULE"
```

### Packet Mirroring Setup
```bash
# Step 1: Create an internal TCP/UDP load balancer for mirrored traffic
# (This receives the mirrored traffic and forwards to IDS endpoint)
# Note: Cloud IDS auto-creates this — use the endpoint forwarding rule

# Step 2: Create a packet mirroring policy for a subnet
gcloud compute packet-mirrorings create my-mirroring \
  --region=us-central1 \
  --network=my-vpc \
  --mirrored-subnets=my-subnet \
  --collector-ilb=$ENDPOINT_FWD_RULE \
  --description="Mirror all subnet traffic to Cloud IDS"

# Create packet mirroring for specific instance tags
gcloud compute packet-mirrorings create tag-based-mirroring \
  --region=us-central1 \
  --network=my-vpc \
  --mirrored-tags=mirror-this \
  --collector-ilb=$ENDPOINT_FWD_RULE \
  --filter-direction=BOTH

# Create with traffic filter (only mirror ingress)
gcloud compute packet-mirrorings create ingress-only-mirroring \
  --region=us-central1 \
  --network=my-vpc \
  --mirrored-subnets=my-subnet \
  --collector-ilb=$ENDPOINT_FWD_RULE \
  --filter-direction=INGRESS

# List packet mirroring policies
gcloud compute packet-mirrorings list --region=us-central1

# Describe a mirroring policy
gcloud compute packet-mirrorings describe my-mirroring \
  --region=us-central1

# Disable mirroring (without deleting)
gcloud compute packet-mirrorings update my-mirroring \
  --region=us-central1 \
  --no-enable

# Delete mirroring policy
gcloud compute packet-mirrorings delete my-mirroring \
  --region=us-central1
```

### Viewing IDS Threats in Cloud Logging
```bash
# View all IDS threat detections
gcloud logging read \
  'resource.type="ids.googleapis.com/Endpoint"' \
  --format="table(timestamp,jsonPayload.threat_id,jsonPayload.category,jsonPayload.severity,jsonPayload.source_ip_address,jsonPayload.destination_ip_address)"

# View only CRITICAL and HIGH severity
gcloud logging read \
  'resource.type="ids.googleapis.com/Endpoint" AND 
   (jsonPayload.severity="CRITICAL" OR jsonPayload.severity="HIGH")' \
  --format="json" \
  --limit=10

# View threats from a specific source IP
gcloud logging read \
  'resource.type="ids.googleapis.com/Endpoint" AND 
   jsonPayload.source_ip_address="192.0.2.100"' \
  --limit=20

# Count threats by severity
gcloud logging read \
  'resource.type="ids.googleapis.com/Endpoint"' \
  --format="value(jsonPayload.severity)" | sort | uniq -c | sort -rn
```

### Setting Up Automated Response with Log-Based Alerts
```bash
# Create a log-based metric for HIGH/CRITICAL IDS alerts
gcloud logging metrics create ids-high-critical \
  --description="Count of HIGH/CRITICAL IDS threats" \
  --log-filter='resource.type="ids.googleapis.com/Endpoint" AND (jsonPayload.severity="HIGH" OR jsonPayload.severity="CRITICAL")'

# Create a Pub/Sub topic for alerts
gcloud pubsub topics create ids-threat-alerts

# Create a log sink to Pub/Sub for real-time alerts
gcloud logging sinks create ids-sink \
  pubsub.googleapis.com/projects/PROJECT_ID/topics/ids-threat-alerts \
  --log-filter='resource.type="ids.googleapis.com/Endpoint" AND jsonPayload.severity="CRITICAL"'

# Grant the sink's SA permission to publish to Pub/Sub
SINK_SA=$(gcloud logging sinks describe ids-sink --format="value(writerIdentity)")
gcloud pubsub topics add-iam-policy-binding ids-threat-alerts \
  --member=$SINK_SA \
  --role=roles/pubsub.publisher

# Create an alerting policy on the metric
gcloud alpha monitoring policies create \
  --policy-from-file=alert-policy.json
```

### VPC Flow Logs for Network Analysis
```bash
# Enable flow logs with full metadata
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-flow-sampling=1.0 \
  --logging-metadata=include-all

# Export flow logs to BigQuery for analysis
gcloud logging sinks create flow-logs-bq \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/network_logs \
  --log-filter='resource.type="gce_subnetwork" AND log_name="projects/PROJECT_ID/logs/compute.googleapis.com%2Fvpc_flows"'

# Query for suspicious large data transfers (in BigQuery)
# SELECT
#   jsonPayload.connection.src_ip,
#   jsonPayload.connection.dest_ip,
#   SUM(CAST(jsonPayload.bytes_sent AS INT64)) as total_bytes
# FROM `project.dataset.compute_googleapis_com_vpc_flows_*`
# WHERE _PARTITIONTIME >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
# GROUP BY 1, 2
# HAVING total_bytes > 1000000000  -- 1 GB threshold
# ORDER BY total_bytes DESC
```

### Network Intelligence Center
```bash
# Run a connectivity test
gcloud network-management connectivity-tests create test-ssh \
  --source-instance=projects/PROJECT_ID/zones/us-central1-a/instances/my-vm \
  --destination-instance=projects/PROJECT_ID/zones/us-central1-a/instances/private-vm \
  --protocol=TCP \
  --destination-port=22

# Get connectivity test result
gcloud network-management connectivity-tests describe test-ssh \
  --format="yaml(reachabilityDetails)"

# Rerun a test
gcloud network-management connectivity-tests rerun test-ssh

# List all connectivity tests
gcloud network-management connectivity-tests list
```

---

## Hands-On Practice

### Exercise 1: Full Cloud IDS Setup

```bash
PROJECT_ID=$(gcloud config get-value project)

# Enable APIs
gcloud services enable ids.googleapis.com networksecurity.googleapis.com

# Create endpoint (takes ~15 min)
gcloud ids endpoints create prod-ids \
  --network=default \
  --zone=us-central1-a \
  --severity=LOW \
  --description="Production IDS"

echo "Waiting for endpoint to be ready..."
while [[ $(gcloud ids endpoints describe prod-ids --zone=us-central1-a \
  --format="value(state)") != "READY" ]]; do
  echo "  Status: $(gcloud ids endpoints describe prod-ids \
    --zone=us-central1-a --format='value(state)')"
  sleep 60
done

echo "Endpoint ready!"

# Get the forwarding rule
FWD_RULE=$(gcloud ids endpoints describe prod-ids \
  --zone=us-central1-a \
  --format="value(endpointForwardingRule)")

# Create packet mirroring
gcloud compute packet-mirrorings create mirror-to-ids \
  --region=us-central1 \
  --network=default \
  --mirrored-subnets=default \
  --collector-ilb=$FWD_RULE

echo "IDS setup complete. Monitoring traffic in us-central1."
```

### Exercise 2: Simulate Threat Detection (Test Environment)

```bash
# Create a VM to test from (in the mirrored subnet)
gcloud compute instances create test-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --network=default

# SSH into the VM and run a nmap scan (simulates reconnaissance)
gcloud compute ssh test-vm --zone=us-central1-a \
  --command="sudo apt-get install -y nmap && nmap -sV 10.0.0.0/24"

# Check IDS logs (within a few minutes)
gcloud logging read \
  'resource.type="ids.googleapis.com/Endpoint"' \
  --limit=10 \
  --format="table(timestamp,jsonPayload.severity,jsonPayload.category,jsonPayload.threat_id)"
```

---

## Review Questions

1. Cloud IDS detects a **Critical** severity threat from a VM in your VPC. The VM is actively exfiltrating data. What does Cloud IDS do automatically? What must you do manually?

2. What is the relationship between **Packet Mirroring** and **Cloud IDS**? Can you use Packet Mirroring without Cloud IDS?

3. You need to detect threats on VMs in `us-central1-a` and `us-central1-b`. How many Cloud IDS endpoints do you need?

4. VPC Flow Logs are already enabled. Why would you also enable Cloud IDS?

5. How would you automatically block an IP detected by Cloud IDS?

---

## Key Exam Points

- **Cloud IDS DETECTS only** — it does NOT block traffic. You need Cloud Armor or firewall rules to block
- **Packet Mirroring copies traffic** to the IDS — the original traffic is unaffected  
- **IDS endpoint is per-zone** — not regional
- **Palo Alto signatures are managed by Google** — automatically updated, no maintenance
- **Minimum severity matters** — setting `INFORMATIONAL` captures everything (more noise), `HIGH` captures only serious threats
- **Cloud IDS vs Cloud Armor** — IDS is layer 3/4/7 network detection; Cloud Armor is layer 7 WAF/DDoS prevention at the edge
- **Response automation** — Cloud IDS → Pub/Sub → Cloud Functions → Cloud Armor/Firewall is the recommended automated response pattern
