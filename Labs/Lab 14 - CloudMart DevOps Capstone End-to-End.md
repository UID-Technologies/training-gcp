# Lab 14 — CloudMart DevOps Capstone: End-to-End Retail API

| | |
| --- | --- |
| **Training module** | Module 14 — GCP DevOps capstone |
| **Companion presentation** | Presentation 14 — GCP DevOps Capstone: Deploying a Retail API End to End |
| **Level** | Intermediate to advanced |
| **Estimated time** | 4–6 hours (can split across sessions) |
| **Primary tools** | Terraform, Docker, Cloud Build, GKE, Artifact Registry, kubectl |
| **Lab prefix** | `cloudmart` |
| **Region** | `asia-south1` (Mumbai) |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [Architecture](#architecture)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Capstone overview and repository layout](#part-1--capstone-overview-and-repository-layout)
8. [Part 2 — Build the Retail API and Redis](#part-2--build-the-retail-api-and-redis)
9. [Part 3 — Containerize and test with Docker Compose](#part-3--containerize-and-test-with-docker-compose)
10. [Part 4 — Provision foundation with Terraform](#part-4--provision-foundation-with-terraform)
11. [Part 5 — Push images to Artifact Registry](#part-5--push-images-to-artifact-registry)
12. [Part 6 — Deploy workloads to GKE](#part-6--deploy-workloads-to-gke)
13. [Part 7 — Automate with Cloud Build](#part-7--automate-with-cloud-build)
14. [Part 8 — Logging, monitoring, and verification](#part-8--logging-monitoring-and-verification)
15. [Part 9 — Release, troubleshoot, and roll back](#part-9--release-troubleshoot-and-roll-back)
16. [Part 10 — Capstone deliverables and grading rubric](#part-10--capstone-deliverables-and-grading-rubric)
17. [Part 11 — Clean up all resources](#part-11--clean-up-all-resources)
18. [Validation checklist](#validation-checklist)
19. [Knowledge check](#knowledge-check)
20. [Official references](#official-references)

---

## Lab description

This **capstone** ties together **Infrastructure as Code**, **containers**, **Artifact Registry**, **GKE**, and **CI/CD** for **CloudMart’s Retail API**. You provision a **GKE Autopilot** cluster with **Terraform**, deploy an **API** and **Redis** on Kubernetes, automate **build and deploy** with **Cloud Build**, and demonstrate **health checks**, **logs**, and **rollback**. The flow aligns with **Presentation 14** (simplified where noted for a single training project).

**Standalone lab:** All source, Terraform, manifests, and pipelines are created in Cloud Shell during this lab. You do not need to complete Labs 01–13 first, but familiarity with VMs, networking, Docker, Kubernetes, and Terraform helps.

**Presentation 14** uses a **Node.js** and **Express** Retail API with **Redis**, **Helm**, and **GitHub triggers**. This guide implements the same API in **Node.js 22** with **`kubectl apply`** and **`gcloud builds submit`**; **Helm** and **Git triggers** are optional extensions.

---

## Scenario and use case

You are a **GCP DevOps engineer** for CloudMart. Leadership wants a **standardized platform**:

| Before | After (capstone outcome) |
| --- | --- |
| Manual deploys | **Terraform** foundation |
| Snowflake servers | **Containers** on **GKE** |
| Ad hoc releases | **Cloud Build** pipeline |
| Blind production | **Logging** and health endpoints |

**Success:** A working API behind **HTTPS/LB** (or port-forward for lab), deployed at least once through **Cloud Build**, with evidence captured for the rubric.

---

## Learning objectives

When you finish this capstone, you should be able to:

- Organize a repo with **src**, **terraform**, **k8s**, and **cloudbuild.yaml**
- Run **Redis** and an API locally with **Docker Compose**
- Provision **VPC + subnet + GKE Autopilot + Artifact Registry** with **Terraform**
- Build, tag, and push images to **Artifact Registry**
- Deploy **Deployments**, **Services**, probes, and **ConfigMaps** on GKE
- Create a **Cloud Build** pipeline that builds and deploys (or build + manual kubectl)
- Verify **`/health`**, **`/ready`**, **`/products`**, and **`/visits`**
- Troubleshoot a failed rollout and **roll back**
- **Destroy** infrastructure with Terraform and remove cluster workloads

---

## Architecture

```text
Git / Cloud Shell source
        |
        v
Cloud Build ----> Docker build ----> Artifact Registry
        |                                    |
        +----> (optional) terraform plan     |
        |                                    v
        +----------------------------> GKE (Autopilot)
                                              |
                         retail-api + redis <--+
                                              |
                         Cloud Logging / Monitoring
```

| Layer | Tools |
| --- | --- |
| Infrastructure | Terraform (`terraform/`) |
| Application | Node.js (Express) API + Redis |
| Delivery | Cloud Build, Artifact Registry |
| Runtime | GKE Autopilot, kubectl |

---

## Prerequisites

- [ ] Training project with **billing enabled**
- [ ] Roles: Compute, Kubernetes Engine, Cloud Build, Artifact Registry, Storage, Service Account User
- [ ] **4–6 hours** uninterrupted or split schedule
- [ ] Presentation **14** reviewed
- [ ] Cloud Shell with **Terraform**, **kubectl**, **Docker** (Cloud Build runs Docker for pipeline)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="asia-south1"
export ZONE="asia-south1-a"
export CLUSTER="cloudmart-capstone-gke"
export REPO="cloudmart-retail-images"
export NAMESPACE="cloudmart-dev"

gcloud config set project "$PROJECT_ID"
```

---

## How to work through this lab

| Part | Time (guide) |
| --- | --- |
| 1–2 | 45–60 min |
| 3 | 30 min |
| 4 | 45–60 min |
| 5–6 | 60–90 min |
| 7–9 | 60–90 min |
| 10–11 | 30–45 min |

---

## Part 1 — Capstone overview and repository layout

**Time estimate:** ~20 minutes

### Step 1 — Create project structure

```bash
mkdir -p ~/cloudmart-capstone/{src,terraform,k8s}
cd ~/cloudmart-capstone
```

| Path | Responsibility |
| --- | --- |
| `src/` | API application code |
| `terraform/` | VPC, GKE, Artifact Registry |
| `k8s/` | Kubernetes manifests |
| `package.json` / `package-lock.json` | Node.js dependencies |
| `Dockerfile` | API container (`node:22-alpine`) |
| `docker-compose.yaml` | Local API + Redis |
| `cloudbuild.yaml` | CI/CD pipeline (`npm ci`, `npm test`) |

This mirrors **Presentation 14** (repo with `src`, `terraform`, `k8s`, Dockerfile, cloudbuild).

---

## Part 2 — Build the Retail API and Redis

**Time estimate:** ~40 minutes

### Step 2 — API code (Node.js and Express)

```bash
mkdir -p src test

cat > package.json <<'EOF'
{
  "name": "cloudmart-retail-api",
  "version": "1.0.0",
  "private": true,
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "test": "node --test test/*.test.js"
  },
  "dependencies": {
    "express": "^4.21.0",
    "redis": "^4.7.0"
  }
}
EOF

cat > src/index.js <<'EOF'
const express = require("express");
const { createClient } = require("redis");

const PORT = Number(process.env.PORT || 8080);
const API_VERSION = process.env.API_VERSION || "1.0.0";
const REDIS_HOST = process.env.REDIS_HOST || "localhost";
const REDIS_PORT = process.env.REDIS_PORT || "6379";

const PRODUCTS = [
  { sku: "1001", name: "Wireless Earbuds", price: 49.99 },
  { sku: "1002", name: "USB-C Hub", price: 29.99 },
];

function log(level, msg, extra = {}) {
  console.log(
    JSON.stringify({
      level,
      msg,
      service: "cloudmart-retail-api",
      ...extra,
    })
  );
}

const app = express();
let redisClient;

async function getRedis() {
  if (!redisClient) {
    redisClient = createClient({
      url: `redis://${REDIS_HOST}:${REDIS_PORT}`,
    });
    redisClient.on("error", (err) =>
      log("error", "redis_client_error", { error: String(err) })
    );
    await redisClient.connect();
  }
  return redisClient;
}

app.get("/", (req, res) => {
  res.json({ service: "cloudmart-retail-api", version: API_VERSION });
});

app.get("/products", (req, res) => {
  res.json({ products: PRODUCTS });
});

app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

app.get("/ready", async (req, res) => {
  try {
    const client = await getRedis();
    await client.ping();
    res.json({ status: "ready" });
  } catch (err) {
    res.status(503).json({ status: "not_ready" });
  }
});

app.get("/version", (req, res) => {
  res.json({ version: API_VERSION });
});

app.get("/visits", async (req, res) => {
  try {
    const client = await getRedis();
    const visits = await client.incr("visits");
    res.json({ visits });
  } catch (err) {
    log("error", "visits_failed", { error: String(err) });
    res.status(500).json({ error: String(err) });
  }
});

app.listen(PORT, () => {
  log("info", "server_started", { port: PORT, version: API_VERSION });
});
EOF

cat > test/smoke.test.js <<'EOF'
const test = require("node:test");
const assert = require("node:assert");

test("catalog has products", () => {
  const products = [
    { sku: "1001", name: "Wireless Earbuds", price: 49.99 },
    { sku: "1002", name: "USB-C Hub", price: 29.99 },
  ];
  assert.equal(products.length, 2);
});
EOF
```

Generate **`package-lock.json`** (required for `npm ci` in Docker and Cloud Build):

```bash
cd ~/cloudmart-capstone
npm install
npm test
```

Endpoints match **Presentation 14** (`/`, `/products`, `/health`, `/ready`, `/version`, `/visits`). Logs are **JSON** to stdout for Cloud Logging.

---

## Part 3 — Containerize and test with Docker Compose

**Time estimate:** ~30 minutes

### Step 3 — Dockerfile

```bash
cat > Dockerfile <<'EOF'
FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY src ./src
RUN addgroup -g 1001 app && adduser -D -u 1001 -G app app
USER app
ENV PORT=8080
EXPOSE 8080
CMD ["node", "src/index.js"]
EOF
```

This matches **Presentation 14** (`node:22-alpine`, `npm ci`, non-root user).

---

### Step 4 — docker-compose.yaml

```bash
cat > docker-compose.yaml <<'EOF'
services:
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - redis-data:/data

  retail-api:
    build: .
    ports:
      - "8080:8080"
    environment:
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      API_VERSION: "1.0.0"
    depends_on:
      redis:
        condition: service_healthy

volumes:
  redis-data:
EOF
```

Run locally **if Docker is available**:

```bash
docker compose up -d --build
docker compose ps
curl -s http://localhost:8080/health
curl -s http://localhost:8080/visits
docker compose down
```

If Docker is unavailable in Cloud Shell, skip to Part 4; validate on GKE later.

---

## Part 4 — Provision foundation with Terraform

**Time estimate:** ~60 minutes

### Step 5 — Terraform files

```bash
cd ~/cloudmart-capstone/terraform

cat > versions.tf <<'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = { source = "hashicorp/google", version = "~> 6.0" }
  }
}
EOF

cat > variables.tf <<'EOF'
variable "project_id" { type = string }
variable "region" { type = string, default = "asia-south1" }
EOF

cat > terraform.tfvars <<EOF
project_id = "${PROJECT_ID}"
region     = "${REGION}"
EOF

cat > main.tf <<'EOF'
provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_compute_network" "retail_vpc" {
  name                    = "cloudmart-retail-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "retail_subnet" {
  name          = "cloudmart-retail-subnet"
  ip_cidr_range = "10.10.0.0/24"
  region        = var.region
  network       = google_compute_network.retail_vpc.id
}

resource "google_container_cluster" "retail_gke" {
  name     = "cloudmart-capstone-gke"
  location = var.region

  enable_autopilot = true
  network          = google_compute_network.retail_vpc.id
  subnetwork       = google_compute_subnetwork.retail_subnet.id

  deletion_protection = false
}

resource "google_artifact_registry_repository" "retail_images" {
  repository_id = "cloudmart-retail-images"
  format        = "DOCKER"
  location      = var.region
}
EOF

cat > outputs.tf <<'EOF'
output "cluster_name" {
  value = google_container_cluster.retail_gke.name
}
output "artifact_registry" {
  value = google_artifact_registry_repository.retail_images.repository_id
}
EOF
```

**Note:** Presentation 14 shows secondary IP ranges for pods/services on Standard GKE; **Autopilot** manages pod networking for this simplified capstone.

---

### Step 6 — Apply Terraform

```bash
terraform init
terraform fmt
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

```bash
gcloud container clusters get-credentials cloudmart-capstone-gke --region="$REGION"
kubectl get nodes
```

---

## Part 5 — Push images to Artifact Registry

**Time estimate:** ~30 minutes

### Step 7 — Build and push API image

```bash
cd ~/cloudmart-capstone

gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet

IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/retail-api:capstone-v1"

docker build -t "$IMAGE" .
docker push "$IMAGE"
```

List images:

```bash
gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}"
```

---

## Part 6 — Deploy workloads to GKE

**Time estimate:** ~60 minutes

### Step 8 — Kubernetes manifests

Replace `IMAGE_URI` in the deployment with your pushed image URL.

```bash
cd ~/cloudmart-capstone/k8s
export IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/retail-api:capstone-v1"

cat > namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: ${NAMESPACE}
EOF

cat > configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: retail-api-config
  namespace: ${NAMESPACE}
data:
  API_VERSION: "1.0.0"
  REDIS_HOST: "redis"
  REDIS_PORT: "6379"
EOF

cat > redis.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: cloudmart-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: cloudmart-dev
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
EOF

cat > retail-api.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
  namespace: ${NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: retail-api
  template:
    metadata:
      labels:
        app: retail-api
    spec:
      containers:
        - name: api
          image: ${IMAGE_URI}
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: retail-api-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: retail-api-service
  namespace: ${NAMESPACE}
spec:
  type: LoadBalancer
  selector:
    app: retail-api
  ports:
    - port: 80
      targetPort: 8080
EOF
```

Fix namespace in redis.yaml if you changed `NAMESPACE` variable—or keep `cloudmart-dev` consistently.

---

### Step 9 — Apply and test

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f redis.yaml
kubectl apply -f retail-api.yaml

kubectl rollout status deployment/redis -n "$NAMESPACE"
kubectl rollout status deployment/retail-api -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE"
```

```bash
EXTERNAL_IP=$(kubectl get svc retail-api-service -n "$NAMESPACE" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://${EXTERNAL_IP}"

curl -s "http://${EXTERNAL_IP}/products"
curl -s "http://${EXTERNAL_IP}/health"
curl -s "http://${EXTERNAL_IP}/visits"
```

---

## Part 7 — Automate with Cloud Build

**Time estimate:** ~45 minutes

### Step 10 — cloudbuild.yaml

Pipeline aligns with **Presentation 14** (install → test → build → push → deploy). Tests are minimal here.

```bash
cd ~/cloudmart-capstone

cat > cloudbuild.yaml <<'EOF'
substitutions:
  _REGION: asia-south1
  _REPO: cloudmart-retail-images
  _IMAGE: retail-api
  _CLUSTER: cloudmart-capstone-gke
  _NAMESPACE: cloudmart-dev

steps:
  - id: npm-ci
    name: node:22
    entrypoint: npm
    args: ["ci"]

  - id: npm-test
    name: node:22
    entrypoint: npm
    args: ["test"]

  - id: build
    name: gcr.io/cloud-builders/docker
    args:
      - build
      - -t
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_IMAGE}:$BUILD_ID
      - .

  - id: push
    name: gcr.io/cloud-builders/docker
    args:
      - push
      - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_IMAGE}:$BUILD_ID

  - id: deploy
    name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: bash
    args:
      - -c
      - |
        gcloud container clusters get-credentials ${_CLUSTER} --region=${_REGION}
        kubectl set image deployment/retail-api \
          api=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_IMAGE}:$BUILD_ID \
          -n ${_NAMESPACE}
        kubectl rollout status deployment/retail-api -n ${_NAMESPACE}

images:
  - ${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_IMAGE}:$BUILD_ID

options:
  logging: CLOUD_LOGGING_ONLY
EOF
```

Grant Cloud Build cluster access (if not already from Lab 12 pattern):

```bash
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
CB_SA="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/container.developer"
```

---

### Step 11 — Run pipeline after a code change

Bump version in `configmap.yaml` and default in `src/index.js` if needed (`API_VERSION=1.1.0`).

```bash
gcloud builds submit . --config=cloudbuild.yaml
curl -s "http://${EXTERNAL_IP}/version"
```

---

## Part 8 — Logging, monitoring, and verification

**Time estimate:** ~30 minutes

### Step 12 — Cloud Logging

```bash
gcloud logging read \
  'resource.type="k8s_container"
   resource.labels.namespace_name="cloudmart-dev"
   resource.labels.cluster_name="cloudmart-capstone-gke"' \
  --limit=15 \
  --format="table(timestamp,textPayload)"
```

Console: **Kubernetes Engine → Workloads → retail-api → Logs**.

---

### Step 13 — Monitoring (Console)

Open **Monitoring → Dashboards** or **GKE workload metrics** and note CPU/memory for `retail-api` pods (illustrative dashboard in Presentation 14).

---

## Part 9 — Release, troubleshoot, and roll back

**Time estimate:** ~30 minutes

### Step 14 — Simulate a bad image (optional)

```bash
kubectl set image deployment/retail-api api=nginx:does-not-exist -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE"
kubectl describe pod -n "$NAMESPACE" -l app=retail-api | tail -20
kubectl rollout undo deployment/retail-api -n "$NAMESPACE"
kubectl rollout status deployment/retail-api -n "$NAMESPACE"
```

---

### Step 15 — Helm (optional, Presentation 14)

If your instructor provides a **Helm chart**, run:

```bash
helm upgrade --install retail-api ./helm/retail-api -n "$NAMESPACE" --atomic
```

This lab uses raw YAML; Helm is the production-oriented extension from the deck.

---

## Part 10 — Capstone deliverables and grading rubric

**Time estimate:** ~20 minutes (evidence collection)

Adapted from **Presentation 14** (100 points total):

| Deliverable | Points | Evidence |
| --- | --- | --- |
| Repository layout (`src`, `terraform`, `k8s`, Dockerfile, cloudbuild) | 10 | Tree listing or screenshot |
| Docker / Compose local test | 10 | `docker compose ps` or skip note |
| Artifact Registry image | 10 | `gcloud artifacts docker images list` |
| Kubernetes YAML applied | 10 | `kubectl get all -n cloudmart-dev` |
| GKE app responding | 15 | `curl` `/products` and `/visits` |
| Cloud Build pipeline run | 20 | Build SUCCESS screenshot |
| Terraform foundation | 10 | `terraform output` |
| Health/readiness probes working | 5 | `kubectl describe deploy retail-api` |
| Logs or monitoring snapshot | 5 | Logging query output |
| **Demo:** one pipeline deploy after a commit | 15 | Brief screen recording or instructor sign-off |

**Instructor demo goal (deck):** Commit a change; show **automated deployment** without manual image retagging.

---

## Part 11 — Clean up all resources

**Time estimate:** ~45 minutes

### Step 16 — Remove Kubernetes workloads

```bash
kubectl delete namespace "$NAMESPACE" --wait
```

---

### Step 17 — Terraform destroy

```bash
cd ~/cloudmart-capstone/terraform
terraform destroy
```

Confirm **GKE**, **VPC**, and **Artifact Registry** removal. Delete any remaining images if the repository fails to delete.

---

### Step 18 — Remove local project

```bash
rm -rf ~/cloudmart-capstone
```

Verify:

```bash
gcloud container clusters list --filter="name:cloudmart-capstone"
gcloud compute networks list --filter="name:cloudmart-retail"
```

---

## Validation checklist

- [ ] Created capstone directory structure
- [ ] API exposes required endpoints including `/visits` with Redis
- [ ] Terraform applied VPC, GKE Autopilot, Artifact Registry
- [ ] Image pushed and deployed with 2 API replicas
- [ ] LoadBalancer returns product catalog
- [ ] Cloud Build pipeline succeeded at least once
- [ ] Performed rollout undo exercise
- [ ] Captured rubric evidence
- [ ] **`terraform destroy`** and namespace deleted

---

## Knowledge check

1. What are the three capstone layers in Presentation 14?  
   - A) Infrastructure provisioning, application delivery, runtime operations  
   - B) Email, chat, calendar  
   - C) Only BigQuery  

2. Why run Redis with the Retail API?  
   - A) Visit counter and readiness checks via shared state  
   - B) Replace GKE  
   - C) Host static HTML only  

3. What should trigger the full pipeline in the reference architecture?  
   - A) Merge to `main` (or manual `gcloud builds submit` in this lab)  
   - B) Delete the VPC  
   - C) Disable logging  

4. Which command rolls back a bad Kubernetes Deployment?  
   - A) `kubectl rollout undo`  
   - B) `terraform fmt`  
   - C) `bq rm`  

5. Final cleanup order recommended in this lab?  
   - A) Delete workloads, then `terraform destroy`  
   - B) Delete project first  
   - C) Leave GKE running  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — Three layers.  
2. **A** — Redis for visits/readiness.  
3. **A** — CI/CD on main (manual submit acceptable in lab).  
4. **A** — `rollout undo`.  
5. **A** — Workloads then Terraform.

</details>

---

## Official references

- [Terraform on Google Cloud](https://cloud.google.com/docs/terraform)
- [GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)
- [Cloud Build](https://cloud.google.com/build/docs)
- [Artifact Registry](https://cloud.google.com/artifact-registry/docs)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- Presentation 14 — *GCP DevOps Capstone: Deploying a Retail API End to End* (course materials)

---

## Lab outcome

You delivered **CloudMart’s Retail API** end to end: **Terraform** foundation, **GKE** runtime, **Redis**-backed endpoints, **Artifact Registry** images, **Cloud Build** automation, observability, rollback practice, and **full cleanup**.

You can take earlier labs in any order; this capstone **reuses patterns** from them but remains self-contained.