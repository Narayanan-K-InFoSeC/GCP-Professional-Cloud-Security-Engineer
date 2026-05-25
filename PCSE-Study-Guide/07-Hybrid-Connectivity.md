# 07 — Hybrid Connectivity Security

> **Domain 2 | Weight: ~4% of total exam**  
> **Time:** ~2.5 hours concept + 1 hour hands-on

---

## Concepts

### 1. Connectivity Options Overview

| Option | Type | Bandwidth | Encryption | SLA |
|--------|------|-----------|-----------|-----|
| **Cloud VPN (HA)** | Internet-based IPsec | Up to 3 Gbps/tunnel | IPsec (built-in) | 99.99% |
| **Cloud VPN (Classic)** | Internet-based IPsec | Up to 3 Gbps/tunnel | IPsec (built-in) | 99.9% |
| **Dedicated Interconnect** | Physical fiber to Google | 10 or 100 Gbps | None (MACsec optional) | 99.9%–99.99% |
| **Partner Interconnect** | Via service provider | 50 Mbps–50 Gbps | Depends on provider | 99.9%–99.99% |
| **Cross-Cloud Interconnect** | Physical to AWS/Azure | 10 or 100 Gbps | Provider dependent | 99.9% |

---

### 2. Cloud VPN — HA VPN (Recommended)

**HA VPN (High Availability VPN):**
- Two VPN gateways with two external IP addresses each
- Uses BGP (Border Gateway Protocol) for dynamic routing
- **SLA: 99.99%** when configured with two tunnels
- All traffic encrypted with **IKEv2** by default

**Classic VPN (Legacy):**
- Single external IP per gateway
- Supports static routing
- **SLA: 99.9%**
- Avoid for new deployments

**Tunnel specs:**
- Maximum throughput: ~3 Gbps per tunnel
- For higher bandwidth: multiple tunnels with ECMP

**IKE versions:**
- **IKEv2** — recommended, more secure, faster
- **IKEv1** — legacy, supported but avoid

---

### 3. Cloud Interconnect

**Dedicated Interconnect:**
- Physical fiber from your data center to a Google colocation facility
- 10 Gbps or 100 Gbps connections
- Traffic does NOT traverse the public internet
- **No built-in encryption** — use MACsec or application-level TLS
- Google peering locations (edge PoPs) around the world

**Partner Interconnect:**
- Connect via a service provider (AT&T, Equinix, etc.)
- Capacity: 50 Mbps to 50 Gbps
- Good when you can't reach a Google colocation facility

**When to use Interconnect vs. VPN:**
- Interconnect: high throughput (>3 Gbps), consistent latency, compliance requirements
- VPN: lower cost, works anywhere, encrypted by default, lower throughput

---

### 4. MACsec (Layer 2 Encryption for Interconnect)

**MACsec (Media Access Control security)** encrypts traffic at Layer 2 on Dedicated Interconnect.

- Provides encryption **between your router and Google's edge**
- Fills the encryption gap that Dedicated Interconnect has by default
- Uses **256-bit GCM-AES** encryption
- Configure on both your router and Google's Cloud Router side

**When needed:**
- Compliance requirements mandate data encryption in transit
- You use Dedicated Interconnect but need encryption

---

### 5. Cloud Router and BGP

**Cloud Router** manages dynamic routing between on-premises and GCP:
- Exchanges routes via BGP
- Advertises GCP subnet routes to on-premises
- Learns on-premises routes from BGP peer

**Route advertisement modes:**
- `ALL_SUBNETS` — advertise all VPC subnets
- `CUSTOM` — advertise specific routes/custom prefixes

**Security consideration:** Only advertise the routes that on-premises needs.

---

### 6. Network Connectivity Center (NCC)

Hub-and-spoke model for connecting multiple on-premises networks and GCP VPCs:

```
On-Prem Site A (VPN spoke) ──┐
On-Prem Site B (IC spoke)  ──┤── NCC Hub (GCP) ──── GCP VPC
On-Prem Site C (VPN spoke) ──┘
```

Simplifies multi-site connectivity management.

---

### 7. Private Service Connect (PSC)

Allows **private, one-way access** from a VPC to:
- Google APIs (`storage.googleapis.com`, `bigquery.googleapis.com`)
- Third-party services (published via PSC)
- Your own services in other VPCs/projects

PSC endpoint in your VPC → routes to the service privately (no internet, no VPC peering required).

**For Google APIs:**
- `private.googleapis.com` — standard APIs
- `restricted.googleapis.com` — APIs supported by VPC Service Controls

---

### 8. Cloud NAT Security

**Cloud NAT** provides outbound internet access for VMs without external IPs.

Security features:
- All VMs share NAT IPs (not traceable to individual VMs without logs)
- **NAT logging** — record successful and failed connections
- **Min ports per VM** — control NAT port exhaustion
- **Endpoint-Independent Mapping (EIM)** — prevent asymmetric routing attacks

---

## gcloud Commands

### Cloud VPN — HA VPN Setup
```bash
# Step 1: Create Cloud Router
gcloud compute routers create vpn-router \
  --network=my-vpc \
  --region=us-central1 \
  --asn=65001  # Your GCP BGP ASN

# Step 2: Create HA VPN gateway
gcloud compute vpn-gateways create my-vpn-gateway \
  --network=my-vpc \
  --region=us-central1

# List HA VPN gateways
gcloud compute vpn-gateways list --region=us-central1

# Describe (get external IPs for on-prem config)
gcloud compute vpn-gateways describe my-vpn-gateway \
  --region=us-central1

# Step 3: Create external (peer) VPN gateway (represents on-prem)
gcloud compute external-vpn-gateways create on-prem-gw \
  --interfaces=0=ON_PREM_IP_1,1=ON_PREM_IP_2

# Step 4: Create VPN tunnels (two tunnels for HA)
gcloud compute vpn-tunnels create vpn-tunnel-1 \
  --region=us-central1 \
  --vpn-gateway=my-vpn-gateway \
  --interface=0 \
  --peer-external-gateway=on-prem-gw \
  --peer-external-gateway-interface=0 \
  --shared-secret="STRONG_SHARED_SECRET_HERE" \
  --ike-version=2 \
  --router=vpn-router

gcloud compute vpn-tunnels create vpn-tunnel-2 \
  --region=us-central1 \
  --vpn-gateway=my-vpn-gateway \
  --interface=1 \
  --peer-external-gateway=on-prem-gw \
  --peer-external-gateway-interface=1 \
  --shared-secret="STRONG_SHARED_SECRET_HERE" \
  --ike-version=2 \
  --router=vpn-router

# Step 5: Add BGP sessions to the router
gcloud compute routers add-bgp-peer vpn-router \
  --region=us-central1 \
  --interface=vpn-tunnel-1 \
  --peer-name=bgp-peer-1 \
  --peer-asn=65002 \
  --peer-ip-address=169.254.1.2 \
  --interface-ip-address=169.254.1.1/30

gcloud compute routers add-bgp-peer vpn-router \
  --region=us-central1 \
  --interface=vpn-tunnel-2 \
  --peer-name=bgp-peer-2 \
  --peer-asn=65002 \
  --peer-ip-address=169.254.2.2 \
  --interface-ip-address=169.254.2.1/30

# Check tunnel status
gcloud compute vpn-tunnels describe vpn-tunnel-1 \
  --region=us-central1 \
  --format="get(status,detailedStatus)"

# List all tunnels
gcloud compute vpn-tunnels list --region=us-central1
```

### Cloud Router Operations
```bash
# View learned routes from on-prem
gcloud compute routers get-status vpn-router \
  --region=us-central1 \
  --format="yaml(result.bestRoutes)"

# View router BGP status
gcloud compute routers get-status vpn-router \
  --region=us-central1 \
  --format="yaml(result.bgpPeerStatus)"

# Update router to advertise custom routes
gcloud compute routers update-bgp-peer vpn-router \
  --peer-name=bgp-peer-1 \
  --region=us-central1 \
  --advertisement-mode=CUSTOM \
  --advertised-route-priority=100
```

### Cloud Interconnect
```bash
# List VLAN attachments (after physical Interconnect is set up)
gcloud compute interconnects attachments list

# Create VLAN attachment for Dedicated Interconnect
gcloud compute interconnects attachments dedicated create my-attachment \
  --region=us-central1 \
  --router=vpn-router \
  --interconnect=my-interconnect \
  --bandwidth=BPS_1G \
  --vlan=100

# Describe attachment
gcloud compute interconnects attachments describe my-attachment \
  --region=us-central1

# Check interconnect link state
gcloud compute interconnects describe my-interconnect \
  --format="get(operationalStatus,linkType)"
```

### Private Service Connect
```bash
# Create PSC endpoint for Google APIs
gcloud compute addresses create psc-google-apis \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.0.0.100 \
  --network=my-vpc

# Create PSC forwarding rule
gcloud compute forwarding-rules create psc-googleapis-fw-rule \
  --global \
  --network=my-vpc \
  --address=psc-google-apis \
  --target-google-apis-bundle=all-apis  # or 'vpc-sc' for restricted

# Verify PSC endpoint
gcloud compute forwarding-rules describe psc-googleapis-fw-rule --global

# Create DNS response policy to route googleapis.com to PSC endpoint
gcloud dns response-policies create psc-policy \
  --networks=my-vpc \
  --description="Route Google APIs to PSC endpoint"

gcloud dns response-policies rules create googleapis-rule \
  --response-policy=psc-policy \
  --dns-name="*.googleapis.com." \
  --local-data=name="*.googleapis.com.",type=A,ttl=300,rrdatas=10.0.0.100
```

### Cloud NAT
```bash
# Create NAT gateway with logging
gcloud compute routers nats create my-nat-gateway \
  --router=vpn-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --enable-logging \
  --log-filter=ALL  # ALL, ERRORS_ONLY, TRANSLATIONS_ONLY

# View NAT logs
gcloud logging read \
  'resource.type="nat_gateway"' \
  --limit=10 \
  --format="table(timestamp,jsonPayload.connection.src_ip,jsonPayload.connection.dest_ip)"
```

---

## Hands-On Practice

### Exercise 1: Verify VPN Tunnel Health

```bash
# Check all tunnel statuses
gcloud compute vpn-tunnels list \
  --format="table(name,region,status,peerIp)"

# Check specific tunnel detail
gcloud compute vpn-tunnels describe vpn-tunnel-1 \
  --region=us-central1

# Check BGP routes learned
gcloud compute routers get-status vpn-router \
  --region=us-central1 \
  --format="yaml"
```

### Exercise 2: Set Up Private API Access via PSC

```bash
PROJECT=$(gcloud config get-value project)

# Create a PSC endpoint for restricted googleapis (for VPC-SC)
gcloud compute addresses create restricted-apis-endpoint \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.0.10.10 \
  --network=default

gcloud compute forwarding-rules create restricted-apis-rule \
  --global \
  --network=default \
  --address=restricted-apis-endpoint \
  --target-google-apis-bundle=vpc-sc

echo "PSC endpoint created at 10.0.10.10"
echo "Configure DNS: *.googleapis.com → 10.0.10.10"
```

---

## Review Questions

1. Your on-prem network connects to GCP via Dedicated Interconnect. A compliance audit requires all traffic to be encrypted in transit. What are your options and what are the trade-offs?

2. You need HA VPN with 99.99% SLA. What is the minimum configuration required?

3. Explain the difference between `private.googleapis.com` and `restricted.googleapis.com`. When should you use each?

4. Your BGP session shows `Established` but no routes are being learned from on-prem. What are the first 3 things you check?

5. What is the maximum bandwidth of a single HA VPN tunnel? How do you exceed this?

---

## Key Exam Points

- **HA VPN requires TWO tunnels** for 99.99% SLA — one tunnel = 99.9% only
- **Dedicated Interconnect has NO built-in encryption** — use MACsec or app-level TLS
- **MACsec encrypts at Layer 2** — between your router and Google's edge
- **Partner Interconnect** uses a service provider — Google doesn't control the middle
- **`restricted.googleapis.com`** is required for VPC Service Controls — not `private.googleapis.com`
- **Cloud Router is required** for HA VPN and Interconnect (for dynamic routing)
- **VPN shared secret** should be strong (32+ characters, random) — weak secrets = security risk
- **Cloud NAT** does NOT allow inbound initiated connections — outbound only
