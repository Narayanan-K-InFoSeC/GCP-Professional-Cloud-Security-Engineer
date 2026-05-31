# 12 — Sensitive Data Protection (Cloud DLP)

> **Domain 3 | Weight: ~4% of total exam**  
> **Time:** ~3 hours concept + 1.5 hours hands-on

---

## Concepts

### 1. What is Sensitive Data Protection?

Formerly called **Cloud DLP (Data Loss Prevention)**, now officially called **Sensitive Data Protection**.

Core capabilities:
1. **Inspect** — find sensitive data (PII, PAN, PHI, credentials)
2. **De-identify** — transform sensitive data (mask, tokenize, redact)
3. **Re-identify** — reverse certain de-identification for authorized access
4. **Classify** — categorize data by sensitivity and risk
5. **Discover** — continuously scan and profile your data estate

---

### 2. InfoTypes

**InfoTypes** are the categories of sensitive data DLP detects.

**Built-in InfoTypes (examples):**

| InfoType | Detects |
|----------|---------|
| `PERSON_NAME` | Full names |
| `EMAIL_ADDRESS` | Email addresses |
| `PHONE_NUMBER` | Phone numbers |
| `US_SOCIAL_SECURITY_NUMBER` | US SSNs |
| `CREDIT_CARD_NUMBER` | Credit card numbers |
| `US_DRIVERS_LICENSE_NUMBER` | Driver's license numbers |
| `US_PASSPORT` | Passport numbers |
| `IP_ADDRESS` | IP addresses |
| `DATE_OF_BIRTH` | Dates of birth |
| `IBAN_CODE` | Bank account numbers |
| `SWIFT_CODE` | SWIFT/BIC codes |
| `AWS_CREDENTIALS` | AWS access keys |
| `GCP_CREDENTIALS` | GCP service account keys |
| `AUTH_TOKEN` | Auth tokens |
| `PASSWORD` | Password strings |
| `MEDICAL_RECORD_NUMBER` | Medical record IDs |
| `GENERIC_ID` | Alphanumeric IDs |

**Custom InfoTypes:**
- **Dictionary** — match a list of specific words/phrases
- **Regex** — match patterns (e.g., `EMP-[0-9]{6}` for employee IDs)
- **Large custom dictionary** — millions of words (stored in GCS or BigQuery)
- **Stored InfoType** — pre-compiled large dictionary for performance

---

### 3. De-identification Techniques

| Technique | What Happens | Reversible? | Use Case |
|-----------|-------------|-------------|---------|
| **Masking** | Replace chars with `*` or `#` | No | Display/logs (SSN → `***-**-6789`) |
| **Redaction** | Replace with `[REDACTED]` or empty | No | Removing PII entirely |
| **Replacement** | Replace with fixed value | No | Replace email with `user@example.com` |
| **Hashing** | SHA-256 hash | No | One-way anonymization |
| **Tokenization (FPE)** | Format-preserving encryption | Yes (with key) | Analytics — preserves format |
| **Pseudonymization** | Deterministic encryption | Yes (with key) | Re-linkable anonymization |
| **Date shifting** | Shift dates by random offset | Yes (with context) | Preserve time relationships |
| **Bucketing** | Generalize values into ranges | No | Age → `30-40` |
| **Cryptographic hashing** | HMAC hash | Yes (with key) | Pseudonymous IDs |

**Format-Preserving Encryption (FPE/FFX):**
- Encrypted SSN `123-45-6789` looks like another valid SSN `987-65-4321`
- Database schema stays the same
- De-identified data still passes validation checks

---

### 4. Inspection Jobs — Scanning Storage

**Where DLP can scan:**
- Cloud Storage (GCS buckets, objects, file formats)
- BigQuery (datasets, tables, rows)
- Datastore/Firestore (entity kinds)
- Text/JSON/CSV/Images (inline content)

**Job types:**
- **Inspection job** — find sensitive data, generate findings
- **Risk analysis job** — compute statistical re-identification risk (k-anonymity, l-diversity, t-closeness)

---

### 5. Data Profiles (Discovery Service)

**Discovery** (org-wide, 2023+ feature):
- Continuously scans all BigQuery datasets and tables across your org
- Generates **data profiles** — sensitivity, risk level, info types found
- No manual job scheduling needed
- Results flow into Security Command Center and Cloud Logging

**Data risk levels:** `LOW`, `MODERATE`, `HIGH`  
**Data sensitivity levels:** `LOW`, `MODERATE`, `HIGH`

---

### 6. Re-identification Risk Analysis

Measures how easy it is to re-identify individuals from de-identified data.

**k-anonymity:** Every record looks like at least k-1 other records.  
**l-diversity:** Each equivalence class has at least l distinct sensitive values.  
**t-closeness:** Distribution of sensitive attribute in each group mirrors the overall distribution.

---

## gcloud and API Commands

### Inspect Content (Quick Test)
```bash
# Inspect a string for PII
gcloud dlp text inspect \
  --content="My SSN is 123-45-6789 and email is john@example.com" \
  --info-types=US_SOCIAL_SECURITY_NUMBER,EMAIL_ADDRESS

# Inspect a file
gcloud dlp text inspect \
  --content-file=sample-data.txt \
  --info-types=CREDIT_CARD_NUMBER,PHONE_NUMBER,PERSON_NAME \
  --min-likelihood=POSSIBLE
```

### De-identify Content
```bash
# Mask SSN (replace digits with *)
gcloud dlp text de-identify \
  --content="SSN: 123-45-6789" \
  --info-types=US_SOCIAL_SECURITY_NUMBER \
  --masking-character="*" \
  --number-to-mask=0

# Redact email addresses
gcloud dlp text de-identify \
  --content="Contact john@example.com for support" \
  --info-types=EMAIL_ADDRESS \
  --replace-with-info-type
```

### Create a DLP Inspection Job (Cloud Storage)
```bash
# Using the API via curl (gcloud dlp has limited job support)
ACCESS_TOKEN=$(gcloud auth print-access-token)
PROJECT_ID=$(gcloud config get-value project)

# Create inspection job for a GCS bucket
curl -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://dlp.googleapis.com/v2/projects/$PROJECT_ID/dlpJobs" \
  -d '{
    "inspectJob": {
      "storageConfig": {
        "cloudStorageOptions": {
          "fileSet": {
            "url": "gs://my-bucket/**"
          },
          "fileTypes": ["TEXT_FILE", "CSV", "JSON"]
        }
      },
      "inspectConfig": {
        "infoTypes": [
          {"name": "US_SOCIAL_SECURITY_NUMBER"},
          {"name": "CREDIT_CARD_NUMBER"},
          {"name": "EMAIL_ADDRESS"},
          {"name": "PHONE_NUMBER"}
        ],
        "minLikelihood": "LIKELY",
        "limits": {
          "maxFindingsPerRequest": 100
        },
        "includeQuote": false
      },
      "actions": [
        {
          "saveFindings": {
            "outputConfig": {
              "table": {
                "projectId": "'"$PROJECT_ID"'",
                "datasetId": "dlp_results",
                "tableId": "inspection_findings"
              }
            }
          }
        },
        {
          "publishSummaryToCscc": {}
        }
      ]
    }
  }'

# List DLP jobs
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  "https://dlp.googleapis.com/v2/projects/$PROJECT_ID/dlpJobs"
```

### Create DLP Templates (Reusable Configs)
```bash
# Create an inspect template
curl -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://dlp.googleapis.com/v2/projects/$PROJECT_ID/inspectTemplates" \
  -d '{
    "inspectTemplate": {
      "displayName": "PII Template",
      "description": "Standard PII detection",
      "inspectConfig": {
        "infoTypes": [
          {"name": "US_SOCIAL_SECURITY_NUMBER"},
          {"name": "CREDIT_CARD_NUMBER"},
          {"name": "EMAIL_ADDRESS"},
          {"name": "DATE_OF_BIRTH"},
          {"name": "PERSON_NAME"},
          {"name": "PHONE_NUMBER"},
          {"name": "US_PASSPORT"}
        ],
        "minLikelihood": "LIKELY"
      }
    }
  }'

# Create a de-identify template (masking SSN)
curl -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://dlp.googleapis.com/v2/projects/$PROJECT_ID/deidentifyTemplates" \
  -d '{
    "deidentifyTemplate": {
      "displayName": "Mask SSN",
      "deidentifyConfig": {
        "infoTypeTransformations": {
          "transformations": [
            {
              "infoTypes": [{"name": "US_SOCIAL_SECURITY_NUMBER"}],
              "primitiveTransformation": {
                "characterMaskConfig": {
                  "maskingCharacter": "*",
                  "numberToMask": 5,
                  "maskingDirection": "LEFT_TO_RIGHT"
                }
              }
            },
            {
              "infoTypes": [{"name": "EMAIL_ADDRESS"}],
              "primitiveTransformation": {
                "replaceWithInfoTypeConfig": {}
              }
            }
          ]
        }
      }
    }
  }'
```

### Discovery Scan (Data Profiles)
```bash
# Create a discovery configuration (org-wide scan)
curl -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  "https://dlp.googleapis.com/v2/projects/$PROJECT_ID/locations/us/discoveryConfigs" \
  -d '{
    "config": {
      "displayName": "Org-wide BigQuery Discovery",
      "orgConfig": {
        "projectId": "'"$PROJECT_ID"'"
      },
      "targets": [
        {
          "bigQueryTarget": {
            "filter": {
              "otherTables": {}
            }
          }
        }
      ],
      "actions": [
        {"exportData": {"profileTable": {
          "projectId": "'"$PROJECT_ID"'",
          "datasetId": "dlp_profiles",
          "tableId": "data_profiles"
        }}},
        {"publishToChronicle": {}},
        {"publishToCscc": {"minimumSensitivity": "MODERATE"}}
      ],
      "status": "RUNNING"
    }
  }'
```

### Working with DLP via Python SDK
```python
# Install: pip install google-cloud-dlp

from google.cloud import dlp_v2

def inspect_string(project_id, content, info_types):
    client = dlp_v2.DlpServiceClient()
    
    inspect_config = {
        "info_types": [{"name": t} for t in info_types],
        "min_likelihood": dlp_v2.Likelihood.POSSIBLE,
        "include_quote": True,
    }
    
    item = {"value": content}
    parent = f"projects/{project_id}/locations/global"
    
    response = client.inspect_content(
        request={
            "parent": parent,
            "inspect_config": inspect_config,
            "item": item,
        }
    )
    
    for finding in response.result.findings:
        print(f"Found {finding.info_type.name}: likelihood={finding.likelihood.name}")
        if finding.quote:
            print(f"  Quote: {finding.quote}")

# Usage
inspect_string(
    "my-project",
    "SSN: 123-45-6789, Email: john@example.com, CC: 4532-1234-5678-9012",
    ["US_SOCIAL_SECURITY_NUMBER", "EMAIL_ADDRESS", "CREDIT_CARD_NUMBER"]
)
```

---

## Hands-On Practice

### Exercise 1: De-identify a CSV Before Storing in BigQuery

**Scenario:** You receive CSV data with SSNs, emails, and names. De-identify before storing.

```python
from google.cloud import dlp_v2

def deidentify_csv(project_id, input_csv, output_file):
    client = dlp_v2.DlpServiceClient()
    parent = f"projects/{project_id}/locations/global"
    
    deidentify_config = {
        "info_type_transformations": {
            "transformations": [
                {
                    "info_types": [{"name": "US_SOCIAL_SECURITY_NUMBER"}],
                    "primitive_transformation": {
                        "crypto_replace_ffx_fpe_config": {
                            "crypto_key": {
                                "unwrapped": {
                                    "key": b"\x00" * 32  # Use real key in production
                                }
                            },
                            "common_alphabet": "NUMERIC",
                        }
                    }
                },
                {
                    "info_types": [{"name": "EMAIL_ADDRESS"}],
                    "primitive_transformation": {
                        "replace_with_info_type_config": {}
                    }
                },
                {
                    "info_types": [{"name": "PERSON_NAME"}],
                    "primitive_transformation": {
                        "character_mask_config": {
                            "masking_character": "X",
                            "number_to_mask": 0,  # 0 = mask all
                        }
                    }
                },
            ]
        }
    }
    
    inspect_config = {
        "info_types": [
            {"name": "US_SOCIAL_SECURITY_NUMBER"},
            {"name": "EMAIL_ADDRESS"},
            {"name": "PERSON_NAME"},
        ]
    }
    
    with open(input_csv) as f:
        table_csv = f.read()
    
    item = {"byte_item": {"type_": dlp_v2.ByteContentItem.BytesType.TEXT_UTF8, "data": table_csv.encode()}}
    
    response = client.deidentify_content(
        request={
            "parent": parent,
            "deidentify_config": deidentify_config,
            "inspect_config": inspect_config,
            "item": item,
        }
    )
    
    with open(output_file, "wb") as f:
        f.write(response.item.byte_item.data)
    
    print(f"De-identified data written to {output_file}")
```

### Exercise 2: Scan GCS Bucket for PII

```bash
# Create a sample file with PII
cat > pii-sample.txt << 'EOF'
Customer: John Smith
SSN: 123-45-6789
Email: john.smith@example.com
Credit Card: 4532-1234-5678-9012
Phone: +1-800-555-1234
EOF

# Upload to GCS
gcloud storage buckets create gs://dlp-scan-test-$RANDOM --location=us-central1
gcloud storage cp pii-sample.txt gs://dlp-scan-test-$(gcloud config get-value project)/

# Quick inspect via CLI
gcloud dlp text inspect \
  --content-file=pii-sample.txt \
  --info-types=PERSON_NAME,US_SOCIAL_SECURITY_NUMBER,EMAIL_ADDRESS,CREDIT_CARD_NUMBER,PHONE_NUMBER \
  --min-likelihood=POSSIBLE
```

---

## Review Questions

1. Explain the difference between **masking** and **tokenization (FPE)**. When would you choose each?

2. A compliance team needs to share a BigQuery table containing SSNs with an analytics team. The analytics team should not see actual SSNs, but the data should maintain its format for validation. Which de-identification technique is most appropriate?

3. What is the difference between an **inspection job** and **discovery** in Sensitive Data Protection?

4. What is **re-identification risk** and what techniques does DLP use to measure it?

5. Your application needs to scan user-submitted text for PII before storing it. Should you use an inspection job or call the DLP API inline? Why?

---

## Key Exam Points

- **Sensitive Data Protection = Cloud DLP** (new name same service)
- **InfoTypes** are the detection templates — built-in (300+) and custom
- **FPE/Tokenization** is reversible with the correct key — preserves format
- **Redaction** is irreversible — data is removed/replaced
- **`minLikelihood`** threshold matters — `LIKELY` reduces false positives vs `POSSIBLE`
- **Discovery** is automatic and org-wide — inspection jobs are manual/scheduled
- **DLP data access logs** must be enabled separately — not on by default
- **Risk analysis jobs** compute k-anonymity etc. on BigQuery — they don't modify data
- **Custom InfoTypes** extend detection — regex for employee IDs, dictionaries for sensitive terms
