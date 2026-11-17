# Lab 05 — Cloud SQL: Managed MySQL for CloudMart

| | |
| --- | --- |
| **Training module** | Module 5 — Cloud SQL |
| **Level** | Beginner |
| **Estimated time** | 60–90 minutes |
| **Primary service** | Cloud SQL (MySQL) |
| **Lab prefix** | `cloudmart` |
| **Region** | `asia-south1` (Mumbai) |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [What you will build](#what-you-will-build)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Cloud SQL fundamentals](#part-1--cloud-sql-fundamentals)
8. [Part 2 — Create a MySQL instance](#part-2--create-a-mysql-instance)
9. [Part 3 — Create database and application user](#part-3--create-database-and-application-user)
10. [Part 4 — Load CloudMart sample data](#part-4--load-cloudmart-sample-data)
11. [Part 5 — Query and verify](#part-5--query-and-verify)
12. [Part 6 — Backups and operations (overview)](#part-6--backups-and-operations-overview)
13. [Part 7 — Disable and delete the instance](#part-7--disable-and-delete-the-instance)
14. [Validation checklist](#validation-checklist)
15. [Evidence to submit](#evidence-to-submit)
16. [Knowledge check](#knowledge-check)
17. [Optional challenge](#optional-challenge)
18. [Official references](#official-references)

---

## Lab description

In this lab, you create a small **Cloud SQL for MySQL** instance for **CloudMart**, add a **database** and **application user**, load a **simple product and order schema**, and run basic queries. You skip advanced topics (high availability, read replicas, private Service Connect) so the focus stays on **managed database fundamentals** and **cleanup to stop billing**.

---

## Scenario and use case

CloudMart needs a **managed relational database** for:

- **Products** — SKU, name, price  
- **Orders** — which product was sold and how many units  

The development team should not patch MySQL or manage disks on a VM. **Cloud SQL** provides a managed MySQL instance with automated maintenance and backups.

**Your role:** Provision a **dev-sized** instance, load sample data, confirm queries work, then **delete** the instance when the lab ends.

**Standalone lab:** You do not need any other lab in this folder. The instance, database, and users exist only for this exercise.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain when to use Cloud SQL vs self-managed databases on Compute Engine
- Create a single-zone MySQL instance in a region
- Create a database and a least-privilege database user
- Connect from **Cloud Shell** with `gcloud sql connect`
- Run simple DDL/DML for a retail schema
- Locate **backups** in the Console (conceptual only)
- **Delete** the instance so charges stop

---

## What you will build

| Item | Value |
| --- | --- |
| Instance name | `cloudmart-mysql` |
| Engine | MySQL 8.0 |
| Size | Small dev tier (e.g. `db-g1-small` or smallest available in Console) |
| Storage | 10 GB |
| Database | `cloudmart_db` |
| App user | `cloudmart_app` |
| Tables | `products`, `orders` |

**Cost note:** Cloud SQL runs 24/7 until deleted. **Part 7 is required** when you finish.

---

## Prerequisites

- [ ] Training project with **billing enabled**
- [ ] Permission to create/delete Cloud SQL instances (e.g. **Cloud SQL Admin** or **Editor** on a sandbox project)
- [ ] **Cloud Shell** available (simplest way to connect in this lab)
- [ ] A strong lab password you choose (not shared publicly)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export INSTANCE="cloudmart-mysql"
export REGION="asia-south1"

gcloud config set project "$PROJECT_ID"

gcloud services enable sqladmin.googleapis.com
```

---

## How to work through this lab

This is a **straight path**: create instance → create DB/user → load data → query → delete instance.

---

## Part 1 — Cloud SQL fundamentals

**Time estimate:** ~10 minutes

### Step 1 — Managed database basics

| Self-managed DB on a VM | Cloud SQL |
| --- | --- |
| You install MySQL, patches, backups | Google manages the instance software and infrastructure |
| You size the VM and disk | You choose **tier** and **storage** |
| You configure replication for HA | **High availability** is optional (not used in this lab) |

### Step 2 — Open Cloud SQL in the Console

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → **SQL**.
2. Review the empty instance list (or existing team instances).
3. Open **Create an instance** but do not submit yet — you will configure it in Part 2.

---

## Part 2 — Create a MySQL instance

**Time estimate:** ~15 minutes (plus 5–10 minutes for provisioning)

### Step 3 — Choose a strong root password

In Cloud Shell:

```bash
# Choose your own password; example length check only:
export ROOT_PASSWORD='ChangeMe-StrongLabPassword99!'
```

Do not commit this password to git or screenshots.

---

### Step 4 — Create the instance (Console — recommended)

1. **SQL → Create instance → Choose MySQL**.
2. **Instance ID:** `cloudmart-mysql`.
3. **Password:** set the **root** password (same as `ROOT_PASSWORD` if you use CLI later).
4. **Region:** `asia-south1`.
5. **Zonal availability:** **Single zone** (not HA).
6. **Machine configuration:** pick the **smallest development** tier shown (e.g. **Shared core — 1 vCPU, 1.7 GB** or `db-g1-small`).
7. **Storage:** 10 GB; **HDD** is fine for this lab (lower cost).
8. **Connections:** leave defaults for this simple lab (no private IP setup required).
9. **Backup:** leave **automated backups enabled** (default) — you only view them in Part 6.
10. **Flags / replicas / advanced:** leave **defaults** — do not enable read replicas.
11. Click **Create instance** and wait until status is **Runnable**.

---

### Step 5 — Create the instance (CLI alternative)

If Console SKUs differ, use the Console. Otherwise:

```bash
gcloud sql instances create "$INSTANCE" \
  --database-version=MYSQL_8_0 \
  --tier=db-g1-small \
  --region="$REGION" \
  --storage-size=10GB \
  --storage-type=HDD \
  --root-password="$ROOT_PASSWORD"
```

Describe the instance:

```bash
gcloud sql instances describe "$INSTANCE" \
  --format="yaml(name,state,region,databaseVersion,settings.tier)"
```

**Expected result:** `state: RUNNABLE`.

---

## Part 3 — Create database and application user

**Time estimate:** ~10 minutes

### Step 6 — Create the database (Console)

1. Open instance **`cloudmart-mysql`**.
2. Go to **Databases → Create database**.
3. **Name:** `cloudmart_db`.
4. Create.

### Step 7 — Create the database (CLI)

```bash
gcloud sql databases create cloudmart_db --instance="$INSTANCE"
```

---

### Step 8 — Create application user

Do not use **root** for the application.

```bash
export APP_PASSWORD='ChangeMe-AppUserPassword99!'

gcloud sql users create cloudmart_app \
  --instance="$INSTANCE" \
  --password="$APP_PASSWORD"
```

Console: **Users → Add user →** name `cloudmart_app`, password as above.

---

## Part 4 — Load CloudMart sample data

**Time estimate:** ~20 minutes

### Step 9 — Prepare SQL file in Cloud Shell

```bash
cat > ~/cloudmart_schema.sql <<'EOF'
USE cloudmart_db;

CREATE TABLE IF NOT EXISTS products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(20) NOT NULL UNIQUE,
  name VARCHAR(100) NOT NULL,
  price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE IF NOT EXISTS orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  product_sku VARCHAR(20) NOT NULL,
  quantity INT NOT NULL,
  order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (sku, name, price) VALUES
  ('1001', 'Wireless Earbuds', 49.99),
  ('1002', 'USB-C Hub', 29.99),
  ('1003', 'Laptop Stand', 39.99);

INSERT INTO orders (product_sku, quantity) VALUES
  ('1001', 2),
  ('1002', 1),
  ('1001', 1);
EOF
```

---

### Step 10 — Connect as root and run the script

```bash
gcloud sql connect "$INSTANCE" --user=root --quiet < ~/cloudmart_schema.sql
```

If prompted for a password, enter `ROOT_PASSWORD`.

**Troubleshooting:** If connect fails, wait a few minutes after instance creation, ensure you run from **Cloud Shell**, and retry.

---

### Step 11 — Grant privileges to the app user

```bash
cat > ~/cloudmart_grants.sql <<'EOF'
USE cloudmart_db;
GRANT SELECT, INSERT, UPDATE, DELETE ON cloudmart_db.* TO 'cloudmart_app'@'%';
FLUSH PRIVILEGES;
EOF

gcloud sql connect "$INSTANCE" --user=root --quiet < ~/cloudmart_grants.sql
```

---

## Part 5 — Query and verify

**Time estimate:** ~10 minutes

### Step 12 — Connect as application user

```bash
gcloud sql connect "$INSTANCE" --user=cloudmart_app --quiet
```

At the `mysql>` prompt, run:

```sql
USE cloudmart_db;

SELECT * FROM products;

SELECT o.id, p.name, o.quantity, o.order_date
FROM orders o
JOIN products p ON p.sku = o.product_sku;

INSERT INTO orders (product_sku, quantity) VALUES ('1003', 2);

SELECT COUNT(*) AS total_orders FROM orders;
```

Type `exit` to leave the client.

**Expected result:** Three products, order rows with product names, and `total_orders` = 4 after the insert.

---

### Step 13 — One-line query from Cloud Shell (optional)

```bash
gcloud sql connect "$INSTANCE" --user=cloudmart_app --quiet \
  --database=cloudmart_db \
  --command="SELECT sku, name, price FROM products ORDER BY sku;"
```

If your `gcloud` version does not support `--command`, use the interactive session from Step 12.

---

## Part 6 — Backups and operations (overview)

**Time estimate:** ~5 minutes (no restore exercise)

### Step 14 — View automated backups

1. In the Console, open **`cloudmart-mysql`** → **Backups**.
2. Note that **automated backups** are enabled by default on new instances.

You will **not** perform point-in-time restore in this lab (kept simple on purpose).

### Step 15 — Managed backups vs export to object storage (concept)

**Automated backups** on Cloud SQL support instance-level recovery on Google’s managed service. Teams may also export logical dumps (`.sql`) to **Cloud Storage** for long-term archive— that is a separate optional pattern and not required for this lab.

---

## Part 7 — Disable and delete the instance

**Time estimate:** ~10 minutes

Cloud SQL does not have a “stop VM” button. To **stop billing**, you **delete** the instance when the lab is done.

### Step 16 — Confirm you exported evidence

Ensure screenshots and the knowledge check are done **before** deletion.

---

### Step 17 — Delete the instance (Console)

1. **SQL → Instances**.
2. Select **`cloudmart-mysql`**.
3. **Delete**.
4. Type the instance name to confirm.
5. Wait until the instance disappears from the list.

---

### Step 18 — Delete the instance (CLI)

```bash
gcloud sql instances delete "$INSTANCE" --quiet
```

Verify:

```bash
gcloud sql instances list --filter="name:cloudmart"
```

**Expected result:** No `cloudmart-mysql` instance.

---

### Step 19 — Clear local password variables (optional)

```bash
unset ROOT_PASSWORD APP_PASSWORD
```

Remove `~/cloudmart_schema.sql` and `~/cloudmart_grants.sql` from Cloud Shell if they contain no other secrets.

---

## Validation checklist

- [ ] Enabled Cloud SQL Admin API
- [ ] Created single-zone instance `cloudmart-mysql`
- [ ] Created database `cloudmart_db` and user `cloudmart_app`
- [ ] Loaded `products` and `orders` sample data
- [ ] Ran JOIN query successfully
- [ ] Viewed **Backups** tab in Console
- [ ] **Deleted** the instance (billing stopped)

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | SQL instances list showing `cloudmart-mysql` **Runnable** |
| 2 | Databases tab with `cloudmart_db` |
| 3 | Output of `SELECT * FROM products` |
| 4 | Output of JOIN query on orders and products |
| 5 | Backups page showing automated backup enabled |
| 6 | Empty instance list after delete (cleanup) |

---

## Knowledge check

1. What is the main benefit of Cloud SQL compared to MySQL on a VM?  
   - A) Google manages database operations and infrastructure  
   - B) No SQL required  
   - C) It replaces Cloud Storage  

2. Which user should a CloudMart application use?  
   - A) `root` always  
   - B) A dedicated user such as `cloudmart_app` with limited grants  
   - C) `allUsers`  

3. How do you stop Cloud SQL charges after this lab?  
   - A) Delete the instance  
   - B) Close the browser tab only  
   - C) Disable Compute Engine API  

4. What does this lab store in the `orders` table?  
   - A) VPC firewall rules  
   - B) Product SKU, quantity, and order date  
   - C) GCS bucket lifecycle policies  

5. Which tool connects from Cloud Shell in this lab?  
   - A) `gcloud sql connect`  
   - B) `kubectl exec`  
   - C) `bq load`  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — Managed service.  
2. **B** — Least-privilege app user.  
3. **A** — Delete the instance.  
4. **B** — Simple order lines.  
5. **A** — `gcloud sql connect`.

</details>

---

## Optional challenge

Only if you finished cleanup and have instructor approval to create another instance:

1. Add a column `stock` to `products` and update rows.  
2. Export query results with `SELECT ... INTO OUTFILE` is restricted on Cloud SQL — discuss why, then export via client-side redirect in Cloud Shell instead.  
3. Export `mysqldump` output to a Cloud Storage bucket you create for logical backups.

---

## Official references

- [Cloud SQL overview](https://cloud.google.com/sql/docs/introduction)
- [Cloud SQL for MySQL](https://cloud.google.com/sql/docs/mysql)
- [Create instances](https://cloud.google.com/sql/docs/mysql/create-instance)
- [Connect using Cloud Shell](https://cloud.google.com/sql/docs/mysql/connect-admin-ip)
- [Create and manage databases](https://cloud.google.com/sql/docs/mysql/create-manage-databases)
- [Backups](https://cloud.google.com/sql/docs/mysql/backup-recovery/backups)

---

## Lab outcome

You created a small **Cloud SQL for MySQL** instance for CloudMart, loaded product and order data, queried it with an application user, reviewed backups at a high level, and **deleted** the instance to avoid ongoing cost.

You can take other training labs in any order; none are required before or after this one.