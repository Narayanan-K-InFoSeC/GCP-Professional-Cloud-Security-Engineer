# GCP Professional Cloud Security Engineer — Study Guide Index 2026

> Work through each file in order. Every file has: **Concepts → gcloud Commands → Hands-On Practice → Review Questions**

---

## How to Use This Guide

1. Read the concept section first
2. Run every `gcloud` command in Cloud Shell or your terminal
3. Complete the hands-on practice exercises
4. Answer the review questions before moving to the next file
5. Check off each file when done

```
gcloud config set project YOUR_PROJECT_ID   # Set your project before starting
gcloud auth login                           # Authenticate
```

---

## Study Files — In Order

| # | File | Domain | Topics |
|---|------|--------|--------|
| 01 | [IAM Fundamentals](01-IAM-Fundamentals.md) | 1 | Roles, policies, conditions, deny policies |
| 02 | [Service Accounts & Workload Identity](02-Service-Accounts-and-Workload-Identity.md) | 1 | SA types, keys, WIF, impersonation |
| 03 | [Org Policy & Resource Hierarchy](03-Org-Policy-and-Resource-Hierarchy.md) | 1 | Org/Folder/Project, constraints, custom policies |
| 04 | [IAP & BeyondCorp Zero Trust](04-IAP-and-BeyondCorp.md) | 1 | IAP, Access Context Manager, access levels |
| 05 | [VPC & Firewall Security](05-VPC-and-Firewall-Security.md) | 2 | VPC design, firewall rules, hierarchical policies |
| 06 | [Cloud Armor & WAF](06-Cloud-Armor-and-WAF.md) | 2 | DDoS, WAF rules, Adaptive Protection, rate limiting |
| 07 | [Hybrid Connectivity Security](07-Hybrid-Connectivity.md) | 2 | VPN, Interconnect, NCC, MACsec |
| 08 | [GKE Security](08-GKE-Security.md) | 2 | Workload Identity, Network Policy, Binary Auth, Sandbox |
| 09 | [Cloud IDS & Network Threat Detection](09-Cloud-IDS-and-Network-Threats.md) | 2 | IDS, Packet Mirroring, Flow Logs |
| 10 | [Encryption & Cloud KMS](10-Encryption-and-KMS.md) | 3 | CMEK, CSEK, HSM, EKM, key lifecycle |
| 11 | [Secret Manager](11-Secret-Manager.md) | 3 | Secrets, versions, rotation, CMEK |
| 12 | [Sensitive Data Protection (DLP)](12-DLP-and-Sensitive-Data-Protection.md) | 3 | InfoTypes, inspection, de-identification |
| 13 | [Storage & Database Security](13-Storage-and-Database-Security.md) | 3 | GCS, BigQuery, Cloud SQL, Spanner security |
| 14 | [VPC Service Controls](14-VPC-Service-Controls.md) | 3 | Perimeters, access levels, ingress/egress rules |
| 15 | [Audit Logging & Cloud Logging](15-Audit-Logging-and-Cloud-Logging.md) | 4 | Log types, sinks, retention, log-based alerts |
| 16 | [Security Command Center](16-Security-Command-Center.md) | 4 | SHA, ETD, CTD, VMTD, findings, posture |
| 17 | [Incident Response](17-Incident-Response.md) | 4 | Detection, containment, forensics, recovery |
| 18 | [Supply Chain & Binary Authorization](18-Supply-Chain-and-Binary-Auth.md) | 4 | SLSA, Binary Auth, Artifact Registry, Cloud Build |
| 19 | [Compliance & Assured Workloads](19-Compliance-and-Assured-Workloads.md) | 5 | FedRAMP, HIPAA, PCI, Assured Workloads, CIS |
| 20 | [Exam Tips & Quick Reference](20-Exam-Tips-and-Quick-Reference.md) | All | Cheatsheet, traps, patterns, mnemonics |

---

## Domain Coverage Map

```
Domain 1 — Configuring Access (27%) ............. Files 01–04
Domain 2 — Network Security (23%) ............... Files 05–09
Domain 3 — Data Protection (20%) ................ Files 10–14
Domain 4 — Managing Operations (22%) ............ Files 15–18
Domain 5 — Compliance (8%) ...................... File  19
Exam Prep ....................................... File  20
```

## 8-Week Schedule

| Week | Files | Focus |
|------|-------|-------|
| 1 | 01–03 | IAM, Service Accounts, Org Policy |
| 2 | 04–05 | BeyondCorp, VPC & Firewalls |
| 3 | 06–07 | Cloud Armor, Hybrid Connectivity |
| 4 | 08–09 | GKE Security, Cloud IDS |
| 5 | 10–12 | KMS, Secrets, DLP |
| 6 | 13–14 | Storage/DB, VPC Service Controls |
| 7 | 15–17 | Logging, SCC, Incident Response |
| 8 | 18–20 | Supply Chain, Compliance, Exam Prep |
