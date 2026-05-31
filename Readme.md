# GCP Professional Cloud Security Engineer — End-to-End Exam Prep Roadmap 2026

> **Exam Code:** `Professional Cloud Security Engineer`  
> **Format:** 50–60 questions | 120 minutes | Multiple choice & multiple select  
> **Passing Score:** ~70% (scaled scoring)  
> **Prerequisites:** Recommended 3+ years hands-on GCP experience  
> **Updated for:** 2025–2026 Exam Guide revision

---

## Table of Contents

- [Exam Domain Weights](#exam-domain-weights)
- [Domain 1: Configuring Access (27%)](#domain-1-configuring-access-within-a-cloud-solution-environment)
- [Domain 2: Network Security (23%)](#domain-2-configuring-network-security)
- [Domain 3: Data Protection (20%)](#domain-3-ensuring-data-protection)
- [Domain 4: Managing Operations (22%)](#domain-4-managing-operations-within-a-cloud-solution-environment)
- [Domain 5: Compliance (8%)](#domain-5-supporting-compliance-requirements)
- [Key GCP Security Services Cheatsheet](#key-gcp-security-services-cheatsheet)
- [Hands-On Labs Checklist](#hands-on-labs-checklist)
- [Practice Resources](#practice-resources)
- [Study Timeline (8 Weeks)](#study-timeline-8-weeks)

---

## Exam Domain Weights

| Domain | Topic | Weight |
|--------|-------|--------|
| 1 | Configuring access within a cloud solution environment | 27% |
| 2 | Configuring network security | 23% |
| 3 | Ensuring data protection | 20% |
| 4 | Managing operations within a cloud solution environment | 22% |
| 5 | Supporting compliance requirements | 8% |

---

## Domain 1: Configuring Access Within a Cloud Solution Environment

### 1.1 Cloud Identity and Access Management (IAM)

#### Identity Management
- [ ] **Google Cloud Identity** — managed identity service (users, groups, service accounts)
- [ ] **Workspace vs. Cloud Identity** — differences, federation, provisioning
- [ ] **Identity Federation** — SAML 2.0, OIDC, external IdPs (Okta, Azure AD, ADFS)
- [ ] **Workforce Identity Federation** — pool-based external workforce authentication (no GSuite required)
- [ ] **Workload Identity Federation** — keyless auth for on-prem/AWS/Azure workloads
- [ ] **Domain Restriction** — `constraints/iam.allowedPolicyMemberDomains`
- [ ] **User lifecycle management** — provisioning, deprovisioning, SCIM

#### IAM Policies and Roles
- [ ] **Policy hierarchy** — Organization → Folder → Project → Resource
- [ ] **Policy inheritance** — child inherits parent; deny overrides allow
- [ ] **Basic roles** — Owner, Editor, Viewer (avoid in production)
- [ ] **Predefined roles** — service-specific, principle of least privilege
- [ ] **Custom roles** — project-level and org-level, stages (ALPHA, BETA, GA)
- [ ] **IAM conditions** — attribute-based access control (time, resource tags, IP)
- [ ] **IAM deny policies** — explicit deny, override allow bindings
- [ ] **`allUsers` / `allAuthenticatedUsers`** — public access patterns and risks
- [ ] **IAM policy troubleshooting** — `iam.testIamPermissions`, Policy Analyzer
- [ ] **Policy Analyzer** — query who has access to what
- [ ] **Recommender API** — IAM role recommendations, lateral movement insights

#### Service Accounts
- [ ] **Service account types** — user-managed, default, Google-managed
- [ ] **Service account keys** — JSON keys, risks, rotation, `constraints/iam.disableServiceAccountKeyCreation`
- [ ] **Workload Identity Federation** vs. service account keys (preferred pattern)
- [ ] **Service account impersonation** — `roles/iam.serviceAccountTokenCreator`
- [ ] **Service account binding** — attaching to compute resources
- [ ] **Short-lived credentials** — `generateAccessToken`, `generateIdToken`
- [ ] **Service Account Insights** — unused SAs, overprivileged SAs
- [ ] **`iam.disableServiceAccountCreation`** constraint

#### Resource Hierarchy and Org Policies
- [ ] **Organization resource** — root node, domain-based
- [ ] **Folders** — grouping, environment separation (prod/dev/staging)
- [ ] **Projects** — billing, quota, API enablement boundary
- [ ] **Organization Policy Service** — boolean, list, resource constraints
- [ ] **Key org constraints:**
  - `constraints/compute.requireShieldedVm`
  - `constraints/compute.vmExternalIpAccess`
  - `constraints/compute.restrictCloudNATUsage`
  - `constraints/storage.uniformBucketLevelAccess`
  - `constraints/storage.publicAccessPrevention`
  - `constraints/iam.allowedPolicyMemberDomains`
  - `constraints/gcp.resourceLocations`
  - `constraints/compute.restrictLoadBalancerCreationForTypes`
  - `constraints/cloudfunctions.allowedIngressSettings`
- [ ] **Custom org policy constraints** — CEL-based custom constraints (2024+)
- [ ] **Policy inheritance and override** — `merge`, `replace` behaviors

### 1.2 Cloud Identity-Aware Proxy (IAP)

- [ ] **IAP overview** — context-aware access to apps without VPN
- [ ] **IAP for App Engine, GCE, GKE, Cloud Run**
- [ ] **IAP-secured web resources** — backend services, HTTPS load balancer
- [ ] **IAP connector** — on-premises app protection
- [ ] **Context-aware access levels** — device policy, IP, identity
- [ ] **Access Context Manager** — access policies, access levels, service perimeters
- [ ] **IAP programmatic authentication** — OIDC tokens, service accounts
- [ ] **IAP audit logs** — `cloudaudit.googleapis.com/data_access`

### 1.3 Access Context Manager & BeyondCorp

- [ ] **BeyondCorp Enterprise** — zero-trust model, device trust
- [ ] **Access levels** — basic (device/IP), custom (CEL expressions)
- [ ] **Service perimeters** — VPC Service Controls integration
- [ ] **Access policies** — org-level and scoped policies
- [ ] **Dry-run mode** — test perimeters without enforcement
- [ ] **Access binding** — bind IAP to access levels

### 1.4 Privileged Access

- [ ] **Privileged Access Manager (PAM)** — just-in-time access, time-bound grants
- [ ] **Break-glass access** — emergency access procedures
- [ ] **Admin activity logging** — always-on, cannot be disabled
- [ ] **Super admin account protection** — 2FA, hardware keys, separate accounts
- [ ] **`roles/owner` risk** — avoid assigning, audit regularly

---

## Domain 2: Configuring Network Security

### 2.1 Virtual Private Cloud (VPC) Security

#### VPC Fundamentals
- [ ] **VPC architecture** — global, subnets are regional, routing
- [ ] **Default vs. custom VPC** — security implications of default VPC
- [ ] **Shared VPC** — host project, service projects, centralized networking
- [ ] **VPC Peering** — private connectivity, non-transitive, no overlapping CIDRs
- [ ] **Private Google Access** — access Google APIs without external IPs
- [ ] **Private Service Connect (PSC)** — private endpoints for Google services and third-party
- [ ] **VPC Flow Logs** — network monitoring, sampling rate, log fields

#### Firewall Rules
- [ ] **VPC firewall rules** — stateful, ingress/egress, priority (0–65534)
- [ ] **Implied rules** — deny all ingress (65535), allow all egress (65535)
- [ ] **Target tags vs. service accounts** — tag-based vs. SA-based firewall targeting
- [ ] **Firewall rule logging** — per-rule logging, Cloud Logging integration
- [ ] **Hierarchical firewall policies** — org/folder-level, `goto_next` action
- [ ] **Network firewall policies** — regional and global, stateful/stateless rules
- [ ] **Firewall Insights** — shadow rules, overly permissive rules, unused rules
- [ ] **`0.0.0.0/0` ingress rules** — highest risk, always audit

#### Network Intelligence Center
- [ ] **Connectivity Tests** — end-to-end packet path analysis
- [ ] **Network Topology** — visual network map
- [ ] **Firewall Insights** — rule utilization, recommendations
- [ ] **Performance Dashboard** — latency, packet loss metrics

### 2.2 Hybrid and Multi-Cloud Connectivity

- [ ] **Cloud VPN** — IPsec, HA VPN (99.99% SLA), Classic VPN
- [ ] **HA VPN** — two tunnels, BGP routing, redundancy
- [ ] **Cloud Interconnect** — Dedicated (10/100 Gbps), Partner (50 Mbps–50 Gbps)
- [ ] **MACsec encryption** — layer 2 encryption on Dedicated Interconnect
- [ ] **Cloud Router** — dynamic routing, BGP, route advertisements
- [ ] **Network Connectivity Center (NCC)** — hub-and-spoke for hybrid networks
- [ ] **Cross-Cloud Interconnect** — direct connectivity to AWS, Azure

### 2.3 Load Balancing and CDN Security

- [ ] **Cloud Armor** — DDoS protection, WAF, security policies
- [ ] **Cloud Armor policies** — allow/deny rules, rate limiting, adaptive protection
- [ ] **Adaptive Protection (ML-based DDoS)** — automatic attack detection
- [ ] **WAF rules** — OWASP Top 10, preconfigured rules, custom rules
- [ ] **Rate limiting** — throttle by IP, expression-based
- [ ] **Bot management** — reCAPTCHA Enterprise integration
- [ ] **Cloud CDN** — caching, signed URLs, signed cookies
- [ ] **SSL policies** — TLS minimum version, cipher suites (COMPATIBLE, MODERN, RESTRICTED, CUSTOM)
- [ ] **Managed SSL certificates** — Google-managed vs. self-managed
- [ ] **Cloud Load Balancing types** — Global HTTPS, Regional, TCP/UDP, Internal

### 2.4 DNS Security

- [ ] **Cloud DNS** — managed DNS, DNSSEC support
- [ ] **DNSSEC** — enable/disable per zone, key signing, zone signing
- [ ] **DNS response policies** — local overrides, split-horizon DNS
- [ ] **DNS peering** — cross-project DNS resolution
- [ ] **Private DNS zones** — internal resolution, forwarding zones
- [ ] **DNS filtering** — response policy zones for content filtering

### 2.5 Private Connectivity and Endpoints

- [ ] **Private Google Access for on-premises** — via VPN/Interconnect
- [ ] **VPC Service Controls** — API-level perimeters (see Domain 3)
- [ ] **Private Service Connect endpoints** — for Google APIs (`restricted.googleapis.com`)
- [ ] **Serverless VPC Access** — Cloud Run/Functions to VPC connectivity
- [ ] **Cloud NAT** — outbound internet without external IPs, logging

### 2.6 Kubernetes (GKE) Network Security

- [ ] **GKE network policies** — Kubernetes NetworkPolicy (Calico)
- [ ] **Dataplane V2** — eBPF-based, built-in network policy enforcement
- [ ] **Authorized networks** — restrict API server access by CIDR
- [ ] **Private clusters** — no public endpoint for nodes, optional private master
- [ ] **Workload Identity** — pod-to-GCP service auth without SA keys
- [ ] **Binary Authorization** — image signing, admission control
- [ ] **GKE Sandbox (gVisor)** — kernel isolation for untrusted workloads
- [ ] **Confidential GKE nodes** — encrypted in-use memory
- [ ] **Shielded GKE nodes** — Secure Boot, vTPM, integrity monitoring

### 2.7 Intrusion Detection and Threat Protection

- [ ] **Cloud IDS** — network threat detection (Palo Alto-powered), mirrored traffic
- [ ] **Cloud IDS severity levels** — Low, Medium, High, Critical
- [ ] **Packet Mirroring** — clone traffic to IDS/analysis tools
- [ ] **VPC Flow Logs analysis** — detect anomalous traffic patterns

---

## Domain 3: Ensuring Data Protection

### 3.1 Encryption

#### Encryption at Rest
- [ ] **Google default encryption** — AES-256, automatic for all data
- [ ] **Customer-Managed Encryption Keys (CMEK)** — Cloud KMS key management
- [ ] **Customer-Supplied Encryption Keys (CSEK)** — user provides key per-request
- [ ] **Key rotation** — automatic (CMEK), manual rotation, crypto periods
- [ ] **Cloud KMS key types** — Software, HSM, External (EKMS/Cloud EKM)
- [ ] **Cloud External Key Manager (Cloud EKM)** — keys hosted externally (Thales, Fortanix)
- [ ] **Key Access Justifications** — require manual approval for key usage
- [ ] **CMEK for services** — BigQuery, Cloud Storage, Compute Engine, GKE, Spanner, etc.

#### Encryption in Transit
- [ ] **TLS everywhere** — Google infrastructure default
- [ ] **Application Layer Transport Security (ALTS)** — Google-internal mutual auth
- [ ] **SSL policies** — enforce minimum TLS 1.2 or 1.3
- [ ] **Certificate Authority Service (CAS)** — private CA, certificate lifecycle
- [ ] **Certificate Manager** — manage public/private SSL certificates at scale
- [ ] **MACsec** — Dedicated Interconnect layer-2 encryption

#### Confidential Computing
- [ ] **Confidential VMs** — AMD SEV, encrypt in-use memory
- [ ] **Confidential GKE nodes** — node-level memory encryption
- [ ] **Confidential Space** — multi-party computation, attestation-based
- [ ] **Confidential Dataflow** — TEE-based data processing

### 3.2 Cloud Key Management Service (KMS)

- [ ] **Key rings** — regional grouping, cannot be deleted
- [ ] **CryptoKeys** — key material, purpose (ENCRYPT_DECRYPT, SIGN_VERIFY, MAC)
- [ ] **Key versions** — primary, enabled, disabled, destroyed
- [ ] **Key destruction** — 24-hour delay (configurable), irreversible after
- [ ] **Importing key material** — BYOK, RSA-AES key wrapping
- [ ] **Cloud HSM** — FIPS 140-2 Level 3 hardware security modules
- [ ] **KMS IAM roles:**
  - `roles/cloudkms.cryptoKeyEncrypterDecrypter`
  - `roles/cloudkms.cryptoKeyEncrypter`
  - `roles/cloudkms.cryptoKeyDecrypter`
  - `roles/cloudkms.admin`
- [ ] **Key access logging** — data access logs for KMS
- [ ] **Autokey** — automatically create and manage CMEK keys per resource

### 3.3 Secret Management

- [ ] **Secret Manager** — store API keys, passwords, certificates, TLS keys
- [ ] **Secret versions** — immutable, enable/disable/destroy versions
- [ ] **Secret replication** — automatic (Google-managed) or user-managed regions
- [ ] **CMEK for Secret Manager** — encrypt secrets with Cloud KMS
- [ ] **Secret Manager IAM** — `roles/secretmanager.secretAccessor`
- [ ] **Secret rotation** — manual or Pub/Sub-triggered rotation
- [ ] **Secret annotations and labels** — metadata for secrets
- [ ] **Regional secrets** — data residency compliance

### 3.4 Data Loss Prevention (DLP) — now Sensitive Data Protection

- [ ] **Cloud DLP / Sensitive Data Protection** — discover, classify, de-identify PII
- [ ] **InfoType detectors** — built-in (EMAIL, PHONE, SSN, CREDIT_CARD) and custom
- [ ] **Inspection jobs** — scan Cloud Storage, BigQuery, Datastore
- [ ] **De-identification techniques:**
  - Masking (character mask)
  - Redaction (replace with `[REDACTED]`)
  - Tokenization (format-preserving encryption)
  - Pseudonymization (deterministic encryption)
  - Generalization (date shifting, bucketing)
  - Cryptographic hashing
- [ ] **Re-identification risk analysis** — k-anonymity, l-diversity
- [ ] **DLP in CI/CD pipelines** — scan code repos for secrets
- [ ] **Data profiles** — automated scanning, data risk levels
- [ ] **Discovery service** — continuous scanning, org-wide

### 3.5 Cloud Storage Security

- [ ] **Uniform bucket-level access** — disable ACLs, use IAM only
- [ ] **Public Access Prevention** — `constraints/storage.publicAccessPrevention`
- [ ] **Bucket-level vs. object-level IAM** — principle of least privilege
- [ ] **Signed URLs** — time-limited access to private objects
- [ ] **Signed Policy Documents** — control uploads via HTML forms
- [ ] **Retention policies** — WORM (write once read many), locked retention
- [ ] **Object Lifecycle Management** — auto-delete, auto-transition storage classes
- [ ] **Object versioning** — protect against accidental deletion
- [ ] **VPC Service Controls for GCS** — prevent data exfiltration
- [ ] **HMAC keys** — for S3-compatible HMAC authentication
- [ ] **Audit logging** — DATA_READ, DATA_WRITE, ADMIN_READ

### 3.6 Database Security

#### Cloud SQL
- [ ] **Cloud SQL authorized networks** — IP allowlisting
- [ ] **Cloud SQL Auth Proxy** — IAM-based, encrypted tunnel, no public IP needed
- [ ] **Cloud SQL Private IP** — VPC-internal access
- [ ] **Customer-managed keys** — CMEK for Cloud SQL
- [ ] **SSL/TLS connections** — enforce SSL, server/client certificates
- [ ] **IAM database authentication** — use Google identities for login

#### BigQuery
- [ ] **Column-level security** — policy tags, data catalog
- [ ] **Row-level security** — row access policies
- [ ] **Authorized views** — share derived data without exposing source tables
- [ ] **BigQuery CMEK** — dataset-level encryption keys
- [ ] **Data Catalog** — metadata management, policy tags
- [ ] **VPC Service Controls for BigQuery** — prevent exfiltration
- [ ] **BigQuery audit logs** — track queries, access patterns

#### Firestore / Datastore
- [ ] **Security rules** — document-level access control
- [ ] **CMEK** — encryption with Cloud KMS
- [ ] **IAM integration** — `roles/datastore.user`

#### Spanner
- [ ] **CMEK** — database encryption with KMS
- [ ] **IAM roles** — `roles/spanner.databaseReader`, `roles/spanner.databaseAdmin`
- [ ] **Audit logging** — track SQL queries
- [ ] **Private connectivity** — VPC Service Controls

### 3.7 VPC Service Controls

- [ ] **Service perimeters** — logical boundary around GCP resources/APIs
- [ ] **Perimeter types** — regular, bridge
- [ ] **Restricted services** — APIs blocked at perimeter boundary
- [ ] **Access levels** — conditions for ingress (from outside perimeter)
- [ ] **Ingress/egress rules** — fine-grained cross-perimeter access
- [ ] **VPC accessible services** — restrict which APIs VMs can call
- [ ] **Dry-run mode** — log violations without enforcing
- [ ] **Service perimeter bridges** — allow two perimeters to share resources
- [ ] **Org-level vs. project-level** — perimeter scope

---

## Domain 4: Managing Operations Within a Cloud Solution Environment

### 4.1 Cloud Logging and Audit Logs

#### Audit Log Types
- [ ] **Admin Activity logs** — always on, free, captures config changes
- [ ] **Data Access logs** — off by default, captures data reads/writes, chargeable
- [ ] **System Event logs** — GCP-generated, automatic, free
- [ ] **Policy Denied logs** — captured when access denied by org policy
- [ ] **Access Transparency logs** — Google staff access to your data (Premium+)

#### Log Management
- [ ] **Log buckets** — `_Default`, `_Required`, custom buckets
- [ ] **Log sinks** — export to Cloud Storage, BigQuery, Pub/Sub, Splunk, etc.
- [ ] **Log-based metrics** — create metrics from log patterns
- [ ] **Log exclusion filters** — reduce log volume and cost
- [ ] **Log retention** — `_Required` (400 days), `_Default` (30 days), custom
- [ ] **Log Analytics** — SQL queries on logs via BigQuery integration
- [ ] **Aggregated log sinks** — org/folder-level sink to centralize logs
- [ ] **CMEK for log buckets** — encrypt logs with customer-managed keys

#### Log Analysis for Security
- [ ] **Key log sources:**
  - `cloudaudit.googleapis.com/activity` — admin activity
  - `cloudaudit.googleapis.com/data_access` — data access
  - `compute.googleapis.com/firewall` — firewall rule logs
  - `requests` — load balancer access logs
  - VPC Flow Logs — network traffic
- [ ] **Log-based alerting** — alert on suspicious patterns
- [ ] **Anomaly detection** — unusual admin activity patterns

### 4.2 Cloud Monitoring and Security Operations

- [ ] **Cloud Monitoring** — metrics, dashboards, alerting
- [ ] **Uptime checks** — availability monitoring
- [ ] **Alerting policies** — metric-based, log-based, budget-based
- [ ] **Security dashboards** — Security Command Center integrated view
- [ ] **Custom metrics** — application-level security metrics
- [ ] **Workspaces** — cross-project monitoring

### 4.3 Security Command Center (SCC)

#### SCC Overview
- [ ] **SCC tiers** — Standard (free), Premium, Enterprise
- [ ] **SCC Enterprise** — SIEM/SOAR integration, case management, ticketing
- [ ] **Finding types** — Vulnerabilities, Misconfiguration, Threats, Posture violations
- [ ] **Assets** — inventory of GCP resources
- [ ] **Sources** — built-in and third-party finding providers
- [ ] **Marks** — user-defined annotations on findings

#### Built-in Security Sources
- [ ] **Security Health Analytics (SHA)** — misconfiguration detection
  - Public buckets, open firewall ports, weak passwords, unencrypted disks
  - Over 300 managed detectors
- [ ] **Event Threat Detection (ETD)** — real-time threat detection from logs
  - Malware, cryptomining, data exfiltration, brute force, anomalous IAM
- [ ] **Container Threat Detection (CTD)** — runtime container security
  - Added library, reverse shell, malicious script execution
- [ ] **VM Threat Detection (VMTD)** — memory-level threat detection on VMs
  - Cryptomining, rootkits, kernel tampering
- [ ] **Web Security Scanner** — OWASP vulnerability scanning for web apps
- [ ] **Rapid Vulnerability Detection (RVD)** — network-based vulnerability scanning

#### SCC Integrations
- [ ] **Chronicle SIEM** — SCC Enterprise threat correlation
- [ ] **SOAR (Security Orchestration)** — automated response playbooks
- [ ] **Pub/Sub notifications** — real-time finding export
- [ ] **BigQuery export** — findings warehouse for analysis
- [ ] **Jira/ServiceNow** — ticketing integration
- [ ] **SCC findings to log sinks** — export findings to SIEM

#### Posture Management
- [ ] **Security Posture service** — define, deploy, monitor security posture
- [ ] **Posture templates** — predefined security configurations
- [ ] **Posture violations** — drift detection from desired state
- [ ] **Policy Intelligence** — recommendations, asset insights

### 4.4 Cloud Armor and WAF Operations

- [ ] **Security policy management** — rule priority, preview mode
- [ ] **Adaptive Protection** — ML-based attack detection, auto-propose rules
- [ ] **Rate-based banning** — IP-based threshold rules
- [ ] **Named IP lists** — Google-curated lists (Tor, scanners, cloud providers)
- [ ] **Edge security policies** — Cloud CDN edge enforcement
- [ ] **Cloud Armor logs** — `requests` log, `jsonPayload.enforcedSecurityPolicy`
- [ ] **Bot management** — reCAPTCHA token validation, manual challenge

### 4.5 Incident Response

- [ ] **GCP incident response process** — detect, analyze, contain, eradicate, recover
- [ ] **Forensic logging** — preserving audit logs for investigation
- [ ] **Log lock** — `_Required` bucket cannot be modified or deleted
- [ ] **Snapshot for forensics** — disk snapshots of compromised instances
- [ ] **Disable compromised service accounts** — `gcloud iam service-accounts disable`
- [ ] **Revoke OAuth tokens** — for compromised user credentials
- [ ] **Metadata server abuse** — SSRF attacks, restrict metadata access
- [ ] **Instance isolation** — quarantine VMs with firewall rules

### 4.6 Infrastructure as Code Security

- [ ] **Terraform with GCP** — secure IaC practices, remote state security
- [ ] **Terraform Validator** — policy enforcement before apply
- [ ] **Config Connector** — Kubernetes-based GCP resource management
- [ ] **Forseti Security** — open-source GCP auditing (legacy, migrated to SCC)
- [ ] **Organization Policy** — IaC-compatible constraints
- [ ] **gcloud CLI security** — secure credential storage, ADC

### 4.7 Vulnerability and Patch Management

- [ ] **Artifact Registry** — private container and package registry
- [ ] **Container Analysis** — vulnerability scanning for containers
- [ ] **On-demand vs. continuous scanning** — push-time and runtime scanning
- [ ] **Artifact Registry vulnerability scanning** — OS and language packages
- [ ] **Binary Authorization** — enforce only signed, approved images
  - Attestors, attestations, policy types (allow, require attestation, deny)
  - Dry-run mode for Binary Auth
- [ ] **OS Config / VM Manager** — patch management, compliance scanning
- [ ] **Assured OSS** — curated, vetted open-source packages
- [ ] **Dependency scanning in Cloud Build** — supply chain security

### 4.8 Supply Chain Security

- [ ] **SLSA framework** — Supply chain Levels for Software Artifacts
- [ ] **Cloud Build provenance** — signed build attestations
- [ ] **Artifact Registry SBOM** — software bill of materials
- [ ] **Sigstore / cosign** — keyless container signing
- [ ] **Binary Authorization for Borg (BAB)** — Google-internal, similar concept for Cloud
- [ ] **Software Delivery Shield** — end-to-end supply chain security solution

---

## Domain 5: Supporting Compliance Requirements

### 5.1 Regulatory and Compliance Frameworks

- [ ] **GDPR** — EU data protection, data residency, right to erasure
- [ ] **HIPAA** — healthcare PHI protection, BAA with Google
- [ ] **PCI DSS** — cardholder data, SAQ types, QSA assessments
- [ ] **SOC 1/2/3** — service organization controls, audit reports
- [ ] **ISO 27001/27017/27018** — information security management
- [ ] **FedRAMP** — US government cloud authorization
- [ ] **NIST Cybersecurity Framework** — Identify, Protect, Detect, Respond, Recover
- [ ] **CIS Benchmarks** — GCP CIS Benchmark v2.0
- [ ] **FIPS 140-2** — cryptographic module validation (Cloud HSM = Level 3)

### 5.2 Google Compliance Offerings

- [ ] **Assured Workloads** — compliance controls for regulated industries
  - FedRAMP Moderate/High, IL2/IL4/IL5, ITAR, CJIS, DoD
  - Data residency, personnel controls, encryption requirements
- [ ] **Sovereignty Controls** — data sovereignty, access transparency
- [ ] **Google Cloud Compliance Reports Manager** — download audit reports
- [ ] **Customer Managed Access Justifications (CMAJ)** — approve Google access
- [ ] **Access Transparency** — logs of Google staff access

### 5.3 Data Residency and Sovereignty

- [ ] **`constraints/gcp.resourceLocations`** — restrict resource creation to regions
- [ ] **Dual-region and multi-region** — data replication considerations
- [ ] **Data residency for Cloud Storage** — bucket location policies
- [ ] **Sovereign controls by T-Systems** — EU sovereign cloud offering
- [ ] **Google Cloud EU** — European-specific commitments

### 5.4 Compliance Monitoring and Reporting

- [ ] **SCC Compliance Posture** — benchmark mappings (CIS, PCI, NIST, ISO)
- [ ] **SCC compliance dashboards** — violations mapped to controls
- [ ] **Policy Intelligence** — policy troubleshooting, recommendations
- [ ] **Org Policy Compliance** — track constraint violations
- [ ] **Config Audit logs** — all resource config changes
- [ ] **BigQuery audit exports** — long-term compliance log retention
- [ ] **Chronicle for compliance** — long-term log retention and analysis

### 5.5 Privacy Controls

- [ ] **Sensitive Data Protection (DLP)** — PII discovery and classification
- [ ] **Data catalog policy tags** — BigQuery column-level access control
- [ ] **Pseudonymization at scale** — DLP tokenization in pipelines
- [ ] **Right to erasure** — data deletion procedures, audit trails
- [ ] **Data processing agreements** — BAAs, DPAs with Google

---

## Key GCP Security Services Cheatsheet

| Service | Purpose | Key Feature |
|---------|---------|-------------|
| **IAM** | Identity & access | Policy hierarchy, conditions, deny policies |
| **Cloud Identity** | Identity provider | SSO, MFA, device management |
| **IAP** | Zero-trust app access | No VPN needed, context-aware |
| **Access Context Manager** | Context-aware access | Access levels, BeyondCorp |
| **Workload Identity Federation** | Keyless auth | External workload auth to GCP |
| **PAM** | Privileged access | Just-in-time, time-bound grants |
| **VPC** | Network isolation | Shared VPC, Private IP, Flow Logs |
| **Firewall Rules** | Traffic control | Hierarchical policies, SA-based |
| **Cloud Armor** | DDoS/WAF | Adaptive protection, OWASP rules |
| **Cloud IDS** | Network threat detection | Palo Alto signatures, mirrored traffic |
| **Cloud VPN / Interconnect** | Hybrid connectivity | HA VPN 99.99%, MACsec |
| **VPC Service Controls** | API perimeter | Data exfiltration prevention |
| **Cloud KMS** | Key management | Software, HSM, External keys |
| **Secret Manager** | Secret storage | Versioning, rotation, CMEK |
| **Sensitive Data Protection** | DLP | PII discovery, de-identification |
| **Binary Authorization** | Image trust | Attestors, admission control |
| **Artifact Registry** | Artifact storage | Vulnerability scanning |
| **Cloud Build** | CI/CD | Provenance, SLSA support |
| **SCC** | Security posture | SHA, ETD, CTD, VMTD |
| **Cloud Logging** | Log management | Audit logs, sinks, retention |
| **Assured Workloads** | Compliance | FedRAMP, IL, ITAR, CJIS |
| **Chronicle** | SIEM | Long-term log analysis |
| **GKE Security** | Container security | Workload Identity, Sandbox, Binary Auth |
| **OS Config** | VM management | Patch management, compliance |
| **Certificate Manager** | TLS certs | Managed certs at scale |
| **Certificate Authority Svc** | Private CA | Internal cert issuance |

---

## Hands-On Labs Checklist

### IAM & Identity
- [ ] Configure Workforce Identity Federation with a SAML provider
- [ ] Set up Workload Identity Federation for GitHub Actions
- [ ] Create custom IAM roles with minimal permissions
- [ ] Configure IAM conditions (time-based, resource-tag-based)
- [ ] Create and enforce IAM deny policies
- [ ] Use Policy Analyzer to audit access
- [ ] Set up PAM for just-in-time access

### Org Policy
- [ ] Enable `constraints/compute.requireShieldedVm` at org level
- [ ] Restrict external IPs with `constraints/compute.vmExternalIpAccess`
- [ ] Enforce uniform bucket-level access org-wide
- [ ] Create a custom org policy constraint with CEL

### Network Security
- [ ] Create hierarchical firewall policies at org and folder levels
- [ ] Set up Shared VPC between host and service projects
- [ ] Configure Cloud Armor security policy with WAF rules
- [ ] Enable Cloud IDS on a VPC subnet
- [ ] Set up Private Google Access and Private Service Connect
- [ ] Configure HA VPN with BGP routing
- [ ] Enable VPC Flow Logs and analyze in BigQuery

### VPC Service Controls
- [ ] Create a service perimeter around BigQuery and Cloud Storage
- [ ] Configure access levels and ingress/egress rules
- [ ] Test perimeter in dry-run mode before enforcement
- [ ] Set up a perimeter bridge between two perimeters

### Data Protection
- [ ] Create Cloud KMS keys (Software, HSM)
- [ ] Enable CMEK for Cloud Storage, BigQuery, and GKE
- [ ] Import external key material (BYOK)
- [ ] Store and rotate secrets in Secret Manager
- [ ] Run a DLP inspection job on Cloud Storage
- [ ] Apply DLP de-identification template to BigQuery data
- [ ] Enable column-level security with BigQuery policy tags
- [ ] Configure Cloud SQL Auth Proxy with IAM auth

### GKE Security
- [ ] Enable Workload Identity on GKE cluster
- [ ] Apply Kubernetes NetworkPolicy with Dataplane V2
- [ ] Configure Binary Authorization with attestors
- [ ] Enable GKE Sandbox for untrusted workloads
- [ ] Set up Container Threat Detection in SCC

### Logging and Monitoring
- [ ] Enable Data Access audit logs for all services
- [ ] Create aggregated log sink to centralized project
- [ ] Set up log-based alert for suspicious IAM changes
- [ ] Export SCC findings to BigQuery
- [ ] Create SCC notification to Pub/Sub
- [ ] Analyze audit logs with Log Analytics (SQL)

### Incident Response
- [ ] Disable a compromised service account
- [ ] Isolate a VM with firewall rule changes
- [ ] Take a disk snapshot for forensic analysis
- [ ] Use Connectivity Tests to diagnose network issues

### Compliance
- [ ] Create an Assured Workloads environment
- [ ] Review CIS Benchmark compliance in SCC
- [ ] Generate compliance report for PCI-DSS in SCC
- [ ] Configure Access Transparency log review

---

## Critical Exam Topics (High Frequency)

> These topics appear most frequently in exam questions:

1. **VPC Service Controls** — perimeter configuration, ingress/egress rules, dry-run
2. **IAM deny policies** — how they override allow policies
3. **Workload Identity Federation** — eliminate service account keys
4. **Cloud KMS + CMEK** — key hierarchy, rotation, destroying keys
5. **Secret Manager** — vs. KMS, use cases, rotation
6. **Binary Authorization** — attestors, policies, GKE enforcement
7. **Hierarchical firewall policies** — org/folder-level, `goto_next`
8. **Cloud Armor + Adaptive Protection** — WAF rules, rate limiting
9. **SCC findings** — SHA vs ETD vs CTD vs VMTD
10. **Audit log types** — Admin Activity vs Data Access, when each fires
11. **Org policy constraints** — which constraint does what
12. **Shared VPC** — host project, service project, IAM for Shared VPC
13. **Cloud IDS** — how it works, Packet Mirroring, threat levels
14. **DLP de-identification** — tokenization vs. masking vs. pseudonymization
15. **Access Context Manager** — access levels, BeyondCorp, IAP integration

---

## Common Exam Traps

| Trap | Reality |
|------|---------|
| CSEK protects keys from Google | CSEK keys still pass through Google infra; Cloud EKM truly keeps keys external |
| IAP replaces firewall rules | IAP works with load balancers; firewall rules are still needed |
| VPC peering is transitive | VPC peering is NOT transitive; must peer each pair |
| Default SA has Editor role | Only in old projects; always audit and disable default SA |
| Org policy works like IAM deny | Org policy restricts resource configurations, not identity access |
| Data Access logs always enabled | They're OFF by default and must be explicitly enabled |
| Cloud IDS blocks traffic | Cloud IDS only DETECTS threats; it doesn't block — use Firewall + Armor |
| CMEK prevents Google access | CMEK controls who manages keys, not Google's infra-level data access |
| Shared VPC shares billing | Billing stays in the service project; networking is centralized |

---

## Practice Resources

### Official Google Resources
- [ ] [Exam guide (2025)](https://cloud.google.com/learn/certification/cloud-security-engineer)
- [ ] [Google Cloud Skills Boost](https://cloudskillsboost.google/) — Security Engineer learning path
- [ ] [Cloud Security Foundations lab series](https://cloudskillsboost.google/paths)
- [ ] [Qwiklabs Security quests](https://cloudskillsboost.google/catalog)
- [ ] [Google Cloud documentation](https://cloud.google.com/docs/security)
- [ ] [Google Cloud architecture center — security](https://cloud.google.com/architecture/security-foundations)
- [ ] [CIS Google Cloud Foundations Benchmark](https://www.cisecurity.org/benchmark/google_cloud_computing_platform)

### Study Materials
- [ ] **"Google Cloud Security Best Practices"** — Google whitepaper
- [ ] **Coursera: Preparing for Google Cloud Security Cert**
- [ ] **A Cloud Guru / Pluralsight** — GCP Security Engineer course
- [ ] **ExamTopics / Whizlabs** — practice questions (verify answers independently)
- [ ] **TutorialsDojo** — GCP practice exams

### Hands-On Platforms
- [ ] **Google Cloud Skills Boost** (formerly Qwiklabs) — official labs
- [ ] **Cloud Shell** — free browser-based terminal
- [ ] **Google Cloud Free Tier** — $300 credit for new accounts

---

## Study Timeline (8 Weeks)

```
Week 1: Foundation
├── Cloud Identity & IAM fundamentals
├── Resource hierarchy & org policies
├── Service accounts & Workload Identity
└── Labs: IAM roles, org policies, WIF

Week 2: Network Security Part 1
├── VPC architecture & firewall rules
├── Hierarchical firewall policies
├── Shared VPC & VPC Peering
└── Labs: VPC, firewall policies, Shared VPC

Week 3: Network Security Part 2
├── Cloud Armor & WAF
├── Cloud IDS & Packet Mirroring
├── Hybrid connectivity (VPN, Interconnect)
└── Labs: Cloud Armor rules, Cloud IDS, HA VPN

Week 4: Data Protection
├── Cloud KMS & CMEK deep dive
├── Secret Manager
├── Sensitive Data Protection (DLP)
└── Labs: KMS, CMEK, Secret Manager, DLP

Week 5: Container & Application Security
├── GKE security (Workload Identity, Network Policy)
├── Binary Authorization
├── VPC Service Controls
└── Labs: GKE security, Binary Auth, VPC SC

Week 6: Security Operations
├── SCC (SHA, ETD, CTD, VMTD)
├── Cloud Logging & audit logs
├── Incident response procedures
└── Labs: SCC, log sinks, log-based alerts

Week 7: Compliance & Advanced Topics
├── Assured Workloads
├── Compliance frameworks (PCI, HIPAA, FedRAMP)
├── Supply chain security (SLSA, Cloud Build)
└── Review Confidential Computing

Week 8: Exam Prep
├── Full practice exams (aim for 80%+ before booking)
├── Review weak areas from practice tests
├── Redo failed labs
└── Read exam tips & question strategies
```

---

## Quick Reference: IAM Role Mapping

| Need | Role |
|------|------|
| Read-only GCS | `roles/storage.objectViewer` |
| Read GCS + list buckets | `roles/storage.legacyBucketReader` |
| Use KMS to encrypt/decrypt | `roles/cloudkms.cryptoKeyEncrypterDecrypter` |
| Access secrets | `roles/secretmanager.secretAccessor` |
| View SCC findings | `roles/securitycenter.findingsViewer` |
| Admin SCC | `roles/securitycenter.admin` |
| Impersonate service account | `roles/iam.serviceAccountTokenCreator` |
| Log sink admin | `roles/logging.configWriter` |
| Network admin | `roles/compute.networkAdmin` |
| Security admin | `roles/compute.securityAdmin` |
| Org policy admin | `roles/orgpolicy.policyAdmin` |
| Access Transparency viewer | `roles/axt.viewer` |
| IAP-secured resource access | `roles/iap.httpsResourceAccessor` |

---

## Architecture Patterns for Exam

### Pattern 1: Secure Multi-Tenant GCP Landing Zone
```
Organization
├── Shared Services Folder
│   ├── Networking Project (Shared VPC Host)
│   ├── Security Project (SCC, Logging)
│   └── KMS Project (centralized keys)
├── Production Folder (Assured Workloads)
│   └── Service Projects (Shared VPC Service)
└── Non-Production Folder
    └── Service Projects
```

### Pattern 2: Zero-Trust Access Architecture
```
User → BeyondCorp/IAP → Cloud Armor → Load Balancer → Backend
                ↑
         Access Level (device trust + identity)
```

### Pattern 3: Data Exfiltration Prevention
```
VPC Service Perimeter:
  Protected Resources: BigQuery, GCS, Spanner
  Access Levels: Corp network IP + managed device
  Ingress: Allow from org users via access level
  Egress: Deny all to external projects
```

### Pattern 4: Defense-in-Depth Network Security
```
Internet → Cloud Armor (L7 WAF/DDoS)
         → HTTPS Load Balancer (SSL policy: TLS 1.2+)
         → IAP (identity verification)
         → Firewall Rules (SA-based ingress)
         → Private GKE Cluster (Workload Identity)
         → Cloud SQL (Auth Proxy, private IP)
```

---

## Mnemonics and Memory Aids

| Concept | Memory Aid |
|---------|-----------|
| Audit log types | **ADSP** — Admin, Data access, System event, Policy denied |
| KMS key purposes | **ESM** — Encrypt/decrypt, Sign/verify, MAC signing |
| SCC built-in sources | **SHEC-RV** — Security Health, Event Threat, Container, VM Threat Detection, Rapid Vulnerability |
| DLP de-id techniques | **MRTPGC** — Mask, Redact, Tokenize, Pseudonymize, Generalize, Crypto hash |
| VPC SC perimeter types | **RB** — Regular, Bridge |
| Cloud Armor rule actions | **ADTR** — Allow, Deny, Throttle, Redirect |
| IAM policy hierarchy | **OFPR** — Org → Folder → Project → Resource |

---

*Last updated: May 2026 | Covers exam guide revision Q1 2026*  
*Always verify against the [official exam guide](https://cloud.google.com/learn/certification/cloud-security-engineer) before your exam date.*
