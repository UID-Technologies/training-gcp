# Lab 12 — CI/CD: Cloud Build and Deploy to Cloud Run for CloudMart

| | |
| --- | --- |
| **Training module** | Module 12 — CI/CD on Google Cloud |
| **Companion presentation** | Presentation 12 — CI/CD on Google Cloud (Cloud Build, Artifact Registry, and Cloud Deploy) |
| **Level** | Beginner to intermediate |
| **Estimated time** | 90–120 minutes |
| **Primary services** | Cloud Build, Artifact Registry, Cloud Run |
| **Lab prefix** | `cloudmart` |
| **Region** | `asia-south1` (Mumbai) |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [Pipeline overview](#pipeline-overview)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — CI/CD and Cloud Build fundamentals](#part-1--cicd-and-cloud-build-fundamentals)
8. [Part 2 — Enable APIs and IAM for Cloud Build](#part-2--enable-apis-and-iam-for-cloud-build)
9. [Part 3 — Prepare the application and Dockerfile](#part-3--prepare-the-application-and-dockerfile)
10. [Part 4 — Author the cloudbuild.yaml pipeline](#part-4--author-the-cloudbuildyaml-pipeline)
11. [Part 5 — Run the pipeline (build, push, deploy)](#part-5--run-the-pipeline-build-push-deploy)
12. [Part 6 — Verify deployment and run a second build](#part-6--verify-deployment-and-run-a-second-build)
13. [Part 7 — Operations (build history and logs)](#part-7--operations-build-history-and-logs)
14. [Part 8 — Clean up lab resources](#part-8--clean-up-lab-resources)
15. [Validation checklist](#validation-checklist)
16. [Evidence to submit](#evidence-to-submit)
17. [Knowledge check](#knowledge-check)
18. [Optional challenge](#optional-challenge)
19. [Official references](#official-references)

---

## Lab description

In this lab, you implement a **CI/CD pipeline** for **CloudMart** using **Cloud Build**. The pipeline **builds** a container image, **pushes** it to **Artifact Registry**, and **deploys** it to **Cloud Run**—all from a **`cloudbuild.yaml`** file. You run the pipeline with **`gcloud builds submit`**, verify the live API, trigger a **second build** after a code change, and review **build logs** in the Console.

**Presentation 12** also explains **Cloud Deploy**, multi-environment **GKE** promotion, and triggers; this lab concentrates on the **build → registry → deploy** path in one environment so it stays standalone and about 90 minutes. Discuss those deck topics in class; optional challenges mention triggers and multi-stage delivery.

**Standalone lab:** You do not need any other lab in this folder. Application source, registry, build config, and Cloud Run service are created and removed here.

---

## Scenario and use case

CloudMart’s engineering team wants **repeatable releases**:

| Manual process | Automated pipeline |
| --- | --- |
| Developer builds image on a laptop | **Cloud Build** builds in GCP |
| Someone runs `docker push` | Image pushed to **Artifact Registry** |
| Ops deploys by hand | **`gcloud run deploy`** in the pipeline |

**Your role:** Define **`cloudmart-catalog-cicd`** on Cloud Run, wire **Cloud Build** permissions, ship version **v1** and **v2** of the catalog API through the same pipeline.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain **CI** (build/test) vs **CD** (deploy) on Google Cloud
- Describe **Cloud Build** steps, images, and build logs
- Grant the **Cloud Build service account** permissions to deploy to Cloud Run
- Write a **`cloudbuild.yaml`** with build, push, and deploy steps
- Run **`gcloud builds submit`** and track build status
- Verify a **Cloud Run revision** created by the pipeline
- Clean up services, images, and IAM bindings added for the lab

---

## Pipeline overview

```text
Source (Cloud Shell folder)
    |
    v
Cloud Build  --->  docker build  --->  Artifact Registry (image tag = BUILD_ID)
    |
    v
Cloud Run deploy  --->  HTTPS URL (cloudmart-catalog-cicd)
```

| Artifact | Name |
| --- | --- |
| Registry repo | `cloudmart-cicd` |
| Image | `asia-south1-docker.pkg.dev/PROJECT_ID/cloudmart-cicd/cloudmart-catalog-cicd` |
| Cloud Run service | `cloudmart-catalog-cicd` |
| Build config | `cloudbuild.yaml` |

**Cost note:** Cloud Build minutes, Artifact Registry storage, and Cloud Run usage may incur charges. Complete **Part 8** when finished.

---

## Prerequisites

Before you start, confirm:

- [ ] Training project with **billing enabled**
- [ ] Permission to edit **IAM**, run **Cloud Build**, manage **Artifact Registry** and **Cloud Run**
- [ ] **Cloud Shell** (recommended)
- [ ] Basic familiarity with **Dockerfile** syntax (revisited in this lab)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="asia-south1"
export REPO="cloudmart-cicd"
export SERVICE="cloudmart-catalog-cicd"
export PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
export CB_SA="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

gcloud config set project "$PROJECT_ID"
```

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1 | CI/CD concepts |
| 2 | APIs and IAM |
| 3–4 | App + `cloudbuild.yaml` |
| 5–6 | Submit builds and test |
| 7 | Logs and history |
| 8 | Cleanup |

Each build may take **3–8 minutes**.

---

## Part 1 — CI/CD and Cloud Build fundamentals

**Time estimate:** ~10 minutes

### Step 1 — CI vs CD

| Phase | Purpose | In this lab |
| --- | --- | --- |
| **CI (continuous integration)** | Build and validate each change | Docker **build** + unit sanity (HTTP check manually) |
| **CD (continuous delivery/deployment)** | Ship to an environment | **Push** image + **deploy** to Cloud Run |

---

### Step 2 — Cloud Build components

| Component | Role |
| --- | --- |
| **Build** | Executes steps (VM or pool) |
| **Step** | Container or `gcloud` action |
| **Substitution** | Variables like `$PROJECT_ID`, `$BUILD_ID` |
| **`cloudbuild.yaml`** | Pipeline as code |
| **Trigger** | Starts builds on Git events (optional; not required here) |

Open Console → **Cloud Build → History** (may be empty initially).

---

## Part 2 — Enable APIs and IAM for Cloud Build

**Time estimate:** ~15 minutes

### Step 3 — Enable APIs

```bash
gcloud services enable cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com
```

---

### Step 4 — Create Artifact Registry repository

```bash
gcloud artifacts repositories create "$REPO" \
  --repository-format=docker \
  --location="$REGION" \
  --description="CloudMart CI/CD images"
```

---

### Step 5 — Grant Cloud Build permission to deploy

The **Cloud Build service account** must push images and deploy to Cloud Run.

```bash
echo "Cloud Build SA: $CB_SA"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/iam.serviceAccountUser"

gcloud artifacts repositories add-iam-policy-binding "$REPO" \
  --location="$REGION" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/artifactregistry.writer"
```

**Security note:** In production, use **custom roles** or **least privilege** instead of broad project bindings where possible.

---

## Part 3 — Prepare the application and Dockerfile

**Time estimate:** ~15 minutes

### Step 6 — Create the project directory

```bash
mkdir -p ~/cloudmart-cicd && cd ~/cloudmart-cicd
```

---

### Step 7 — Application code (catalog API v1)

```bash
cat > requirements.txt <<'EOF'
flask==3.0.*
gunicorn==22.*
EOF

cat > main.py <<'EOF'
import os
from flask import Flask, jsonify

app = Flask(__name__)

API_VERSION = os.environ.get("API_VERSION", "1.0")

PRODUCTS = [
    {"sku": "1001", "name": "Wireless Earbuds", "price": 49.99},
    {"sku": "1002", "name": "USB-C Hub", "price": 29.99},
]


@app.route("/")
def health():
    return jsonify({
        "service": "cloudmart-catalog-cicd",
        "api_version": API_VERSION,
        "status": "ok",
    })


@app.route("/products")
def products():
    return jsonify({"api_version": API_VERSION, "products": PRODUCTS})


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
EOF
```

---

### Step 8 — Dockerfile

```bash
cat > Dockerfile <<'EOF'
FROM python:3.11-slim
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py .
ENV PORT=8080
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 main:app
EOF
```

Test locally only if Docker works in your environment (optional):

```bash
# docker build -t test-catalog . && docker run -p 8080:8080 test-catalog
```

---

## Part 4 — Author the cloudbuild.yaml pipeline

**Time estimate:** ~15 minutes

### Step 9 — Pipeline definition

Image tag uses **`$BUILD_ID`** so every build is unique.

```bash
cat > cloudbuild.yaml <<'EOF'
substitutions:
  _REGION: asia-south1
  _REPOSITORY: cloudmart-cicd
  _SERVICE: cloudmart-catalog-cicd

steps:
  - id: build-image
    name: gcr.io/cloud-builders/docker
    args:
      - build
      - -t
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_SERVICE}:$BUILD_ID
      - -t
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_SERVICE}:latest
      - .

  - id: push-image
    name: gcr.io/cloud-builders/docker
    args:
      - push
      - --all-tags
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_SERVICE}

  - id: deploy-cloud-run
    name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - run
      - deploy
      - ${_SERVICE}
      - --image=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_SERVICE}:$BUILD_ID
      - --region=${_REGION}
      - --platform=managed
      - --allow-unauthenticated
      - --set-env-vars=API_VERSION=1.0
      - --quiet

images:
  - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_SERVICE}:$BUILD_ID
  - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPOSITORY}/${_SERVICE}:latest

options:
  logging: CLOUD_LOGGING_ONLY
EOF
```

| Step | Action |
| --- | --- |
| `build-image` | `docker build` with two tags |
| `push-image` | Push to Artifact Registry |
| `deploy-cloud-run` | New Cloud Run revision |

---

## Part 5 — Run the pipeline (build, push, deploy)

**Time estimate:** ~15 minutes

### Step 10 — Submit the build

```bash
cd ~/cloudmart-cicd

gcloud builds submit . --config=cloudbuild.yaml
```

**Expected result:** Build status **SUCCESS**; final step prints the Cloud Run service URL.

---

### Step 11 — Confirm image and service

```bash
gcloud artifacts docker images list \
  "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${SERVICE}"

gcloud run services describe "$SERVICE" --region="$REGION" \
  --format="yaml(status.url,status.latestReadyRevisionName)"
```

---

## Part 6 — Verify deployment and run a second build

**Time estimate:** ~15 minutes

### Step 12 — Test the API

```bash
URL=$(gcloud run services describe "$SERVICE" --region="$REGION" \
  --format='value(status.url)')

curl -s "${URL}/"
curl -s "${URL}/products"
```

**Expected result:** `"api_version": "1.0"` in JSON.

---

### Step 13 — Change application (v2) and rebuild

Add a product and bump the default version in code:

```bash
# Append product 1003 in main.py (edit manually or use sed)
# Set API_VERSION default to 2.0 in main.py for clarity

cat > main.py <<'EOF'
import os
from flask import Flask, jsonify

app = Flask(__name__)

API_VERSION = os.environ.get("API_VERSION", "2.0")

PRODUCTS = [
    {"sku": "1001", "name": "Wireless Earbuds", "price": 49.99},
    {"sku": "1002", "name": "USB-C Hub", "price": 29.99},
    {"sku": "1003", "name": "Laptop Stand", "price": 39.99},
]


@app.route("/")
def health():
    return jsonify({
        "service": "cloudmart-catalog-cicd",
        "api_version": API_VERSION,
        "status": "ok",
    })


@app.route("/products")
def products():
    return jsonify({"api_version": API_VERSION, "products": PRODUCTS})


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
EOF
```

Update deploy env in `cloudbuild.yaml` for v2:

```bash
sed -i 's/API_VERSION=1.0/API_VERSION=2.0/' cloudbuild.yaml
```

Submit again:

```bash
gcloud builds submit . --config=cloudbuild.yaml
curl -s "${URL}/products" | head -c 400
```

**Expected result:** Three products; `api_version` **2.0** (from env var).

List Cloud Run revisions:

```bash
gcloud run revisions list --service="$SERVICE" --region="$REGION"
```

---

## Part 7 — Operations (build history and logs)

**Time estimate:** ~10 minutes

### Step 14 — Console build history

1. **Cloud Build → History** — open the last two builds.
2. Expand steps **build-image**, **push-image**, **deploy-cloud-run**.
3. Open **View in Cloud Logging** for a failed step (if instructor demos failure).

---

### Step 15 — CLI build list

```bash
gcloud builds list --limit=5
BUILD_ID=$(gcloud builds list --limit=1 --format='value(id)')
gcloud builds describe "$BUILD_ID"
```

Discuss **triggers** (GitHub, Cloud Source Repositories) as the next maturity step—see **Optional challenge**.

---

## Part 8 — Clean up lab resources

**Time estimate:** ~15 minutes

### Step 16 — Delete Cloud Run service

```bash
gcloud run services delete "$SERVICE" --region="$REGION" --quiet
```

---

### Step 17 — Delete container images (optional before repo delete)

```bash
gcloud artifacts docker images delete \
  "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${SERVICE}" \
  --delete-tags --quiet 2>/dev/null || true
```

---

### Step 18 — Delete Artifact Registry repository

```bash
gcloud artifacts repositories delete "$REPO" \
  --location="$REGION" --quiet
```

---

### Step 19 — Remove lab IAM bindings (if policy allows)

Only if your instructor wants a pristine project:

```bash
gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/run.admin" 2>/dev/null || true

gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/iam.serviceAccountUser" 2>/dev/null || true
```

Repository IAM is removed when the repo is deleted.

---

### Step 20 — Remove local sources

```bash
rm -rf ~/cloudmart-cicd
```

Verify:

```bash
gcloud run services list --region="$REGION" --filter="metadata.name:cloudmart"
gcloud artifacts repositories list --location="$REGION" --filter="name:cloudmart"
```

---

## Validation checklist

- [ ] Enabled Cloud Build, Artifact Registry, Cloud Run APIs
- [ ] Granted Cloud Build SA deploy and push permissions
- [ ] Created `cloudbuild.yaml` with build, push, deploy steps
- [ ] Successful `gcloud builds submit`
- [ ] API reachable over HTTPS; v1 then v2 behavior verified
- [ ] Reviewed build history in Console
- [ ] Deleted Cloud Run service and Artifact Registry repo

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | `cloudbuild.yaml` snippet (three steps) |
| 2 | Cloud Build **SUCCESS** with step durations |
| 3 | Artifact Registry image list with tags |
| 4 | `curl` output showing `api_version` 2.0 and three SKUs |
| 5 | Cloud Run revisions list (2+ revisions) |
| 6 | Proof of cleanup |

---

## Knowledge check

1. What file defines Cloud Build steps in this lab?  
   - A) `cloudbuild.yaml`  
   - B) `nginx.conf`  
   - C) `firewall-rules.json`  

2. Which identity usually runs your pipeline steps in the project?  
   - A) Cloud Build service account  
   - B) `allUsers`  
   - C) Bigtable system account  

3. What does the **push** step do?  
   - A) Uploads the image to Artifact Registry  
   - B) Deletes the VPC  
   - C) Creates a GKE cluster  

4. How do you start a manual pipeline run from source in Cloud Shell?  
   - A) `gcloud builds submit`  
   - B) `gcloud sql import`  
   - C) `kubectl delete cluster`  

5. Why tag images with `$BUILD_ID`?  
   - A) Unique, traceable image per build  
   - B) Required for BigQuery partitioning  
   - C) Disables HTTPS  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — `cloudbuild.yaml`.  
2. **A** — Project Cloud Build service account.  
3. **A** — Push to registry.  
4. **A** — `gcloud builds submit`.  
5. **A** — Traceability per build.

</details>

---

## Optional challenge

1. Add a **Cloud Build trigger** on a **Cloud Source Repository** mirroring your Cloud Shell source.  
2. Insert a step that runs **`pip install -r requirements.txt`** tests or `python -m compileall` before the Docker build (CI quality gate).  
3. Deploy to **GKE** instead of Cloud Run in a second `cloudbuild.yaml` target (instructor sandbox only).  
4. Use **Secret Manager** for runtime secrets instead of plain env vars.

---

## Official references

- [Cloud Build overview](https://cloud.google.com/build/docs/overview)
- [Create a build config file](https://cloud.google.com/build/docs/build-config)
- [Cloud Build service account](https://cloud.google.com/build/docs/cloud-build-service-account)
- [Build and push to Artifact Registry](https://cloud.google.com/build/docs/building/build-containers)
- [Deploying to Cloud Run](https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run)
- [`gcloud builds submit`](https://cloud.google.com/sdk/gcloud/reference/builds/submit)
- [Artifact Registry](https://cloud.google.com/artifact-registry/docs)

---

## Lab outcome

You implemented a **Cloud Build CI/CD pipeline** for CloudMart: **build** container images, **push** to **Artifact Registry**, **deploy** to **Cloud Run**, validated **v1** and **v2** releases, and **cleaned up** GCP resources.

You can take other training labs in any order; none are required before or after this one.