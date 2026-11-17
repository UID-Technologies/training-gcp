# Lab 07 — BigQuery: Analytics for CloudMart

| | |
| --- | --- |
| **Training module** | Module 7 — BigQuery |
| **Level** | Beginner |
| **Estimated time** | 60–90 minutes |
| **Primary service** | BigQuery |
| **Lab prefix** | `cloudmart` |
| **Dataset location** | `asia-south1` |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [What you will build](#what-you-will-build)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — BigQuery fundamentals](#part-1--bigquery-fundamentals)
8. [Part 2 — Enable APIs and create a dataset](#part-2--enable-apis-and-create-a-dataset)
9. [Part 3 — Load CloudMart sample data](#part-3--load-cloudmart-sample-data)
10. [Part 4 — Run SQL analytics queries](#part-4--run-sql-analytics-queries)
11. [Part 5 — Console exploration and query cost](#part-5--console-exploration-and-query-cost)
12. [Part 6 — Export results (optional)](#part-6--export-results-optional)
13. [Part 7 — Clean up lab resources](#part-7--clean-up-lab-resources)
14. [Validation checklist](#validation-checklist)
15. [Evidence to submit](#evidence-to-submit)
16. [Knowledge check](#knowledge-check)
17. [Optional challenge](#optional-challenge)
18. [Official references](#official-references)

---

## Lab description

In this lab, you use **BigQuery** to analyze **CloudMart** retail data. You create a **dataset**, load **products** and **orders** from CSV files, and run **SQL** queries for sales summaries and product performance. You learn where to run queries, how data is organized, and how to **delete** the dataset when finished.

---

## Scenario and use case

CloudMart leadership asks: *Which products drive the most revenue? How many orders did we process last week?*

| Question | Tool |
| --- | --- |
| Aggregations over millions of rows | **BigQuery** (serverless SQL) |
| Real-time clicks at extreme scale | Cloud Bigtable (separate module) |
| Transactional checkout | Cloud SQL or application database |

**Your role:** Stand up a small **analytics dataset**, load sample sales data, and answer business questions with SQL.

**Standalone lab:** You do not need any other lab in this folder. All datasets and tables are created and removed here.

---

## Learning objectives

When you finish this lab, you should be able to:

- Describe projects, datasets, tables, and jobs in BigQuery
- Create a dataset with a chosen **location**
- Load CSV data into **native** BigQuery tables
- Run `SELECT`, `JOIN`, `GROUP BY`, and aggregates in the Console and `bq` CLI
- Estimate **query size** and follow cost-aware habits
- Delete a dataset and its tables

---

## What you will build

| Item | Value |
| --- | --- |
| Dataset | `cloudmart_analytics` |
| Table | `products` |
| Table | `orders` |
| Location | `asia-south1` |
| Sample rows | 5 products, 8+ order lines |

**Cost note:** Storage and queries are small in this lab. BigQuery has a **free operations tier** per month; still delete the dataset in Part 7.

---

## Prerequisites

- [ ] Training project with **billing enabled**
- [ ] **BigQuery User** or **BigQuery Admin** on the project
- [ ] Cloud Shell or browser access to **BigQuery Studio**
- [ ] Basic SQL (`SELECT`, `WHERE`, `JOIN`)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export DATASET="cloudmart_analytics"
export LOCATION="asia-south1"

gcloud config set project "$PROJECT_ID"
```

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1–2 | Concepts, dataset |
| 3 | Load CSV |
| 4–5 | SQL and Console |
| 6 | Optional export |
| 7 | Delete dataset |

---

## Part 1 — BigQuery fundamentals

**Time estimate:** ~10 minutes

### Step 1 — What BigQuery is

**BigQuery** is Google’s **serverless data warehouse**. You store data in **tables** and analyze it with **SQL** without managing database servers.

| Term | Meaning |
| --- | --- |
| **Project** | Billing and IAM boundary |
| **Dataset** | Group of tables (like a schema) |
| **Table** | Rows and columns (or nested fields) |
| **Job** | Load, query, or export operation |

---

### Step 2 — Open BigQuery Studio

1. Console → **BigQuery** (or **BigQuery Studio**).
2. In the explorer, locate your **project ID**.
3. Open **Public datasets** — many demos use them; this lab uses **your own** dataset only.

---

## Part 2 — Enable APIs and create a dataset

**Time estimate:** ~10 minutes

### Step 3 — Enable the BigQuery API

```bash
gcloud services enable bigquery.googleapis.com
```

---

### Step 4 — Create the dataset

**Console:** **BigQuery → Create dataset**

| Field | Value |
| --- | --- |
| Dataset ID | `cloudmart_analytics` |
| Location | `asia-south1` |
| Default table expiration | None (for lab) |

**CLI:**

```bash
bq --location="$LOCATION" mk --dataset "${PROJECT_ID}:${DATASET}"
```

Verify:

```bash
bq ls
```

---

## Part 3 — Load CloudMart sample data

**Time estimate:** ~20 minutes

### Step 5 — Create CSV files in Cloud Shell

```bash
mkdir -p ~/cloudmart-bq
cd ~/cloudmart-bq

cat > products.csv <<'EOF'
sku,name,category,unit_price
1001,Wireless Earbuds,Electronics,49.99
1002,USB-C Hub,Electronics,29.99
1003,Laptop Stand,Accessories,39.99
1004,Mechanical Keyboard,Electronics,89.99
1005,Webcam HD,Electronics,59.99
EOF

cat > orders.csv <<'EOF'
order_id,order_date,sku,quantity
9001,2026-11-10,1001,2
9002,2026-11-11,1002,1
9003,2026-11-11,1001,1
9004,2026-11-12,1003,3
9005,2026-11-12,1004,1
9006,2026-11-13,1005,2
9007,2026-11-14,1001,4
9008,2026-11-15,1002,2
EOF
```

---

### Step 6 — Load into BigQuery tables

```bash
bq load --source_format=CSV --skip_leading_rows=1 --autodetect \
  "${DATASET}.products" products.csv

bq load --source_format=CSV --skip_leading_rows=1 --autodetect \
  "${DATASET}.orders" orders.csv
```

**Console:** **Create table → Upload** → select CSV → name `products`, repeat for `orders`.

Preview:

```bash
bq show "${DATASET}.products"
bq head -n 5 "${DATASET}.products"
bq head -n 5 "${DATASET}.orders"
```

---

## Part 4 — Run SQL analytics queries

**Time estimate:** ~25 minutes

### Step 7 — Revenue by product name

In **BigQuery Studio → SQL query**, or Cloud Shell:

```sql
SELECT
  p.sku,
  p.name,
  SUM(o.quantity) AS units_sold,
  ROUND(SUM(o.quantity * p.unit_price), 2) AS revenue
FROM `YOUR_PROJECT_ID.cloudmart_analytics.orders` AS o
JOIN `YOUR_PROJECT_ID.cloudmart_analytics.products` AS p
  ON o.sku = p.sku
GROUP BY p.sku, p.name
ORDER BY revenue DESC;
```

Replace `YOUR_PROJECT_ID` with your project ID.

**CLI:**

```bash
bq query --use_legacy_sql=false '
SELECT
  p.name,
  SUM(o.quantity * p.unit_price) AS revenue
FROM `'"${PROJECT_ID}"'.cloudmart_analytics.orders` o
JOIN `'"${PROJECT_ID}"'.cloudmart_analytics.products` p
  ON o.sku = p.sku
GROUP BY p.name
ORDER BY revenue DESC'
```

**Expected result:** Earbuds and other products ranked by revenue.

---

### Step 8 — Orders and units per day

```sql
SELECT
  order_date,
  COUNT(DISTINCT order_id) AS order_count,
  SUM(quantity) AS units_sold
FROM `YOUR_PROJECT_ID.cloudmart_analytics.orders`
GROUP BY order_date
ORDER BY order_date;
```

---

### Step 9 — Filter and limit (category breakdown)

```sql
SELECT
  p.category,
  COUNT(*) AS line_items,
  SUM(o.quantity) AS units
FROM `YOUR_PROJECT_ID.cloudmart_analytics.orders` o
JOIN `YOUR_PROJECT_ID.cloudmart_analytics.products` p
  ON o.sku = p.sku
WHERE p.category = 'Electronics'
GROUP BY p.category;
```

---

### Step 10 — Save a view (optional)

```bash
bq mk --view \
"SELECT sku, SUM(quantity) AS units FROM \`${PROJECT_ID}.${DATASET}.orders\` GROUP BY sku" \
  "${DATASET}.v_units_by_sku"
```

Query the view:

```sql
SELECT * FROM `YOUR_PROJECT_ID.cloudmart_analytics.v_units_by_sku` ORDER BY units DESC;
```

---

## Part 5 — Console exploration and query cost

**Time estimate:** ~10 minutes

### Step 11 — Inspect table details

In the Console, open **products** and **orders** → **Details**, **Preview**, **Schema**.

Note **Number of rows** and **Size**.

---

### Step 12 — Check bytes processed

Before running a query in the Console, look at **This query will process X MB**.

**Habits:**

- Select only columns you need (`SELECT *` costs more on large tables).
- Use **partitioned** and **clustered** tables in production (not required here).
- Prefer **preview** (`LIMIT 10`) when exploring unknown data.

---

## Part 6 — Export results (optional)

**Time estimate:** ~5 minutes

### Step 13 — Save query results to a table

```bash
bq query --use_legacy_sql=false --destination_table="${DATASET}.revenue_report" \
  --replace=true '
SELECT p.name, SUM(o.quantity * p.unit_price) AS revenue
FROM `'"${PROJECT_ID}"'.${DATASET}.orders` o
JOIN `'"${PROJECT_ID}"'.${DATASET}.products` p ON o.sku = p.sku
GROUP BY p.name'
```

List tables:

```bash
bq ls "${DATASET}"
```

---

## Part 7 — Clean up lab resources

**Time estimate:** ~5 minutes

### Step 14 — Delete the dataset (includes tables and views)

```bash
bq rm -r -f -d "${DATASET}"
```

Console: **dataset → Delete dataset** → type the dataset name.

Verify:

```bash
bq ls
```

Remove local files:

```bash
rm -rf ~/cloudmart-bq
```

---

## Validation checklist

- [ ] Created dataset `cloudmart_analytics` in `asia-south1`
- [ ] Loaded `products` and `orders` from CSV
- [ ] Ran JOIN query for revenue by product
- [ ] Ran aggregation by `order_date`
- [ ] Checked query bytes processed in Console
- [ ] Deleted dataset in Part 7

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | Dataset and tables in explorer |
| 2 | Schema of `orders` |
| 3 | Revenue-by-product query results |
| 4 | Query validator showing bytes processed |
| 5 | Empty dataset list after cleanup |

---

## Knowledge check

1. What is the correct hierarchy in BigQuery?  
   - A) Table → Dataset → Project  
   - B) Project → Dataset → Table  
   - C) Dataset → Project → Table  

2. Which language do you use for analytics in BigQuery?  
   - A) SQL  
   - B) HBase shell  
   - C) Nginx config  

3. Where should the dataset live for data processed in Mumbai?  
   - A) A region such as `asia-south1`  
   - B) Only `US` multi-region always  
   - C) On a single VM disk  

4. What does a load job do?  
   - A) Starts a Compute Engine VM  
   - B) Imports data from a file into a table  
   - C) Creates a VPC subnet  

5. How do you remove all lab tables at once?  
   - A) Delete the dataset  
   - B) Disable BigQuery API only  
   - C) Delete the project  

<details>
<summary><strong>Answer key</strong></summary>

1. **B** — Project → Dataset → Table.  
2. **A** — SQL.  
3. **A** — Use an appropriate region.  
4. **B** — Load job imports data.  
5. **A** — Delete the dataset (with tables).

</details>

---

## Optional challenge

1. Query a **public dataset** (for example `bigquery-public-data`) with a `LIMIT 10` query and compare bytes processed to your CloudMart tables.  
2. Load `orders.csv` again with an explicit schema instead of `--autodetect`.  
3. Create a **parameterized** query in the Console for `category = 'Electronics'`.

---

## Official references

- [BigQuery overview](https://cloud.google.com/bigquery/docs/introduction)
- [Create datasets](https://cloud.google.com/bigquery/docs/datasets)
- [Load CSV data](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv)
- [Run queries](https://cloud.google.com/bigquery/docs/running-queries)
- [Query pricing](https://cloud.google.com/bigquery/pricing)
- [`bq` command reference](https://cloud.google.com/bigquery/docs/reference/bq-cli)

---

## Lab outcome

You created a **CloudMart analytics dataset**, loaded products and orders, answered business questions with **SQL**, and deleted the dataset when done.

You can take other training labs in any order; none are required before or after this one.