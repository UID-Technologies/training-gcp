# Lab 04 — Google Cloud Storage: Object Storage and Data Lifecycle

| | |
| --- | --- |
| **Training module** | Module 4 — Google Cloud Storage |
| **Level** | Beginner |
| **Estimated time** | 90–120 minutes |
| **Primary service** | Cloud Storage |
| **Lab prefix** | `cloudmart` |
| **Default location** | `asia-south1` (Mumbai) |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [Bucket design for CloudMart](#bucket-design-for-cloudmart)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Cloud Storage fundamentals](#part-1--cloud-storage-fundamentals)
8. [Part 2 — Create buckets for different workloads](#part-2--create-buckets-for-different-workloads)
9. [Part 3 — Upload and organize objects](#part-3--upload-and-organize-objects)
10. [Part 4 — Access control (uniform access and IAM)](#part-4--access-control-uniform-access-and-iam)
11. [Part 5 — Public static assets for the web store](#part-5--public-static-assets-for-the-web-store)
12. [Part 6 — Object versioning](#part-6--object-versioning)
13. [Part 7 — Lifecycle rules and storage classes](#part-7--lifecycle-rules-and-storage-classes)
14. [Part 8 — Signed URLs and Compute Engine integration](#part-8--signed-urls-and-compute-engine-integration)
15. [Part 9 — Clean up lab resources](#part-9--clean-up-lab-resources)
16. [Validation checklist](#validation-checklist)
17. [Evidence to submit](#evidence-to-submit)
18. [Knowledge check](#knowledge-check)
19. [Optional challenge](#optional-challenge)
20. [Official references](#official-references)

---

## Lab description

In this hands-on lab, you use **Google Cloud Storage** to store **CloudMart** e-commerce data: **product images**, **catalog files**, **database backups**, and **application logs**. You create buckets with appropriate **locations** and **storage classes**, upload objects using the Console and **`gcloud storage`**, configure **IAM**, enable **versioning**, define **lifecycle rules**, and generate **signed URLs** for temporary access.

This lab gives practical examples that map to real retail workloads on GCP.

**Standalone lab:** You do not need any other lab in this folder. All buckets and objects are created and removed in this lab only.

---

## Scenario and use case

### CloudMart data platform needs

| Data type | Example | Access pattern | Lab bucket role |
| --- | --- | --- | --- |
| **Product media** | `products/sku-1001/main.jpg` | Frequent read/write by merchandising tools | Private **Standard** bucket |
| **Website static files** | CSS, public thumbnails | High read, rare write; served to browsers | **Standard** with controlled public read |
| **Nightly backups** | `backups/orders-2026-11-18.sql.gz` | Write daily, read on restore | **Nearline** |
| **Older log files** | `logs/2026/10/app-*.json` | Write once, rarely read; cost-optimized | **Lifecycle** toward **Coldline** / **Archive** |

**Your role:** Design buckets and policies that balance **performance**, **cost**, and **security** for CloudMart.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain buckets, objects, prefixes, and storage classes in Cloud Storage
- Choose **location type** (region, dual-region, multi-region) for a workload
- Create buckets with **uniform bucket-level access**
- Upload, download, copy, and list objects with `gcloud storage`
- Grant access with **IAM** (and understand legacy ACLs are discouraged)
- Publish **static website assets** safely
- Enable **object versioning** and recover a previous version
- Apply **lifecycle rules** to transition or delete objects
- Generate a **V4 signed URL** for time-limited access
- Upload backups from **Compute Engine** or Cloud Shell to a bucket

---

## Bucket design for CloudMart

Bucket names must be **globally unique**. This lab uses your **project ID** in each name.

Replace `YOUR_PROJECT_ID` everywhere below.

| Logical name | Bucket name | Location | Default class | Purpose |
| --- | --- | --- | --- | --- |
| Assets (private) | `YOUR_PROJECT_ID-cloudmart-assets` | `asia-south1` | Standard | Master product images and catalog JSON |
| Static (web) | `YOUR_PROJECT_ID-cloudmart-static` | `asia-south1` | Standard | Public-read storefront assets |
| Backups | `YOUR_PROJECT_ID-cloudmart-backups` | `asia-south1` | Nearline | Simulated DB/order exports |
| Logs archive | `YOUR_PROJECT_ID-cloudmart-logs` | `asia-south1` | Standard → lifecycle | Log dumps with age-based class changes |

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export BUCKET_ASSETS="${PROJECT_ID}-cloudmart-assets"
export BUCKET_STATIC="${PROJECT_ID}-cloudmart-static"
export BUCKET_BACKUPS="${PROJECT_ID}-cloudmart-backups"
export BUCKET_LOGS="${PROJECT_ID}-cloudmart-logs"

gcloud config set project "$PROJECT_ID"
```

**Cost note:** Storage, egress, and early deletion in Nearline/Coldline can incur charges. Delete lab buckets in Part 9.

---

## Prerequisites

Before you start, confirm:

- [ ] Training project with **billing enabled**
- [ ] Roles such as **Storage Admin** or **Storage Object Admin** + **Storage Admin** on the project (or equivalent)
- [ ] Cloud Shell or local `gcloud` configured
- [ ] **Cloud Storage API** enabled (enable if needed):

```bash
gcloud services enable storage.googleapis.com
```

---

## How to work through this lab

| Part | Hands-on focus |
| --- | --- |
| 1 | Concepts |
| 2–3 | Create buckets and sample product/log/backup files |
| 4–5 | IAM and public static content |
| 6 | Versioning on catalog data |
| 7 | Lifecycle on logs |
| 8 | Signed URLs and VM upload |
| 9 | Cleanup |

---

## Part 1 — Cloud Storage fundamentals

**Time estimate:** ~10 minutes

### Step 1 — Explore Cloud Storage in the Console

1. Open [Google Cloud Console](https://console.cloud.google.com/) → **Cloud Storage → Buckets**.
2. Review any existing buckets (names are global across all GCP customers).
3. Open **Settings** or documentation links for **location** and **storage class** tooltips.

### Step 2 — Core concepts

| Concept | Description |
| --- | --- |
| **Bucket** | Top-level container; has one location and default storage class |
| **Object** | Immutable blob (file) identified by `gs://BUCKET/OBJECT_NAME` |
| **Prefix** | Logical “folder” using `/` in names (e.g. `products/sku-1001/`) |
| **Storage class** | Standard, Nearline, Coldline, Archive — affects **cost** and **minimum storage duration** |
| **Uniform bucket-level access** | IAM only (recommended); disables per-object ACL workflows |

### Step 3 — Storage classes (CloudMart mapping)

| Class | Minimum storage duration | CloudMart example |
| --- | --- | --- |
| **Standard** | None | Active product images, static web files |
| **Nearline** | 30 days | Weekly backups still restored occasionally |
| **Coldline** | 90 days | Quarterly audit exports |
| **Archive** | 365 days | Compliance archives rarely accessed |

---

## Part 2 — Create buckets for different workloads

**Time estimate:** ~20 minutes

### Step 4 — Create the private assets bucket (Console)

1. **Buckets → Create**.
2. **Name:** `YOUR_PROJECT_ID-cloudmart-assets`.
3. **Location type:** Region → **asia-south1**.
4. **Storage class:** Standard.
5. **Uniform access:** **Enforce** public access prevention if offered; otherwise enable uniform access.
6. **Soft delete / versioning:** off for now (versioning enabled in Part 6).
7. Create.

### Step 5 — Create remaining buckets (CLI)

```bash
gcloud storage buckets create "gs://${BUCKET_STATIC}" \
  --location=asia-south1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access

gcloud storage buckets create "gs://${BUCKET_BACKUPS}" \
  --location=asia-south1 \
  --default-storage-class=NEARLINE \
  --uniform-bucket-level-access

gcloud storage buckets create "gs://${BUCKET_LOGS}" \
  --location=asia-south1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access
```

Create the assets bucket via CLI if you prefer consistency:

```bash
gcloud storage buckets create "gs://${BUCKET_ASSETS}" \
  --location=asia-south1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access
```

List buckets:

```bash
gcloud storage buckets list --format="table(name,location,storageClass)"
```

---

## Part 3 — Upload and organize objects

**Time estimate:** ~20 minutes

### Step 6 — Create sample CloudMart files locally (Cloud Shell)

```bash
mkdir -p ~/cloudmart-storage-lab/products/sku-1001
mkdir -p ~/cloudmart-storage-lab/products/sku-1002
mkdir -p ~/cloudmart-storage-lab/logs/2026/11

cat > ~/cloudmart-storage-lab/catalog.json <<'EOF'
{
  "store": "CloudMart",
  "version": 1,
  "products": [
    {"sku": "1001", "name": "Wireless Earbuds", "price": 49.99},
    {"sku": "1002", "name": "USB-C Hub", "price": 29.99}
  ]
}
EOF

echo "High-resolution master image placeholder for SKU 1001" \
  > ~/cloudmart-storage-lab/products/sku-1001/master.txt

echo "High-resolution master image placeholder for SKU 1002" \
  > ~/cloudmart-storage-lab/products/sku-1002/master.txt

echo "Optimized thumbnail for web" \
  > ~/cloudmart-storage-lab/products/sku-1001/thumb-optimized.txt

cat > ~/cloudmart-storage-lab/logs/2026/11/app-2026-11-18.json <<'EOF'
{"level":"INFO","msg":"order processed","orderId":"CM-9001"}
EOF

gzip -c ~/cloudmart-storage-lab/catalog.json \
  > ~/cloudmart-storage-lab/backups/catalog-2026-11-18.json.gz
```

You can replace `.txt` placeholders with real `.jpg` files from your computer using **Console upload** or `gcloud storage cp`.

---

### Step 7 — Upload to the assets bucket (prefix layout)

```bash
gcloud storage cp -r ~/cloudmart-storage-lab/products "gs://${BUCKET_ASSETS}/"
gcloud storage cp ~/cloudmart-storage-lab/catalog.json "gs://${BUCKET_ASSETS}/config/"
```

List with prefix:

```bash
gcloud storage ls "gs://${BUCKET_ASSETS}/products/**"
gcloud storage ls -l "gs://${BUCKET_ASSETS}/config/"
```

**Why prefixes matter:** Merchandising pipelines often partition by `products/{sku}/` for IAM conditions and lifecycle rules in advanced setups.

---

### Step 8 — Upload backup and log examples

```bash
gcloud storage cp ~/cloudmart-storage-lab/backups/catalog-2026-11-18.json.gz \
  "gs://${BUCKET_BACKUPS}/database/"

gcloud storage cp ~/cloudmart-storage-lab/logs/2026/11/app-2026-11-18.json \
  "gs://${BUCKET_LOGS}/logs/2026/11/"
```

Console alternative: **Buckets → bucket → Upload files** and drag folders.

---

### Step 9 — Copy object between buckets (publish pipeline simulation)

Copy optimized thumbnail from private assets to static bucket:

```bash
gcloud storage cp \
  "gs://${BUCKET_ASSETS}/products/sku-1001/thumb-optimized.txt" \
  "gs://${BUCKET_STATIC}/storefront/sku-1001-thumb.txt"
```

This mimics a CI job that publishes web-ready assets from a private master bucket.

---

## Part 4 — Access control (uniform access and IAM)

**Time estimate:** ~15 minutes

### Step 10 — Review IAM on a private bucket

1. **Cloud Storage → Buckets →** `${BUCKET_ASSETS}` → **Permissions**.
2. Note your user has access via project-level roles or bucket-level bindings.

Grant a **read-only** role to a developer principal:

```bash
export DEV_MEMBER="user:developer@example.com"

gcloud storage buckets add-iam-policy-binding "gs://${BUCKET_ASSETS}" \
  --member="$DEV_MEMBER" \
  --role="roles/storage.objectViewer"
```

Replace with your teammate email or skip if solo.

### Step 11 — Principle of least privilege for backups

Backups bucket should not be public. Verify no `allUsers` binding:

```bash
gcloud storage buckets get-iam-policy "gs://${BUCKET_BACKUPS}" \
  --format=json
```

Optional: grant a **backup writer** service account you create for this lab:

```bash
gcloud iam service-accounts create cloudmart-backup-sa \
  --display-name="CloudMart backup writer"

gcloud storage buckets add-iam-policy-binding "gs://${BUCKET_BACKUPS}" \
  --member="serviceAccount:cloudmart-backup-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/storage.objectCreator"
```

Delete `cloudmart-backup-sa` in Part 9 if you created it.

---

## Part 5 — Public static assets for the web store

**Time estimate:** ~15 minutes

CloudMart’s storefront needs to read optimized images from a browser. In production, prefer **Cloud CDN** + **signed URLs** or **Identity-Aware Proxy** for sensitive data. For this lab, you grant **public read** only on the **dedicated static bucket** containing non-sensitive files.

### Step 12 — Allow public read on the static bucket only

**Console**

1. Open `YOUR_PROJECT_ID-cloudmart-static` → **Permissions** → **Grant access**.
2. **New principal:** `allUsers`
3. **Role:** **Storage Object Viewer** (`roles/storage.objectViewer`)
4. Save and confirm the warning about public access.

**CLI**

```bash
gcloud storage buckets add-iam-policy-binding "gs://${BUCKET_STATIC}" \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

### Step 13 — Test public URL

```bash
gcloud storage objects describe \
  "gs://${BUCKET_STATIC}/storefront/sku-1001-thumb.txt" \
  --format="value(selfLink)"
```

Open in browser (format may vary):

```text
https://storage.googleapis.com/BUCKET_STATIC/storefront/sku-1001-thumb.txt
```

Replace `BUCKET_STATIC` with your bucket name.

**Expected result:** File contents display in the browser.

**Security note:** Never place **PII**, **secrets**, or **raw database backups** in a public bucket.

---

## Part 6 — Object versioning

**Time estimate:** ~15 minutes

Product catalog changes must be recoverable. Enable **versioning** on the assets bucket.

### Step 14 — Enable versioning

```bash
gcloud storage buckets update "gs://${BUCKET_ASSETS}" --versioning
```

Console: **Bucket details → Protection → Versioning → Enable**.

### Step 15 — Update catalog and observe generations

```bash
cat > ~/cloudmart-storage-lab/catalog-v2.json <<'EOF'
{
  "store": "CloudMart",
  "version": 2,
  "products": [
    {"sku": "1001", "name": "Wireless Earbuds Pro", "price": 59.99},
    {"sku": "1002", "name": "USB-C Hub", "price": 29.99}
  ]
}
EOF

gcloud storage cp ~/cloudmart-storage-lab/catalog-v2.json \
  "gs://${BUCKET_ASSETS}/config/catalog.json"
```

List versions:

```bash
gcloud storage ls -a "gs://${BUCKET_ASSETS}/config/catalog.json"
```

Download a noncurrent version (use generation number from listing):

```bash
gcloud storage cp "gs://${BUCKET_ASSETS}/config/catalog.json#GENERATION" \
  ~/cloudmart-storage-lab/catalog-restored.json
```

Replace `GENERATION` with the older version’s generation ID.

---

## Part 7 — Lifecycle rules and storage classes

**Time estimate:** ~15 minutes

### Step 16 — Add lifecycle rule to logs bucket

Move log objects older than **30 days** to **Nearline** and delete after **365 days** (lab-friendly values).

```bash
cat > /tmp/cloudmart-logs-lifecycle.json <<'EOF'
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "ARCHIVE"},
        "condition": {"age": 90}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      }
    ]
  }
}
EOF

gcloud storage buckets update "gs://${BUCKET_LOGS}" \
  --lifecycle-file=/tmp/cloudmart-logs-lifecycle.json
```

Console: **Bucket → Lifecycle → Add rule** with the same intent.

### Step 17 — Inspect and discuss backup bucket class

The **backups** bucket was created with **Nearline** default class—appropriate for daily dumps accessed infrequently.

Upload simulates a retention policy:

```bash
gcloud storage cp ~/cloudmart-storage-lab/backups/catalog-2026-11-18.json.gz \
  "gs://${BUCKET_BACKUPS}/database/catalog-$(date -u +%Y-%m-%d).json.gz"
```

Discuss with class: restoring from Nearline may have **retrieval cost** and **latency** compared to Standard.

---

## Part 8 — Signed URLs and Compute Engine integration

**Time estimate:** ~15 minutes

### Step 18 — Generate a V4 signed URL (private object)

Time-limited access without making the assets bucket public:

```bash
gcloud storage sign-url \
  "gs://${BUCKET_ASSETS}/config/catalog.json" \
  --duration=15m \
  --http-verb=GET
```

Copy the URL into a browser **within 15 minutes**. **Expected result:** JSON downloads.

Use signed URLs for **partner integrations** or **temporary customer download links**.

---

### Step 19 — Upload from Compute Engine (optional)

From Cloud Shell (simulating a scheduled export or any VM with Storage permissions):

```bash
gcloud storage cp ~/cloudmart-storage-lab/catalog.json \
  "gs://${BUCKET_BACKUPS}/exports/manual-export-$(date -u +%Y%m%d).json"
```

**Relevant example:** Nightly **order database export** lands in `gs://...-cloudmart-backups/` for disaster recovery.

---

### Step 20 — Object metadata and content type (optional)

Set correct MIME type for a JSON file:

```bash
gcloud storage objects update "gs://${BUCKET_STATIC}/storefront/sku-1001-thumb.txt" \
  --content-type=text/plain
```

For real `.jpg` assets, use `image/jpeg` so browsers and CDNs cache correctly.

---

## Part 9 — Clean up lab resources

**Time estimate:** ~10 minutes

Buckets must be **empty** (including all object versions) before deletion.

### Step 21 — Remove public IAM binding

```bash
gcloud storage buckets remove-iam-policy-binding "gs://${BUCKET_STATIC}" \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

### Step 22 — Delete all objects and buckets

```bash
for b in "$BUCKET_ASSETS" "$BUCKET_STATIC" "$BUCKET_BACKUPS" "$BUCKET_LOGS"; do
  gcloud storage rm -r "gs://${b}/**" 2>/dev/null || true
  gcloud storage buckets delete "gs://${b}" --quiet 2>/dev/null || true
done
```

If delete fails due to versions:

```bash
gcloud storage ls -a "gs://${BUCKET_ASSETS}/**"
# Remove specific generations or use Console "Empty bucket"
```

Verify:

```bash
gcloud storage buckets list --filter="name:cloudmart"
```

Also remove developer IAM bindings you added in Part 4.

If you created `cloudmart-backup-sa` for the optional backup exercise:

```bash
gcloud iam service-accounts delete \
  "cloudmart-backup-sa@${PROJECT_ID}.iam.gserviceaccount.com" --quiet
```

---

## Validation checklist

- [ ] Created four CloudMart buckets with correct locations/classes
- [ ] Uploaded product, catalog, backup, and log examples
- [ ] Used prefixes (`products/`, `config/`, `logs/`)
- [ ] Applied IAM on private buckets; verified backups are not public
- [ ] Enabled public read **only** on static bucket and tested URL
- [ ] Enabled versioning and restored an older catalog generation
- [ ] Applied lifecycle rules on logs bucket
- [ ] Generated and tested a signed URL
- [ ] Deleted buckets and removed public access

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | Buckets list with four `cloudmart` buckets |
| 2 | Object browser showing `products/sku-1001/` prefix |
| 3 | IAM on static bucket with `allUsers` (before cleanup) |
| 4 | Browser showing public static object |
| 5 | `gcloud storage ls -a` showing two versions of `catalog.json` |
| 6 | Lifecycle configuration on logs bucket |
| 7 | Signed URL working in browser |
| 8 | Empty project after cleanup |

---

## Knowledge check

1. What uniquely identifies an object in Cloud Storage?  
   - A) Project ID only  
   - B) Bucket name + object name (and optionally generation)  
   - C) VM zone  

2. Which storage class is best for **daily backups** accessed a few times per month?  
   - A) Archive  
   - B) Nearline  
   - C) Standard only, always  

3. What does **uniform bucket-level access** encourage you to use?  
   - A) Object ACLs only  
   - B) IAM policies at bucket/project level  
   - C) Public `default` network  

4. How do you simulate folders in a bucket?  
   - A) Create sub-buckets for each folder  
   - B) Use `/` in object names (prefixes)  
   - C) Attach Persistent Disks  

5. When should you use a **signed URL**?  
   - A) Permanent anonymous access to all company data  
   - B) Time-limited read/write access without making a bucket public  
   - C) VPC firewall configuration  

<details>
<summary><strong>Answer key</strong></summary>

1. **B** — Bucket + object name (+ generation for versioned objects).  
2. **B** — Nearline fits infrequent access with lower storage cost than Standard.  
3. **B** — Uniform access uses IAM.  
4. **B** — Prefixes in object names.  
5. **B** — Signed URLs grant temporary access.

</details>

---

## Optional challenge

1. Enable **soft delete** on the assets bucket and restore a deleted object.  
2. Configure **CORS** on the static bucket for a mock origin `https://shop.cloudmart.example`.  
3. Mount a bucket with **Cloud Storage FUSE** on a GCE VM and edit a file in place (discuss limitations).  
4. Create a **dual-region** bucket for disaster recovery and compare pricing with regional `asia-south1`.

---

## Official references

- [Cloud Storage overview](https://cloud.google.com/storage/docs/introduction)
- [Storage classes](https://cloud.google.com/storage/docs/storage-classes)
- [Create buckets](https://cloud.google.com/storage/docs/creating-buckets)
- [Upload and download objects](https://cloud.google.com/storage/docs/uploading-objects)
- [Uniform bucket-level access](https://cloud.google.com/storage/docs/uniform-bucket-level-access)
- [IAM for Cloud Storage](https://cloud.google.com/storage/docs/access-control/iam)
- [Object versioning](https://cloud.google.com/storage/docs/object-versioning)
- [Lifecycle management](https://cloud.google.com/storage/docs/lifecycle)
- [Signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)
- [`gcloud storage` reference](https://cloud.google.com/sdk/gcloud/reference/storage)

---

## Lab outcome

You implemented a realistic **CloudMart object storage** layout: private product masters, public storefront assets, Nearline backups, and lifecycle-managed logs—using IAM, versioning, and signed URLs.

You can take other training labs in any order; none are required before or after this one.