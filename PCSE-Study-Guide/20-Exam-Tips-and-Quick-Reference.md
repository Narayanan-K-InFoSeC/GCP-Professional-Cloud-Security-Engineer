# 20 — Exam Tips & Quick Reference

> **Final review before exam day**  
> **Time:** 2 hours review + 2 hours practice questions

---

## Exam Strategy

### Question Approach
1. **Read the last sentence first** — it has the actual question
2. **Eliminate obviously wrong answers** — usually 1-2 are clearly incorrect
3. **Watch for "MOST secure", "BEST practice", "LEAST privileged"** — GCP always prefers the most security-conscious answer
4. **Watch for "EXCEPT" and "NOT"** — these flip the expected answer
5. **When two answers seem correct** — pick the one that follows the principle of least privilege

### Time Management
- 120 minutes for ~60 questions = 2 minutes per question
- Flag uncertain questions, come back at end
- Don't spend more than 3 minutes on any one question

### Common Question Patterns
- "A company needs to do X. What is the RECOMMENDED approach?" → Usually: keyless auth, CMEK, private connectivity
- "Your Cloud Build pipeline is failing after enabling VPC-SC. Why?" → Cloud Build SA not in ingress rules
- "You need to prevent data exfiltration from BigQuery." → VPC Service Controls
- "Which service prevents lateral movement in containers?" → Binary Authorization, GKE Sandbox
- "A key was accidentally destroyed. How do you recover the data?" → You can't. This is the point of CMEK.

---

## Critical Traps — Exam Gotchas

| Trap | Correct Answer |
|------|---------------|
| "Cloud IDS blocks threats" | IDS DETECTS only — use Cloud Armor or firewall to block |
| "CSEK keeps keys away from Google" | CSEK key passes through Google's infra — EKM truly keeps keys external |
| "VPC peering is transitive" | NOT transitive — A↔B and B↔C ≠ A↔C |
| "IAP replaces firewall rules" | You still need firewall rules to block direct VM access |
| "Destroying a KMS key destroys data" | Data is encrypted and inaccessible — but NOT deleted |
| "Data Access logs are on by default" | NO — must explicitly enable |
| "CMEK = keys external to GCP" | CMEK keys are IN Cloud KMS (Google infra). EKM = external |
| "Org policy = IAM deny" | Org policy restricts configs; IAM controls who; deny policies override allow |
| "Custom roles can be at folder level" | Custom roles only at PROJECT or ORG level — not folder |
| "VPC-SC is the same as a firewall" | VPC-SC controls API-level access; firewall controls network-level access |
| "Disabling SA revokes existing tokens" | Short-lived tokens (< 1h) may still work briefly — disable AND remove bindings |
| "`allAuthenticatedUsers` = your org only" | Any Google Account anywhere — very broad, not just your org |
| "Binary Authorization is per-image tag" | BinAuthz is per-image DIGEST (SHA256), not per tag |
| "Log retention in `_Default` = forever" | 30 days default — must configure custom bucket for longer |
| "Shared VPC shares billing" | Billing stays in service projects — only networking is shared |

---

## Service Decision Trees

### "How should X authenticate to GCP?"

```
Is X inside GCP (GCE/GKE/Cloud Run/App Engine)?
  YES → Use attached service account (metadata server)
  NO →
    Is X on Kubernetes (outside GCP)?
      YES → Workload Identity Federation (OIDC)
    Is X on GitHub Actions?
      YES → Workload Identity Federation (OIDC)
    Is X on AWS?
      YES → Workload Identity Federation (AWS STS)
    Is X a human user?
      YES → IAP + Access Context Manager
    Last resort → Service account key (avoid!)
```

### "How do I prevent data exfiltration?"

```
Need to prevent BigQuery data from being copied to external projects?
  → VPC Service Controls (service perimeter)
  
Need to prevent GCS bucket from being made public?
  → constraints/storage.publicAccessPrevention
  
Need to prevent VM from exfiltrating data outbound?
  → Egress firewall rules (deny all egress, allow specific)
  + Cloud NAT for legitimate outbound
  
Need to prevent a compromised SA from copying data?
  → VPC Service Controls + IAM least privilege
```

### "How do I encrypt data?"

```
Just need encryption, don't need to manage keys?
  → Google default encryption (nothing to do)
  
Need to control/audit key usage?
  → CMEK with Cloud KMS

Need FIPS 140-2 Level 3 hardware security?
  → Cloud HSM (Cloud KMS with HSM protection level)

Need keys to physically stay outside GCP?
  → Cloud External Key Manager (EKM)

Need to approve/deny each encryption operation?
  → Cloud EKM + Key Access Justifications

Need to provide your own key per-request?
  → CSEK (only for GCS and Compute disks)
```

### "How do I log and monitor X?"

```
Who changed IAM policies?
  → Admin Activity logs (always on, zero config)
  
Who read my GCS objects?
  → Enable Data Access logs for storage.googleapis.com

Is there active cryptomining on my VMs?
  → SCC Premium → VM Threat Detection (VMTD)
  
Are my containers being compromised at runtime?
  → SCC Premium → Container Threat Detection (CTD)
  
Do I have misconfigured resources?
  → SCC → Security Health Analytics (SHA)
  
Is there an active threat in my network traffic?
  → Cloud IDS (intrusion detection)
  
Real-time log streaming to SIEM?
  → Log sink → Pub/Sub → SIEM
```

---

## Role Quick Reference (Most Tested)

| Scenario | Role |
|---------|------|
| App reads GCS objects | `roles/storage.objectViewer` |
| App writes GCS objects | `roles/storage.objectCreator` |
| App reads/writes/deletes objects | `roles/storage.objectAdmin` |
| App accesses Cloud SQL via proxy | `roles/cloudsql.client` |
| App encrypts/decrypts with KMS | `roles/cloudkms.cryptoKeyEncrypterDecrypter` |
| App reads a secret | `roles/secretmanager.secretAccessor` |
| Service account impersonation | `roles/iam.serviceAccountTokenCreator` |
| Attach SA to a resource | `roles/iam.serviceAccountUser` |
| Pod uses GCP APIs (WIF GKE) | `roles/iam.workloadIdentityUser` |
| View SCC findings | `roles/securitycenter.findingsViewer` |
| Create BigQuery jobs + read | `roles/bigquery.jobUser` + `roles/bigquery.dataViewer` |
| Log sink access to write BigQuery | `roles/bigquery.dataEditor` |
| Log sink access to write GCS | `roles/storage.objectCreator` |
| Log sink access to write Pub/Sub | `roles/pubsub.publisher` |
| Manage KMS keys (not use them) | `roles/cloudkms.admin` |
| Use keys only (not manage) | `roles/cloudkms.cryptoKeyEncrypterDecrypter` |
| Read Access Transparency logs | `roles/axt.viewer` |
| Connect via IAP tunnel | `roles/iap.tunnelResourceAccessor` |
| Access IAP-protected web app | `roles/iap.httpsResourceAccessor` |
| Use a subnet in Shared VPC | `roles/compute.networkUser` |
| View BigQuery data (read rows) | `roles/bigquery.dataViewer` |
| Read DLP-column-protected data | `roles/datacatalog.categoryFineGrainedReader` |

---

## Org Policy Constraints Quick Reference

| Constraint | What It Prevents |
|-----------|-----------------|
| `compute.requireShieldedVm` | VMs without Shielded VM |
| `compute.vmExternalIpAccess` | VMs with external IPs |
| `compute.skipDefaultNetworkCreation` | Default VPC in new projects |
| `storage.publicAccessPrevention` | Public GCS buckets |
| `storage.uniformBucketLevelAccess` | Per-object ACLs on GCS |
| `iam.allowedPolicyMemberDomains` | IAM members from other domains |
| `iam.disableServiceAccountKeyCreation` | New SA JSON keys |
| `iam.disableServiceAccountCreation` | New service accounts |
| `gcp.resourceLocations` | Resources outside allowed regions |
| `cloudfunctions.allowedIngressSettings` | Cloud Functions public ingress |
| `run.allowedIngress` | Cloud Run public ingress |
| `compute.restrictLoadBalancerCreationForTypes` | Specific LB types |
| `compute.restrictCloudNATUsage` | Cloud NAT in disallowed locations |

---

## Firewall Key Numbers

| Value | Meaning |
|-------|--------|
| `0` | Highest priority (evaluated first) |
| `65534` | Implied allow-all-egress priority |
| `65535` | Implied deny-all-ingress priority |
| `35.235.240.0/20` | IAP TCP tunnel source range |
| `130.211.0.0/22`, `35.191.0.0/16` | Google health check source ranges |
| `199.36.153.4/30` | `restricted.googleapis.com` (VPC-SC) |
| `199.36.153.8/30` | `private.googleapis.com` |

---

## Network Service Comparison

| Service | Protects | What It Does |
|---------|---------|-------------|
| **VPC Firewall** | Network layer (L3/L4) | Allow/deny by IP, port, protocol |
| **Hierarchical Firewall** | Org/folder (L3/L4) | Same + org-wide enforcement, `goto_next` |
| **Cloud Armor** | HTTP(S) (L7) | DDoS, WAF, rate limiting at edge |
| **Cloud IDS** | Network traffic (L3-L7) | Detects threats via deep packet inspection |
| **VPC Service Controls** | GCP API level | Perimeter around GCP resource APIs |
| **IAP** | Application layer | Identity + context-based access without VPN |
| **Private Service Connect** | API connectivity | Private endpoint to Google APIs/services |

---

## Audit Log Types — Master Summary

```
Log Type          | Generated By    | Default | Paid | Can Disable
Admin Activity    | Your actions    | YES     | NO   | NO
Data Access       | Your actions    | NO      | YES  | YES  
System Event      | Google          | YES     | NO   | NO
Policy Denied     | Org policy      | YES     | NO   | YES
Access Transp.    | Google staff    | YES*    | NO   | NO
                                    (*requires SCC Premium or Workspace)
```

---

## SCC Finding Sources

| Source | Type | What It Finds |
|--------|------|---------------|
| SHA | Vulnerability/Misconfig | Static config issues (daily scan) |
| ETD | Threat | Real-time log-based attack detection |
| CTD | Threat | Runtime container attack detection |
| VMTD | Threat | VM memory-based threat detection |
| WSS | Vulnerability | Web app DAST scanning |
| RVD | Vulnerability | Network-based vulnerability scanning |

---

## Domain Weight Cheatsheet

```
Domain 1 — IAM & Access (27%) ........... Most tested!
  - IAM policies, conditions, deny policies
  - Service accounts, WIF, impersonation
  - Org policies and resource hierarchy  
  - IAP, BeyondCorp, Access Context Manager
  - PAM (just-in-time access)

Domain 4 — Operations (22%) ............. Second most!
  - Audit logging (types, enabling, sinks)
  - SCC (SHA, ETD, CTD, VMTD findings)
  - Incident response procedures
  - Supply chain (BinAuthz, Artifact Registry)

Domain 2 — Network (23%) ............... Close third
  - VPC, Shared VPC, firewall rules
  - Cloud Armor WAF/DDoS
  - Cloud IDS, Packet Mirroring
  - GKE security

Domain 3 — Data Protection (20%) ....... 
  - KMS, CMEK, HSM, EKM
  - Secret Manager
  - DLP / Sensitive Data Protection
  - VPC Service Controls

Domain 5 — Compliance (8%) ............. Shortest!
  - Assured Workloads
  - Framework mapping (PCI, HIPAA, FedRAMP)
  - Data residency
```

---

## Week 8 Practice Schedule

### Day 1-2: Take a Full Practice Exam
- Score yourself honestly
- List every question you got wrong
- Write WHY you got it wrong

### Day 3-4: Targeted Review
- Go back to study files for your weak domains
- Re-run gcloud commands for weak areas

### Day 5: Practice Exam #2
- Aim for 80%+ before booking

### Day 6: Review Common Traps
- Re-read this file
- Review "Common Exam Traps" table above

### Day 7 (Exam Day): Light Review
- Read through the domain weight cheatsheet
- Review the role quick reference
- Trust your preparation — you've got this

---

## Recommended Practice Resources

| Resource | What You Get |
|---------|-------------|
| Google Cloud Skills Boost | Official labs with real GCP environments |
| ExamTopics (PCSE section) | Community questions — verify answers carefully |
| TutorialsDojo | High-quality practice exams |
| WhizLabs | Practice exams with explanations |
| Google Cloud docs | Authoritative — use when unsure |
| `gcloud` help pages | `gcloud COMMAND --help` anytime |

---

## Quick Lab Checklist Before Exam Day

- [ ] Created a custom IAM role with specific permissions
- [ ] Set up Workload Identity Federation for GitHub Actions or another OIDC provider
- [ ] Applied an org policy constraint at org level
- [ ] Built a hierarchical firewall policy with `goto_next`
- [ ] Created a Cloud Armor security policy with WAF + rate limiting
- [ ] Set up HA VPN with BGP routing
- [ ] Enabled GKE Workload Identity on a cluster
- [ ] Created a Cloud IDS endpoint and packet mirroring
- [ ] Created a Cloud KMS key and encrypted a GCS bucket with CMEK
- [ ] Created and rotated a Secret Manager secret
- [ ] Ran a DLP inspection on a text string
- [ ] Created a VPC Service Controls perimeter in dry-run mode
- [ ] Enabled and queried Data Access audit logs
- [ ] Set up a log sink to BigQuery and queried logs with SQL
- [ ] Reviewed SCC findings and muted one
- [ ] Practiced incident response: disabled a SA, deleted its keys
- [ ] Set up Binary Authorization with an attestor
- [ ] Scanned a container image for vulnerabilities in Artifact Registry
- [ ] Created an Assured Workloads folder (or reviewed it in console)

---

*Good luck on your exam! You've covered everything — now trust your practice.*
