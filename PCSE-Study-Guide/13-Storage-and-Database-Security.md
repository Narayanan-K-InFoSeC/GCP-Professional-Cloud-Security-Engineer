# 13 — Cloud Storage & Database Security

> **Domain 3 | Weight: ~4% of total exam**  
> **Time:** ~3 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. Cloud Storage Security

#### Access Control Models

| Model | Description |
|-------|-------------|
| **Uniform bucket-level access** | IAM only — all objects in bucket use same policy |
| **Fine-grained (legacy ACL)** | Per-object ACLs + bucket IAM — avoid |

**Recommendation:** Always use **Uniform Bucket-Level Access** (`--uniform-bucket-level-access`).

Org policy to enforce this: `constraints/storage.uniformBucketLevelAccess`

#### Public Access Prevention

Blocks `allUsers` and `allAuthenticatedUsers` from being added to bucket policies.

- **Enforced** — GCP blocks adding public members
- **Inherited** — from org policy `constraints/storage.publicAccessPrevention`

#### GCS IAM Roles

| Role | Capability |
|------|-----------|
| `roles/storage.objectViewer` | Read objects |
| `roles/storage.objectCreator` | Create objects (no read, no delete) |
| `roles/storage.objectAdmin` | Full control over objects |
| `roles/storage.admin` | Full control over buckets AND objects |
| `roles/storage.legacyBucketReader` | List buckets, read bucket metadata |

#### Signed URLs and Signed Policy Documents

**Signed URLs** — time-limited, pre-authorized URL to access a specific object:
- No GCP account required — share with anyone
- Expiry is embedded in the URL
- Use service account key or `gcloud storage sign-url`

**Signed Policy Documents** — allow HTML form uploads with policy constraints (max file size, allowed content types).

#### Retention Policies and WORM

**Retention policy** — lock objects from deletion/modification for a specified period.

```bash
gcloud storage buckets update gs://my-bucket \
  --retention-period=7y  # 7 years
```

**Locking** the retention policy makes it **permanent and irreversible**:
```bash
gcloud storage buckets update gs://my-bucket \
  --lock-retention-policy
```

After locking: retention period can only be increased, never decreased or removed. This achieves **WORM (Write Once Read Many)** compliance.

#### Object Versioning

Keeps previous versions of objects when they're overwritten or deleted.
- Noncurrent versions can have their own lifecycle rules
- Enables recovery from accidental deletion

---

### 2. Cloud SQL Security

#### Connection Methods

| Method | Security | Auth |
|--------|---------|------|
| **Public IP + SSL** | Encrypted but exposed | Username/password |
| **Public IP + SSL + authorized networks** | Encrypted + IP filtered | Username/password |
| **Cloud SQL Auth Proxy** | IAM-based, encrypted tunnel | IAM roles |
| **Private IP** | No public exposure | Username/password |
| **Private IP + Auth Proxy** | Most secure | IAM + no public IP |

**Cloud SQL Auth Proxy:**
- Runs as a local proxy sidecar
- Uses IAM for authentication — no SQL password needed
- Encrypts the connection automatically
- Role required: `roles/cloudsql.client`

#### Cloud SQL IAM Database Authentication

Use Google identities (user or SA) to log into Cloud SQL:
```sql
-- Log in as: user@example.com or sa@project.iam.gserviceaccount.com
```

Role: `roles/cloudsql.instanceUser` + `GRANT` in MySQL/PostgreSQL

#### Cloud SQL Security Settings

- **Require SSL** — enforce encrypted connections
- **Authorized networks** — IP allowlist for direct public IP connections
- **CMEK** — encrypt Cloud SQL data with Cloud KMS
- **Private IP** — no public endpoint, VPC-only access
- **Point-in-time recovery** — audit/forensic recovery (logs must be retained)
- **Automatic backups** — with encryption

---

### 3. BigQuery Security

#### Column-Level Security (Policy Tags)

1. Create **taxonomy** and **policy tags** in Data Catalog
2. Assign policy tags to BigQuery columns
3. Grant `roles/datacatalog.categoryFineGrainedReader` to users who can see masked columns
4. Users without the role see `NULL` or an error when querying those columns

```sql
-- Without the role, this returns NULL for the SSN column
SELECT name, ssn FROM `project.dataset.users`
-- SSN appears as NULL for unauthorized users
```

#### Row-Level Security (Row Access Policies)

Control which rows a user/group can see:
```sql
CREATE ROW ACCESS POLICY us_customers_only
ON `project.dataset.orders`
GRANT TO ("group:us-team@example.com")
FILTER USING (country = 'US');
```

Users not in `us-team` see no rows from this table.

#### Authorized Views

Share derived data without exposing the source tables:
- Create a view that filters/aggregates sensitive data
- Grant access to the view
- Authorize the view to query the source table (the source table's ACL grants access to the view, not to the querying user)

#### BigQuery Audit Logs

Key log entries:
- `google.cloud.bigquery.v2.JobService.Query` — query execution
- `google.cloud.bigquery.v2.TableService.GetTable` — table access
- `google.cloud.bigquery.v2.JobService.InsertJob` — job creation

---

### 4. Firestore/Datastore Security

**Firestore Security Rules** (for Firestore in Native mode):
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Only authenticated users can read their own data
    match /users/{userId} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if request.auth != null && request.auth.uid == userId;
    }
    // Admin collection — only admins
    match /admin/{document=**} {
      allow read, write: if request.auth.token.admin == true;
    }
  }
}
```

---

### 5. Cloud Spanner Security

- **IAM roles:** `roles/spanner.databaseReader`, `roles/spanner.databaseAdmin`, `roles/spanner.viewer`
- **CMEK** — database-level encryption with Cloud KMS
- **VPC Service Controls** — protect Spanner APIs within a perimeter
- **Fine-grained access** — IAM conditions on specific databases

---

## gcloud Commands

### Cloud Storage Security
```bash
# Create a secure bucket (no public access, uniform ACL, CMEK)
gcloud storage buckets create gs://my-secure-bucket \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --public-access-prevention \
  --default-kms-key=projects/P/locations/us-central1/keyRings/R/cryptoKeys/K

# Enable uniform bucket-level access on existing bucket
gcloud storage buckets update gs://existing-bucket \
  --uniform-bucket-level-access

# Enable public access prevention
gcloud storage buckets update gs://existing-bucket \
  --public-access-prevention

# Grant read access to a user at bucket level
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Grant read access at object level (fine-grained only)
gcloud storage objects add-iam-policy-binding gs://my-bucket/secret-file.txt \
  --member="user:alice@example.com" \
  --role="roles/storage.legacyObjectReader"

# Set retention policy (7 years)
gcloud storage buckets update gs://my-bucket \
  --retention-period=7y

# Lock retention policy (IRREVERSIBLE)
gcloud storage buckets update gs://my-bucket \
  --lock-retention-policy

# Enable object versioning
gcloud storage buckets update gs://my-bucket \
  --versioning

# List objects including noncurrent versions
gcloud storage ls -a gs://my-bucket/

# Delete a specific version
gcloud storage rm gs://my-bucket/file.txt#GENERATION_NUMBER

# Create a signed URL (1 hour expiry) using service account impersonation
gcloud storage sign-url gs://my-bucket/private-file.pdf \
  --duration=1h \
  --impersonate-service-account=storage-sa@PROJECT_ID.iam.gserviceaccount.com

# Check bucket public access status
gcloud storage buckets describe gs://my-bucket \
  --format="get(iamConfiguration.publicAccessPrevention)"

# List buckets with public access check
gcloud storage buckets list \
  --format="table(name,iamConfiguration.uniformBucketLevelAccess.enabled,iamConfiguration.publicAccessPrevention)"
```

### Cloud SQL Security
```bash
# Create a Cloud SQL instance (PostgreSQL) with security settings
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --region=us-central1 \
  --tier=db-standard-2 \
  --no-assign-ip \
  --network=projects/PROJECT_ID/global/networks/my-vpc \
  --enable-google-private-path \
  --require-ssl

# Enable SSL on existing instance
gcloud sql instances patch my-db \
  --require-ssl

# Create SSL certificate for client auth
gcloud sql ssl client-certs create client-cert \
  --instance=my-db

# Add authorized network (if using public IP)
gcloud sql instances patch my-db \
  --authorized-networks=203.0.113.0/24

# Enable IAM database authentication
gcloud sql instances patch my-db \
  --database-flags=cloudsql.iam_authentication=on

# Create an IAM user for database login
gcloud sql users create user@example.com \
  --instance=my-db \
  --type=CLOUD_IAM_USER

# Create an IAM service account user
gcloud sql users create app-sa@PROJECT_ID.iam \
  --instance=my-db \
  --type=CLOUD_IAM_SERVICE_ACCOUNT

# Grant Cloud SQL Client role (required for Auth Proxy)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Connect using Cloud SQL Auth Proxy
# Download proxy: https://cloud.google.com/sql/docs/postgres/sql-proxy
./cloud-sql-proxy PROJECT_ID:us-central1:my-db --port=5432 &
# Then connect locally: psql -h 127.0.0.1 -U username my-database

# Check instance settings
gcloud sql instances describe my-db \
  --format="yaml(settings.ipConfiguration)"

# List instances
gcloud sql instances list \
  --format="table(name,region,databaseVersion,settings.ipConfiguration.requireSsl,settings.ipConfiguration.ipv4Enabled)"
```

### BigQuery Column-Level Security
```bash
# Create a Data Catalog taxonomy for policy tags
gcloud data-catalog taxonomies create \
  --location=us-central1 \
  --display-name="Data Classification" \
  --description="PII sensitivity levels"

# Get taxonomy ID
TAXONOMY_ID=$(gcloud data-catalog taxonomies list \
  --location=us-central1 \
  --format="value(name)" | head -1)

# Create policy tags
gcloud data-catalog taxonomies policy-tags create \
  --taxonomy=$TAXONOMY_ID \
  --display-name="PII - High Sensitivity" \
  --description="Contains directly identifiable PII"

# Grant fine-grained reader role to authorized group
POLICY_TAG_ID=$(gcloud data-catalog taxonomies policy-tags list \
  --taxonomy=$TAXONOMY_ID \
  --format="value(name)" | head -1)

gcloud data-catalog taxonomies policy-tags add-iam-policy-binding $POLICY_TAG_ID \
  --member="group:pii-readers@example.com" \
  --role="roles/datacatalog.categoryFineGrainedReader"

# In BigQuery, apply the policy tag to a column (via API or console)
# The column definition in BigQuery JSON schema:
# {
#   "name": "ssn",
#   "type": "STRING",
#   "policyTags": {
#     "names": ["POLICY_TAG_ID"]
#   }
# }
```

### BigQuery Row-Level Security
```bash
# Create a row access policy via BigQuery API
bq query --use_legacy_sql=false \
'CREATE ROW ACCESS POLICY us_only
 ON `PROJECT_ID.my_dataset.orders`
 GRANT TO ("group:us-analysts@example.com")
 FILTER USING (region = "US");'

# List row access policies
bq query --use_legacy_sql=false \
'SELECT * FROM `PROJECT_ID.my_dataset.INFORMATION_SCHEMA.ROW_ACCESS_POLICIES`'

# Drop a row access policy
bq query --use_legacy_sql=false \
'DROP ALL ROW ACCESS POLICIES ON `PROJECT_ID.my_dataset.orders`;'
```

### BigQuery Authorized Views
```bash
# Step 1: Create a view that masks sensitive columns
bq query --use_legacy_sql=false \
'CREATE VIEW `PROJECT_ID.public_dataset.safe_customers` AS
 SELECT
   customer_id,
   name,
   REGEXP_REPLACE(email, r"(.+)@", "***@") as email_masked,
   country
 FROM `PROJECT_ID.private_dataset.customers`'

# Step 2: Authorize the view to access the source table
bq update \
  --source=projects/PROJECT_ID/datasets/public_dataset/views/safe_customers \
  PROJECT_ID:private_dataset

# Step 3: Grant access to the view (not the source table)
bq add-iam-policy-binding \
  --member="group:analysts@example.com" \
  --role="roles/bigquery.dataViewer" \
  PROJECT_ID:public_dataset
```

---

## Hands-On Practice

### Exercise 1: Audit Bucket Public Access

```bash
# Find all public buckets in a project
gcloud storage buckets list --format="value(name)" | while read bucket; do
  PUBLIC=$(gcloud storage buckets get-iam-policy gs://$bucket 2>/dev/null | \
    grep -c "allUsers\|allAuthenticatedUsers")
  if [ "$PUBLIC" -gt 0 ]; then
    echo "PUBLIC BUCKET: $bucket"
  fi
done

# Fix: Enable public access prevention on all buckets
gcloud storage buckets list --format="value(name)" | while read bucket; do
  gcloud storage buckets update gs://$bucket --public-access-prevention
  echo "Fixed: $bucket"
done
```

### Exercise 2: Set Up Cloud SQL with Auth Proxy

```bash
PROJECT_ID=$(gcloud config get-value project)

# Create Cloud SQL instance (private IP)
gcloud sql instances create secure-db \
  --database-version=POSTGRES_15 \
  --region=us-central1 \
  --tier=db-f1-micro \
  --no-assign-ip \
  --network=default \
  --require-ssl

# Create database and user
gcloud sql databases create mydb --instance=secure-db
gcloud sql users create app_user --instance=secure-db --password=CHANGE_ME

# Create SA for the app
gcloud iam service-accounts create db-app-sa
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:db-app-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

echo "Connection string for Cloud SQL Auth Proxy:"
echo "${PROJECT_ID}:us-central1:secure-db"
```

---

## Review Questions

1. What is the difference between **Uniform Bucket-Level Access** and **Public Access Prevention**? Can you have both enabled?

2. A developer uploads a sensitive file to GCS with `allUsers` read access. The org policy `constraints/storage.publicAccessPrevention` is enforced. Was the upload allowed? What about the public access?

3. Explain how **Authorized Views** in BigQuery protect the source table.

4. A Cloud SQL instance has `--no-assign-ip` (no public IP) and is on a VPC. How does an application connect to it securely?

5. What is a **retention lock** in Cloud Storage? Is it reversible? When would you use it?

---

## Key Exam Points

- **Uniform Bucket-Level Access = IAM only** — disables per-object ACLs
- **Public Access Prevention** blocks `allUsers`/`allAuthenticatedUsers` — even if IAM says allow
- **WORM = retention policy + lock** — locking is irreversible!
- **Cloud SQL Auth Proxy** uses IAM, not username/password — preferred for services
- **`roles/cloudsql.client`** is required for Auth Proxy — not `roles/cloudsql.admin`
- **BigQuery column-level security** uses **policy tags** — not IAM roles directly on columns
- **Authorized Views** — the view is authorized, not the user querying the view
- **Private IP for Cloud SQL** requires VPC peering with the Google-managed service VPC
- **Signed URLs** bypass IAM — any bearer of the URL can access the object until expiry
