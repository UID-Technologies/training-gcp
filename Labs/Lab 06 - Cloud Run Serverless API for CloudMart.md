# Lab 06 — Cloud Run: Serverless API for CloudMart

| | |
| --- | --- |
| **Training module** | Module 6 — Cloud Run |
| **Level** | Beginner |
| **Estimated time** | 60–90 minutes |
| **Primary service** | Cloud Run |
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
7. [Part 1 — Cloud Run fundamentals](#part-1--cloud-run-fundamentals)
8. [Part 2 — Enable APIs and prepare the project](#part-2--enable-apis-and-prepare-the-project)
9. [Part 3 — Deploy a sample container (quick start)](#part-3--deploy-a-sample-container-quick-start)
10. [Part 4 — Build and deploy the CloudMart catalog API](#part-4--build-and-deploy-the-cloudmart-catalog-api)
11. [Part 5 — Test the service URL](#part-5--test-the-service-url)
12. [Part 6 — Configure the service (environment and scaling)](#part-6--configure-the-service-environment-and-scaling)
13. [Part 7 — Revisions, logs, and monitoring](#part-7--revisions-logs-and-monitoring)
14. [Part 8 — Clean up lab resources](#part-8--clean-up-lab-resources)
15. [Validation checklist](#validation-checklist)
16. [Evidence to submit](#evidence-to-submit)
17. [Knowledge check](#knowledge-check)
18. [Optional challenge](#optional-challenge)
19. [Official references](#official-references)

---

## Lab description

In this lab, you deploy a small **CloudMart product catalog API** on **Cloud Run**. You start with a **sample container** to learn the deploy flow, then use **source-based deployment** to build and ship your own Python service. You test the public URL, adjust **environment variables** and **scaling**, review **logs**, and **delete** the services when finished.

---

## Scenario and use case

CloudMart wants a **lightweight HTTP API** that returns product data to the web store and mobile app. Traffic is **spiky** (sales events) and often **idle** at night.

| Requirement | Cloud Run fit |
| --- | --- |
| Pay only when handling requests | Scales to zero (no idle VM cost) |
| Fast experiments | Deploy from source or container image |
| HTTPS URL | Managed TLS on the service URL |
| Simple catalog endpoints | Small stateless container |

**Your role:** Deploy and operate **`cloudmart-catalog-api`** for development, then remove it to stop charges.

**Standalone lab:** You do not need any other lab in this folder. All services and source files are created and removed in this lab only.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain how Cloud Run runs **stateless containers** on demand
- Enable APIs for Run, Cloud Build, and Artifact Registry
- Deploy a service with `gcloud run deploy`
- Build a container **from source** with Cloud Build
- Invoke a service with its **HTTPS URL**
- Set **environment variables** and **concurrency** limits
- View **revisions** and **logs**
- Delete Cloud Run services and related build artifacts (as needed)

---

## What you will build

| Item | Value |
| --- | --- |
| Quick-start service (optional) | `cloudmart-hello` |
| Main service | `cloudmart-catalog-api` |
| Endpoints | `GET /` (health), `GET /products` (JSON catalog) |
| Runtime | Python 3 + Flask (container) |
| Auth (lab) | Allow unauthenticated access to test the API |

**Cost note:** Cloud Run charges for request time and CPU/memory while handling traffic; builds use Cloud Build minutes. **Part 8 is required** when you finish.

---

## Prerequisites

Before you start, confirm:

- [ ] Training project with **billing enabled**
- [ ] Permission to deploy Cloud Run and run Cloud Build (e.g. **Cloud Run Admin**, **Cloud Build Editor**, **Service Account User**)
- [ ] **Cloud Shell** (recommended) or local `gcloud` with Docker not required for `--source` deploy
- [ ] Basic understanding of HTTP (GET, status codes, JSON)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="asia-south1"
export SERVICE_API="cloudmart-catalog-api"
export SERVICE_HELLO="cloudmart-hello"

gcloud config set project "$PROJECT_ID"
gcloud config set run/region "$REGION"
```

---

## How to work through this lab

| Part | Purpose |
| --- | --- |
| 1–2 | Concepts and APIs |
| 3 | One-line sample deploy (confidence builder) |
| 4–5 | CloudMart API and testing |
| 6–7 | Operations |
| 8 | Cleanup |

You may **skip Part 3** if your instructor wants only the catalog API; delete `cloudmart-hello` in Part 8 if you created it.

---

## Part 1 — Cloud Run fundamentals

**Time estimate:** ~10 minutes

### Step 1 — How Cloud Run differs from a VM

| Compute Engine VM | Cloud Run |
| --- | --- |
| You manage the VM lifecycle | Google manages the infrastructure |
| Often runs 24/7 | **Scales to zero** when idle |
| You patch the OS | You ship a **container image** |
| You configure load balancing separately | **HTTPS URL** included |

Cloud Run is suited to **stateless** web APIs, webhooks, and microservices.

---

### Step 2 — Open Cloud Run in the Console

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → **Cloud Run**.
2. Review the **Services** list (may be empty).
3. Note the **Region** filter — this lab uses **`asia-south1`**.

**Key terms**

| Term | Meaning |
| --- | --- |
| **Service** | Named app (e.g. `cloudmart-catalog-api`) |
| **Revision** | Immutable snapshot of a deployed configuration |
| **URL** | `https://SERVICE-....run.app` |
| **Container** | Your app image listening on the port defined by `PORT` (default **8080**) |

---

## Part 2 — Enable APIs and prepare the project

**Time estimate:** ~5 minutes

### Step 3 — Enable required APIs

```bash
gcloud services enable run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com
```

Console: **APIs & Services → Library** → enable the same APIs if needed.

---

### Step 4 — Confirm project and region

```bash
gcloud config get-value project
gcloud config get-value run/region
```

If `run/region` is unset:

```bash
gcloud config set run/region "$REGION"
```

---

## Part 3 — Deploy a sample container (quick start)

**Time estimate:** ~10 minutes

### Step 5 — Deploy a public sample image

Google provides a sample Hello app image. Deploy it to learn the command:

```bash
gcloud run deploy "$SERVICE_HELLO" \
  --image=us-docker.pkg.dev/cloudrun/container/hello \
  --region="$REGION" \
  --platform=managed \
  --allow-unauthenticated
```

When prompted **Allow unauthenticated invocations?** answer **Yes** (for this lab).

**Console alternative:** **Cloud Run → Create service** → deploy **hello** sample from the UI.

---

### Step 6 — Test the hello service

```bash
HELLO_URL=$(gcloud run services describe "$SERVICE_HELLO" \
  --region="$REGION" --format='value(status.url)')

echo "$HELLO_URL"
curl -s "$HELLO_URL"
```

**Expected result:** A greeting message (for example **Hello World!**).

---

## Part 4 — Build and deploy the CloudMart catalog API

**Time estimate:** ~25 minutes

### Step 7 — Create application source in Cloud Shell

```bash
mkdir -p ~/cloudmart-catalog-api
cd ~/cloudmart-catalog-api

cat > requirements.txt <<'EOF'
flask==3.0.*
gunicorn==22.*
EOF

cat > main.py <<'EOF'
import os
from flask import Flask, jsonify

app = Flask(__name__)

PRODUCTS = [
    {"sku": "1001", "name": "Wireless Earbuds", "price": 49.99},
    {"sku": "1002", "name": "USB-C Hub", "price": 29.99},
    {"sku": "1003", "name": "Laptop Stand", "price": 39.99},
]

STORE_NAME = os.environ.get("STORE_NAME", "CloudMart")


@app.route("/")
def health():
    return jsonify({
        "service": "cloudmart-catalog-api",
        "store": STORE_NAME,
        "status": "ok",
    })


@app.route("/products")
def products():
    return jsonify({"store": STORE_NAME, "products": PRODUCTS})


if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
EOF

cat > Dockerfile <<'EOF'
FROM python:3.11-slim
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app
EOF
```

---

### Step 8 — Deploy from source

Cloud Build builds the image and Cloud Run deploys the service:

```bash
gcloud run deploy "$SERVICE_API" \
  --source . \
  --region="$REGION" \
  --platform=managed \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --max-instances=3 \
  --concurrency=80
```

First deploy may take **several minutes**.

**Expected result:** Command ends with **Service URL** and revision name.

---

### Step 9 — Describe the service

```bash
gcloud run services describe "$SERVICE_API" \
  --region="$REGION" \
  --format="yaml(status.url,status.latestReadyRevisionName,spec.template.spec.containers[0].image)"
```

In the Console, open **cloudmart-catalog-api** → **Revisions** and **Observability**.

---

## Part 5 — Test the service URL

**Time estimate:** ~10 minutes

### Step 10 — Call health and products endpoints

```bash
API_URL=$(gcloud run services describe "$SERVICE_API" \
  --region="$REGION" --format='value(status.url)')

curl -s "$API_URL/"
curl -s "$API_URL/products" | head -c 500
```

**Expected result:** JSON with `"status": "ok"` and a `products` array.

---

### Step 11 — Test in the browser

Open `API_URL` and `API_URL/products` in a new browser tab.

**Security note:** This lab allows **unauthenticated** access for learning. Production APIs often require **IAM**, **Identity-Aware Proxy**, or an **API gateway**.

---

## Part 6 — Configure the service (environment and scaling)

**Time estimate:** ~10 minutes

### Step 12 — Set an environment variable

Change the store display name without editing code:

```bash
gcloud run services update "$SERVICE_API" \
  --region="$REGION" \
  --set-env-vars="STORE_NAME=CloudMart India"
```

Creates a **new revision**. Test again:

```bash
curl -s "$API_URL/products"
```

**Expected result:** `"store": "CloudMart India"`.

---

### Step 13 — Scaling settings (concept)

Review current settings:

```bash
gcloud run services describe "$SERVICE_API" \
  --region="$REGION" \
  --format="yaml(spec.template.metadata.annotations)"
```

| Setting | Lab value | Meaning |
| --- | --- | --- |
| **max-instances** | 3 | Caps scale-out during spikes |
| **concurrency** | 80 | Requests per container instance |
| **min instances** | 0 (default) | Scales to zero when idle |

To reduce cold starts in production, teams sometimes set **min instances** > 0 (higher cost). This lab keeps **min = 0**.

---

## Part 7 — Revisions, logs, and monitoring

**Time estimate:** ~10 minutes

### Step 14 — List revisions

```bash
gcloud run revisions list --service="$SERVICE_API" --region="$REGION"
```

You should see at least **two** revisions after the environment variable update.

---

### Step 15 — View logs

```bash
gcloud run services logs read "$SERVICE_API" \
  --region="$REGION" \
  --limit=20
```

Console: **Cloud Run → cloudmart-catalog-api → Logs**.

Generate traffic if logs are empty:

```bash
for i in 1 2 3; do curl -s -o /dev/null -w "%{http_code}\n" "$API_URL/products"; done
```

---

### Step 16 — Roll back (optional)

Route **100% traffic** to the previous revision if needed:

```bash
PREV=$(gcloud run revisions list --service="$SERVICE_API" --region="$REGION" \
  --format='value(metadata.name)' --limit=2 | tail -1)

gcloud run services update-traffic "$SERVICE_API" \
  --region="$REGION" \
  --to-revisions="$PREV=100"
```

Restore latest when done experimenting:

```bash
LATEST=$(gcloud run services describe "$SERVICE_API" --region="$REGION" \
  --format='value(status.latestReadyRevisionName)')

gcloud run services update-traffic "$SERVICE_API" \
  --region="$REGION" \
  --to-revisions="$LATEST=100"
```

---

## Part 8 — Clean up lab resources

**Time estimate:** ~10 minutes

### Step 17 — Delete Cloud Run services

```bash
gcloud run services delete "$SERVICE_API" --region="$REGION" --quiet
gcloud run services delete "$SERVICE_HELLO" --region="$REGION" --quiet 2>/dev/null || true
```

Verify:

```bash
gcloud run services list --region="$REGION" --filter="metadata.name:cloudmart"
```

---

### Step 18 — Remove local source (optional)

```bash
rm -rf ~/cloudmart-catalog-api
```

---

### Step 19 — Optional: clean Artifact Registry images

Source deploys may leave images in an Artifact Registry repository (often `cloud-run-source-deploy` in your region). If your instructor allows cleanup:

```bash
gcloud artifacts repositories list --location="$REGION"
```

Delete unused repositories only when your instructor confirms they are lab-created and not shared.

---

## Validation checklist

- [ ] Enabled Run, Cloud Build, and Artifact Registry APIs
- [ ] Deployed sample or catalog service successfully
- [ ] Retrieved HTTPS service URL
- [ ] `GET /products` returned JSON catalog
- [ ] Updated `STORE_NAME` via environment variable (new revision)
- [ ] Viewed logs for the service
- [ ] Deleted `cloudmart-catalog-api` (and hello service if used)

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | Cloud Run services list with `cloudmart-catalog-api` deployed |
| 2 | Browser or `curl` output for `/products` |
| 3 | Revisions list showing two or more revisions |
| 4 | Logs showing HTTP requests |
| 5 | Empty services list after cleanup |

---

## Knowledge check

1. What must a Cloud Run container listen on?  
   - A) Port 22  
   - B) The port in the `PORT` environment variable (default 8080)  
   - C) Port 443 only  

2. What happens when no traffic hits a service with default min instances?  
   - A) The service scales to zero  
   - B) You must delete the service manually every night  
   - C) Cloud Run creates a new VPC  

3. Which command deploys from source in this lab?  
   - A) `gcloud run deploy --source .`  
   - B) `gcloud compute instances create`  
   - C) `gcloud sql connect`  

4. What is a **revision**?  
   - A) A billing account  
   - B) An immutable deployed configuration snapshot  
   - C) A firewall rule  

5. Why did we use `--allow-unauthenticated` in the lab?  
   - A) To test the public HTTPS URL without IAM tokens  
   - B) To disable HTTPS  
   - C) To connect to Cloud SQL automatically  

<details>
<summary><strong>Answer key</strong></summary>

1. **B** — Listen on `PORT`.  
2. **A** — Scales to zero.  
3. **A** — Source-based deploy.  
4. **B** — Revision snapshot.  
5. **A** — Public invoke for learning (not a production pattern for sensitive APIs).

</details>

---

## Optional challenge

1. Add `GET /products/<sku>` returning one product or 404.  
2. Deploy with **`--no-allow-unauthenticated`** and call the API with an **identity token** from `gcloud auth print-identity-token`.  
3. Set **`--min-instances=1`** briefly and compare cold-start behavior (remember to remove after).  
4. Describe how you would connect this API to a **managed database** in production (no extra deploy required for this lab).

---

## Official references

- [Cloud Run overview](https://cloud.google.com/run/docs/overview/what-is-cloud-run)
- [Deploy from source](https://cloud.google.com/run/docs/deploying-source-code)
- [Container contract](https://cloud.google.com/run/docs/container-contract)
- [Configure environment variables](https://cloud.google.com/run/docs/configuring/environment-variables)
- [Manage revisions](https://cloud.google.com/run/docs/managing/revisions)
- [Write and view logs](https://cloud.google.com/run/docs/logging)
- [Authenticate requests](https://cloud.google.com/run/docs/authenticating/overview)

---

## Lab outcome

You deployed a **CloudMart catalog API** on **Cloud Run**, tested HTTPS endpoints, updated configuration with a new revision, reviewed logs, and deleted services to avoid ongoing cost.

You can take other training labs in any order; none are required before or after this one.