# Lab 08 — Cloud Bigtable: Real-Time Events for CloudMart

| | |
| --- | --- |
| **Training module** | Module 8 — Cloud Bigtable |
| **Level** | Beginner to intermediate |
| **Estimated time** | 60–90 minutes |
| **Primary service** | Cloud Bigtable |
| **Lab prefix** | `cloudmart` |
| **Instance zone** | `asia-south1-b` |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [What you will build](#what-you-will-build)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Cloud Bigtable fundamentals](#part-1--cloud-bigtable-fundamentals)
8. [Part 2 — Enable APIs and install the cbt tool](#part-2--enable-apis-and-install-the-cbt-tool)
9. [Part 3 — Create a development instance](#part-3--create-a-development-instance)
10. [Part 4 — Design row keys for CloudMart events](#part-4--design-row-keys-for-cloudmart-events)
11. [Part 5 — Create table and column family](#part-5--create-table-and-column-family)
12. [Part 6 — Write and read event data](#part-6--write-and-read-event-data)
13. [Part 7 — Scan and count rows](#part-7--scan-and-count-rows)
14. [Part 8 — Clean up (delete instance)](#part-8--clean-up-delete-instance)
15. [Validation checklist](#validation-checklist)
16. [Evidence to submit](#evidence-to-submit)
17. [Knowledge check](#knowledge-check)
18. [Optional challenge](#optional-challenge)
19. [Official references](#official-references)

---

## Lab description

In this hands-on lab, you use **Cloud Bigtable** to store **high-volume shopping events** for **CloudMart** (product views and add-to-cart clicks). You create a **development instance**, design **row keys**, write and read data with the **`cbt`** CLI, and **delete the instance** to stop charges.

---

## Scenario and use case

CloudMart’s website sends millions of **small events**:

| Event | Example row key idea | Why not only BigQuery |
| --- | --- | --- |
| Product view | `sku#1001#reverse_timestamp` | Very high **write** rate, millisecond reads |
| Add to cart | `user#alice#event_id` | Low-latency lookup by key |

**BigQuery** is ideal for **SQL reports** over batch or streamed data. **Bigtable** is ideal when you need **huge throughput** and **key-based** access with predictable row keys.

**Your role:** Provision a **lab instance**, model a simple **events** table, insert sample rows, read them back, then **delete** the instance.

**Standalone lab:** You do not need any other lab in this folder.

---

## Learning objectives

When you finish this lab, you should be able to:

- Contrast **Bigtable** vs **BigQuery** vs relational databases
- Create a **Cloud Bigtable instance** (development type for labs)
- Explain **row keys**, **column families**, and **cells**
- Use **`cbt`** to create tables, write, and read rows
- Perform a simple **prefix scan** on row keys
- **Delete** the instance to stop billing

---

## What you will build

| Item | Value |
| --- | --- |
| Instance | `cloudmart-bt` |
| Cluster | `cloudmart-bt-c1` |
| Zone | `asia-south1-b` |
| Table | `product_events` |
| Column family | `cf` (events attributes) |

**Cost note:** Even a **development** instance can incur charges. Complete the lab in one session and run **Part 8** immediately after.

---

## Prerequisites

- [ ] Training project with **billing enabled**
- [ ] **Bigtable Admin** or equivalent permissions
- [ ] Cloud Shell (includes `gcloud`; you will install **`cbt`**)
- [ ] Basic comfort with key–value style data (not SQL for main exercises)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export INSTANCE="cloudmart-bt"
export CLUSTER="cloudmart-bt-c1"
export ZONE="asia-south1-b"
export TABLE="product_events"

gcloud config set project "$PROJECT_ID"
```

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1 | Concepts vs BigQuery |
| 2–3 | Instance |
| 4–5 | Schema design |
| 6–7 | Read/write |
| 8 | **Required cleanup** |

---

## Part 1 — Cloud Bigtable fundamentals

**Time estimate:** ~10 minutes

### Step 1 — When to use Bigtable

| Use BigQuery | Use Bigtable |
| --- | --- |
| SQL analytics, BI, ad hoc aggregates | Massive **reads/writes** by row key |
| Petabyte scans with columnar engine | Single-digit ms **key lookup** patterns |
| Batch and micro-batch pipelines | Open-source **HBase**-style apps |

Bigtable is **not** a replacement for Cloud SQL or BigQuery; it fills a different layer in the **data platform**.

---

### Step 2 — Core terms

| Term | Meaning |
| --- | --- |
| **Instance** | Bigtable service endpoint in your project |
| **Cluster** | Nodes in a zone that serve data |
| **Table** | Logical table (rows) |
| **Row key** | Unique key; **design** drives performance |
| **Column family** | Group of columns, configured once |
| **Cell** | Value at row + column family + qualifier + timestamp |

---

## Part 2 — Enable APIs and install the cbt tool

**Time estimate:** ~10 minutes

### Step 3 — Enable the Bigtable API

```bash
gcloud services enable bigtableadmin.googleapis.com bigtable.googleapis.com
```

---

### Step 4 — Install `cbt` (Cloud Shell)

```bash
if ! command -v cbt &> /dev/null; then
  go install github.com/googlecloudplatform/google-cloud-go/bigtable/cmd/cbt@latest
  export PATH="$PATH:$(go env GOPATH)/bin"
fi
cbt version
```

If `go install` is not available, use the **package** or **binary** from [Install cbt](https://cloud.google.com/bigtable/docs/cbt-overview) per instructor notes.

Set defaults:

```bash
export CBT_PROJECT="$PROJECT_ID"
export CBT_INSTANCE="$INSTANCE"
```

Or prefix every command with `cbt -project $PROJECT_ID -instance $INSTANCE`.

---

## Part 3 — Create a development instance

**Time estimate:** ~15 minutes

### Step 5 — Create instance (Console)

1. Console → **Bigtable → Instances → Create instance**.
2. **Instance name:** `cloudmart-bt`.
3. **Instance type:** **Development** (if offered — single node, for learning).
4. **Cluster name:** `cloudmart-bt-c1`.
5. **Zone:** `asia-south1-b`.
6. **Storage type:** SSD.
7. Create and wait until **Ready**.

If **Development** is not available in your project, create a **Production** instance with **1 node** and delete it promptly in Part 8.

---

### Step 6 — Create instance (CLI)

```bash
gcloud bigtable instances create "$INSTANCE" \
  --cluster="$CLUSTER" \
  --cluster-zone="$ZONE" \
  --display-name="CloudMart Bigtable Lab" \
  --instance-type=development \
  --storage-type=SSD
```

If your `gcloud` version uses different flags, follow the Console path in Step 5.

List instances:

```bash
gcloud bigtable instances list
```

**Expected result:** Instance state **READY**.

---

## Part 4 — Design row keys for CloudMart events

**Time estimate:** ~10 minutes

### Step 7 — Row key pattern

For **product view** events, use:

```text
{sku}#{reverse_timestamp}#{event_id}
```

Example:

```text
1001#79240751999#evt-001
```

`reverse_timestamp` is `9223372036854775807 - unix_millis` so newer events sort **first** for a SKU prefix scan.

For this lab you may use **simple keys** such as `1001#evt-001` and scan prefix `1001#`.

**Avoid:** monotonically increasing row keys only (`evt-0001`, `evt-0002`, …) on huge tables — they hotspot one node. CloudMart lab has few rows; pattern is for learning.

---

## Part 5 — Create table and column family

**Time estimate:** ~10 minutes

### Step 8 — Create table `product_events`

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" createtable "$TABLE"
```

---

### Step 9 — Create column family `cf`

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" createfamily "$TABLE" cf
```

List tables:

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" ls
```

Console: **Tables** under instance → verify `product_events` and family `cf`.

---

## Part 6 — Write and read event data

**Time estimate:** ~15 minutes

### Step 10 — Insert sample events

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" set \
  "$TABLE" "1001#evt-001" \
  cf:event_type=view cf:sku=1001 cf:user_id=alice

cbt -project="$PROJECT_ID" -instance="$INSTANCE" set \
  "$TABLE" "1001#evt-002" \
  cf:event_type=add_to_cart cf:sku=1001 cf:user_id=alice

cbt -project="$PROJECT_ID" -instance="$INSTANCE" set \
  "$TABLE" "1002#evt-003" \
  cf:event_type=view cf:sku=1002 cf:user_id=bob
```

---

### Step 11 — Read a single row

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" lookup "$TABLE" "1001#evt-001"
```

**Expected result:** Column qualifiers `event_type`, `sku`, `user_id` with values.

---

### Step 12 — Read specific cells

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" read \
  "$TABLE" "1001#evt-002" columns=cf:event_type
```

---

## Part 7 — Scan and count rows

**Time estimate:** ~10 minutes

### Step 13 — Prefix scan for SKU 1001

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" read \
  "$TABLE" prefix=1001#
```

**Expected result:** Rows `1001#evt-001` and `1001#evt-002`.

---

### Step 14 — Count rows (approximate for lab)

```bash
cbt -project="$PROJECT_ID" -instance="$INSTANCE" count "$TABLE"
```

**Expected result:** `3` (if only lab rows exist).

Discuss: for billions of rows, apps use **batch scanners** and **filters**, not full table scans in production.

---

## Part 8 — Clean up (delete instance)

**Time estimate:** ~10 minutes

Deleting the **instance** removes clusters, tables, and data.

### Step 15 — Delete instance (CLI)

```bash
gcloud bigtable instances delete "$INSTANCE" --quiet
```

---

### Step 16 — Verify (Console)

**Bigtable → Instances** → confirm `cloudmart-bt` is gone.

```bash
gcloud bigtable instances list
```

**Expected result:** No lab instance listed.

---

## Validation checklist

- [ ] Enabled Bigtable APIs
- [ ] Installed and ran `cbt version`
- [ ] Created instance `cloudmart-bt`
- [ ] Created table `product_events` and family `cf`
- [ ] Wrote at least three event rows
- [ ] Performed lookup and prefix read
- [ ] **Deleted** instance in Part 8

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | Instance list showing `cloudmart-bt` Ready |
| 2 | Table schema with column family `cf` |
| 3 | `lookup` output for one row |
| 4 | Prefix scan for `1001#` |
| 5 | Empty instance list after delete |

---

## Knowledge check

1. Bigtable is best described as:  
   - A) Wide-column NoSQL for massive key-based workloads  
   - B) SQL data warehouse  
   - C) Object storage for images  

2. What must you design carefully for performance?  
   - A) Row keys  
   - B) VM machine type only  
   - C) Nginx worker count  

3. What tool does this lab use for administration?  
   - A) `cbt`  
   - B) `kubectl` only  
   - C) `ftp`  

4. How do you stop Bigtable instance charges after the lab?  
   - A) Delete the instance  
   - B) Close the browser  
   - C) Delete only column family `cf`  

5. Compared to BigQuery, Bigtable is typically used for:  
   - A) Ad hoc SQL on huge historical tables  
   - B) Low-latency access by row key at very high throughput  
   - C) Replacing all MySQL applications  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — Wide-column, key-based scale.  
2. **A** — Row key design.  
3. **A** — `cbt` CLI.  
4. **A** — Delete the instance.  
5. **B** — Key-based, high-throughput access.

</details>

---

## Optional challenge

1. Add a second column family `cf_meta` with `source=web` on one row — note only one is needed for the lab.  
2. Document **three** reasons CloudMart would **export** aggregates to BigQuery from Bigtable pipelines (concept).  
3. Read [Row key best practices](https://cloud.google.com/bigtable/docs/row-keys) and write one sentence on **hotspotting**.

---

## Official references

- [Cloud Bigtable overview](https://cloud.google.com/bigtable/docs/overview)
- [Instances and clusters](https://cloud.google.com/bigtable/docs/instances)
- [Row key design](https://cloud.google.com/bigtable/docs/row-keys)
- [cbt CLI reference](https://cloud.google.com/bigtable/docs/cbt-overview)
- [Compare Bigtable and BigQuery](https://cloud.google.com/blog/products/databases/bigquery-vs-bigtable)

---

## Lab outcome

You created a **Cloud Bigtable** instance, stored **CloudMart product events**, practiced **read and scan** operations with **`cbt`**, and **deleted** the instance when finished.

You can take other training labs in any order; none are required before or after this one.