# 10 — Encryption & Cloud Key Management Service (KMS)

> **Domain 3 | Weight: ~6% of total exam**  
> **Time:** ~4 hours concept + 2 hours hands-on

---

## Concepts

### 1. Encryption Hierarchy in GCP

```
Data Encryption Key (DEK)       — encrypts actual data (AES-256)
     ↓ encrypted by
Key Encryption Key (KEK)        — encrypts the DEK
     ↓ stored in
Cloud KMS / Cloud HSM / EKM     — where KEK lives
```

Google uses **envelope encryption** — data is encrypted with a DEK, and the DEK is encrypted with a KEK stored in KMS.

---

### 2. Encryption at Rest — Options

| Option | Key Location | Who Controls Key | Use Case |
|--------|-------------|-----------------|---------|
| **Google Default** | Google-managed | Google | Most services, no config needed |
| **CMEK** | Cloud KMS | You (via IAM on KMS) | Compliance, audit key usage |
| **CSEK** | You supply per-request | You (but Google sees it) | Specific objects |
| **Cloud HSM** | FIPS 140-2 Level 3 HSM | You | Regulatory HSM requirement |
| **Cloud EKM** | External (Thales, Fortanix) | You (keys never leave your system) | Sovereignty, true external control |

---

### 3. Cloud KMS Key Hierarchy

```
Project
└── Key Ring (regional — cannot be deleted)
    └── CryptoKey (the key itself)
        └── CryptoKey Version (actual key material)
            ├── Primary (used for new encrypt operations)
            ├── Enabled (can decrypt old data)
            ├── Disabled (cannot encrypt/decrypt)
            └── Destroyed (key material deleted, metadata remains)
```

**Key ring cannot be deleted** — plan your naming carefully.

---

### 4. CryptoKey Purposes

| Purpose | Used For |
|---------|---------|
| `ENCRYPT_DECRYPT` | Symmetric encryption (AES-256) |
| `ASYMMETRIC_SIGN` | RSA/EC signing |
| `ASYMMETRIC_DECRYPT` | RSA decryption |
| `MAC` | HMAC message authentication |

---

### 5. Key Rotation

**Automatic rotation:**
- Set a `rotation-period` on the key
- A new key version becomes Primary automatically
- Old versions remain enabled (for decryption of old data)
- Google recommends: 90 days

**Manual rotation:**
- Create a new version manually
- Set it as Primary
- Disable and schedule destruction of old versions

**Destroy delay:** Minimum 24 hours (default) — configurable up to 120 days. Once destroyed, data encrypted with that version is unrecoverable.

---

### 6. CMEK — Customer Managed Encryption Keys

You create and control the KEK in Cloud KMS. GCP services use this key to encrypt the DEK before storing.

**Services that support CMEK:**
- Cloud Storage, BigQuery, Compute Engine (persistent disks)
- Cloud SQL, Cloud Spanner, Bigtable, Firestore
- GKE, Cloud Run, Artifact Registry
- Secret Manager, Cloud Logging, Pub/Sub
- Cloud Composer, Dataflow, Dataproc

**If you delete or disable the CMEK key:** GCP cannot decrypt the data — it becomes inaccessible.

---

### 7. Cloud HSM

**Hardware Security Module** — key material stored in a physical HSM chip.

- **FIPS 140-2 Level 3** validated
- Keys are non-exportable (cannot be extracted from HSM)
- Operations happen within the HSM hardware
- Higher cost than software keys
- Same API as regular Cloud KMS

**When required:** PCI DSS, HIPAA, FedRAMP High, government requirements.

---

### 8. Cloud External Key Manager (Cloud EKM)

Keys are stored and managed **outside** GCP (in your own key management system).

- GCP calls your external KMS for every encrypt/decrypt operation
- Keys never leave your external system
- **Key Access Justifications** — Google provides a reason for each key usage request; you can approve or deny
- If your external KMS is unavailable → GCP cannot encrypt/decrypt → data inaccessible

**External KMS providers:** Thales, Fortanix, Entrust, Equinix SmartKey.

---

### 9. Key Access Justifications (KAJ)

An extension of Cloud EKM where:
- Every GCP request to use your external key includes a **justification reason**
- Your external KMS can auto-approve or manual-review based on the reason
- You can deny access even to Google SREs
- Reasons include: `CUSTOMER_INITIATED_ACCESS`, `GOOGLE_INITIATED_SYSTEM_OPERATION`, `THIRD_PARTY_DATA_REQUEST_LEGAL_PROCESS`

---

### 10. CSEK — Customer Supplied Encryption Keys

You provide the raw AES-256 key as part of the API request.

- The key is used by GCP to encrypt/decrypt your data
- The key is **not stored by GCP** — you must supply it every time
- Available for: Cloud Storage, Compute Engine disks
- If you lose the key → data is permanently inaccessible

**Difference from CMEK:** With CSEK, GCP never stores the key. With CMEK, GCP stores the key (in KMS) but you control access.

---

### 11. Encryption in Transit

- All GCP services use **TLS 1.2+** by default
- **ALTS (Application Layer Transport Security)** — Google's internal mutual auth protocol
- **SSL policies** on Load Balancers — enforce minimum TLS version and cipher suites
- **MACsec** — Layer 2 encryption on Dedicated Interconnect

**SSL Policy profiles:**
- `COMPATIBLE` — widest support, allows TLS 1.0+
- `MODERN` — TLS 1.2+ (recommended minimum)
- `RESTRICTED` — TLS 1.2+ with strong cipher suites only
- `CUSTOM` — you pick specific TLS versions and ciphers

---

### 12. Certificate Authority Service (CAS)

Create your own **private Certificate Authority** in GCP:
- Issue TLS certificates for internal services
- Custom CA hierarchy (root → subordinate → leaf)
- Integrate with Kubernetes (cert-manager), VM provisioning
- Audit logs for all certificate issuances

---

## gcloud Commands

### Key Rings and Keys
```bash
# Enable Cloud KMS API
gcloud services enable cloudkms.googleapis.com

# Create a key ring (regional)
gcloud kms keyrings create my-keyring \
  --location=us-central1

# Create a key ring in global location (for global services)
gcloud kms keyrings create global-keyring \
  --location=global

# List key rings
gcloud kms keyrings list --location=us-central1

# Create a symmetric encryption key
gcloud kms keys create my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --purpose=encryption \
  --rotation-period=90d \
  --next-rotation-time=2026-08-01T00:00:00Z

# Create an HSM key (FIPS 140-2 Level 3)
gcloud kms keys create my-hsm-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --purpose=encryption \
  --protection-level=hsm

# Create an asymmetric signing key (for Binary Auth, etc.)
gcloud kms keys create my-signing-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --purpose=asymmetric-signing \
  --default-algorithm=rsa-sign-pkcs1-4096-sha512

# List all keys in a key ring
gcloud kms keys list \
  --keyring=my-keyring \
  --location=us-central1

# Describe a key
gcloud kms keys describe my-aes-key \
  --keyring=my-keyring \
  --location=us-central1
```

### Key Versions
```bash
# List key versions
gcloud kms keys versions list \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1

# Create a new key version (manual rotation)
gcloud kms keys versions create \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1

# Set a specific version as primary
gcloud kms keys versions update 2 \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --primary

# Disable a key version (can re-enable)
gcloud kms keys versions disable 1 \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1

# Schedule key version for destruction (24h delay by default)
gcloud kms keys versions destroy 1 \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1

# Restore a key version (before destruction completes)
gcloud kms keys versions restore 1 \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1

# Get key version state
gcloud kms keys versions describe 1 \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --format="get(state,destroyEventTime)"
```

### KMS IAM — Grant Access to Keys
```bash
# Grant encrypt/decrypt access to a service account
gcloud kms keys add-iam-policy-binding my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Grant encrypt-only (write path)
gcloud kms keys add-iam-policy-binding my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:writer-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypter"

# Grant decrypt-only (read path)
gcloud kms keys add-iam-policy-binding my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:reader-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyDecrypter"

# View key IAM policy
gcloud kms keys get-iam-policy my-aes-key \
  --keyring=my-keyring \
  --location=us-central1

# Grant KMS admin (manage keys — NOT encrypt/decrypt)
gcloud kms keyrings add-iam-policy-binding my-keyring \
  --location=us-central1 \
  --member="user:kms-admin@example.com" \
  --role="roles/cloudkms.admin"
```

### Encrypt and Decrypt Data
```bash
# Encrypt a file
gcloud kms encrypt \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --plaintext-file=secret.txt \
  --ciphertext-file=secret.txt.enc

# Decrypt a file
gcloud kms decrypt \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --ciphertext-file=secret.txt.enc \
  --plaintext-file=secret_decrypted.txt

# Encrypt a string (base64 encode first)
echo -n "my secret data" | base64 | gcloud kms encrypt \
  --key=my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --plaintext-file=- \
  --ciphertext-file=- | base64
```

### CMEK for Cloud Storage
```bash
# Set default CMEK key for a bucket
gcloud storage buckets create gs://my-encrypted-bucket \
  --location=us-central1 \
  --default-kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-aes-key

# Update an existing bucket to use CMEK
gcloud storage buckets update gs://existing-bucket \
  --default-kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-aes-key

# Grant GCS service account permission to use the key
GCS_SA=$(gcloud storage service-agent --project=PROJECT_ID)
gcloud kms keys add-iam-policy-binding my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:${GCS_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

### CMEK for BigQuery
```bash
# Create a BigQuery dataset with CMEK
bq mk \
  --dataset \
  --default_kms_key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-aes-key \
  PROJECT_ID:my_encrypted_dataset

# Grant BigQuery SA permission to use the key
BQ_SA="bq-$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')@bigquery-encryption.iam.gserviceaccount.com"
gcloud kms keys add-iam-policy-binding my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:${BQ_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

### CMEK for Compute Engine Disks
```bash
# Create a VM with CMEK-encrypted boot disk
gcloud compute instances create my-encrypted-vm \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --boot-disk-kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-aes-key

# Create an encrypted persistent disk
gcloud compute disks create my-encrypted-disk \
  --zone=us-central1-a \
  --size=100GB \
  --kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-aes-key
```

### Import External Keys (BYOK)
```bash
# Create an import job
gcloud kms import-jobs create my-import-job \
  --keyring=my-keyring \
  --location=us-central1 \
  --import-method=rsa-oaep-3072-sha1-aes-256 \
  --protection-level=software

# Get the import job public key (used to wrap your key)
gcloud kms import-jobs describe my-import-job \
  --keyring=my-keyring \
  --location=us-central1 \
  --format="value(publicKey.pem)"

# Create target key (empty — to be filled by import)
gcloud kms keys create imported-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --purpose=encryption \
  --skip-initial-version-creation

# Import the wrapped key material
# (Wrapping done externally using the import job's public key)
gcloud kms keys versions import \
  --key=imported-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --import-job=my-import-job \
  --wrapped-key-file=wrapped-key.bin \
  --algorithm=google-symmetric-encryption
```

### SSL Policies
```bash
# Create an SSL policy (TLS 1.2+ MODERN)
gcloud compute ssl-policies create modern-ssl-policy \
  --profile=MODERN \
  --min-tls-version=TLS_1_2

# Create RESTRICTED policy
gcloud compute ssl-policies create restricted-ssl-policy \
  --profile=RESTRICTED \
  --min-tls-version=TLS_1_2

# Create CUSTOM policy
gcloud compute ssl-policies create custom-ssl-policy \
  --profile=CUSTOM \
  --min-tls-version=TLS_1_2 \
  --custom-features=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256

# Attach to target HTTPS proxy
gcloud compute target-https-proxies update my-https-proxy \
  --ssl-policy=modern-ssl-policy

# List SSL policies
gcloud compute ssl-policies list

# Check if any LBs use weak TLS
gcloud compute ssl-policies list-available-features \
  --filter="minTlsVersion=TLS_1_0"
```

---

## Hands-On Practice

### Exercise 1: Secure KMS Setup with Separation of Duties

```bash
# Security principle: Key admins cannot encrypt/decrypt; encrypters cannot manage keys

# Create key admin SA (manages keys, cannot use them)
gcloud iam service-accounts create kms-admin-sa

# Create crypto user SA (uses keys, cannot manage them)
gcloud iam service-accounts create kms-crypto-sa

# Grant admin SA only key management rights
gcloud kms keyrings add-iam-policy-binding my-keyring \
  --location=us-central1 \
  --member="serviceAccount:kms-admin-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudkms.admin"

# Grant crypto SA only encrypt/decrypt rights on specific key
gcloud kms keys add-iam-policy-binding my-aes-key \
  --keyring=my-keyring \
  --location=us-central1 \
  --member="serviceAccount:kms-crypto-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

echo "Separation of duties: admin cannot encrypt, cryptoSA cannot manage keys"
```

### Exercise 2: Set Up CMEK for Cloud Storage

```bash
PROJECT_ID=$(gcloud config get-value project)
LOCATION=us-central1
KEYRING=storage-keyring
KEY=storage-cmek-key
BUCKET="${PROJECT_ID}-encrypted-test"

# Create keyring and key
gcloud kms keyrings create $KEYRING --location=$LOCATION
gcloud kms keys create $KEY \
  --keyring=$KEYRING \
  --location=$LOCATION \
  --purpose=encryption \
  --rotation-period=90d

# Get GCS service account
GCS_SA=$(gcloud storage service-agent --project=$PROJECT_ID)
echo "GCS SA: $GCS_SA"

# Grant GCS SA access to key
gcloud kms keys add-iam-policy-binding $KEY \
  --keyring=$KEYRING \
  --location=$LOCATION \
  --member="serviceAccount:${GCS_SA}" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"

# Create encrypted bucket
gcloud storage buckets create gs://$BUCKET \
  --location=$LOCATION \
  --default-kms-key=projects/${PROJECT_ID}/locations/${LOCATION}/keyRings/${KEYRING}/cryptoKeys/${KEY}

# Upload a test file
echo "Secret data" > test.txt
gcloud storage cp test.txt gs://$BUCKET/test.txt

# Verify encryption key used
gcloud storage objects describe gs://$BUCKET/test.txt \
  --format="get(kmsKeyName)"
```

---

## Review Questions

1. A compliance requirement states that encryption keys must be stored in a FIPS 140-2 Level 3 HSM. Which GCP encryption option satisfies this?

2. Your team uses CMEK for Cloud Storage. The security team accidentally destroys the KMS key version used to encrypt the data. What happens to the data? Can it be recovered?

3. Explain **envelope encryption** in GCP. Why is it used instead of encrypting data directly with the KMS key?

4. What is the difference between **CMEK** and **CSEK**? Which gives you more control? Which gives GCP more visibility into your key?

5. You want different teams to manage keys and use keys, but no team should be able to do both. What IAM roles do you assign to each team?

---

## Key Exam Points

- **FIPS 140-2 Level 3** = Cloud HSM — not software keys
- **Cloud EKM** = keys truly external, Google never has them
- **Deleting a CMEK key** = data is unrecoverable — 24h destroy delay is your safety net
- **KMS key admin** (`roles/cloudkms.admin`) can manage keys but CANNOT encrypt/decrypt
- **`cryptoKeyEncrypterDecrypter`** can use keys but CANNOT rotate, create, or delete
- **Key rotation doesn't re-encrypt old data** — old versions must remain enabled to decrypt old ciphertext
- **CSEK is per-request** — must be supplied every API call; GCP does not store it
- **KMS key ring is regional** — must be in same region as the data it protects (or `global` for global services)
- **Access Transparency** shows Google staff access; **KAJ** lets you approve/deny those requests
