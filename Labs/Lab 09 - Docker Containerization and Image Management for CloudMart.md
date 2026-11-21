# Lab 09 — Docker: Containerization and Image Management for CloudMart

| | |
| --- | --- |
| **Training module** | Module 9 — Docker and containerization |
| **Companion presentation** | Presentation 09 — Docker and Containerization Fundamentals |
| **Level** | Beginner |
| **Estimated time** | 75–90 minutes |
| **Primary tools** | Docker, Artifact Registry |
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
7. [Part 1 — Container and Docker fundamentals](#part-1--container-and-docker-fundamentals)
8. [Part 2 — Prepare Cloud Shell and verify Docker](#part-2--prepare-cloud-shell-and-verify-docker)
9. [Part 3 — Run and inspect containers](#part-3--run-and-inspect-containers)
10. [Part 4 — Author a Dockerfile for CloudMart](#part-4--author-a-dockerfile-for-cloudmart)
11. [Part 5 — Build, tag, and list images](#part-5--build-tag-and-list-images)
12. [Part 6 — Push images to Artifact Registry](#part-6--push-images-to-artifact-registry)
13. [Part 7 — Pull, run, logs, and exec](#part-7--pull-run-logs-and-exec)
14. [Part 8 — Clean up lab resources](#part-8--clean-up-lab-resources)
15. [Validation checklist](#validation-checklist)
16. [Evidence to submit](#evidence-to-submit)
17. [Knowledge check](#knowledge-check)
18. [Optional challenge](#optional-challenge)
19. [Official references](#official-references)

---

## Lab description

In this lab, you **containerize** a small **CloudMart storefront** web application using **Docker**. You run official images, write a **Dockerfile**, **build** a custom image, **tag** it, **push** it to **Google Artifact Registry**, and **run** the image from the registry. You practice everyday commands: **logs**, **exec**, and **cleanup**.

This lab matches **Presentation 09** (architecture, Dockerfile, image management). It focuses on **Docker mechanics**, not Kubernetes or GKE (see Presentations 10–11).

**Standalone lab:** You do not need any other lab in this folder. Containers, images, and the Artifact Registry repository are created and removed here.

---

## Scenario and use case

CloudMart developers package the storefront so it runs the same on a laptop, in CI, and later on **GKE** or **Cloud Run**.

| Problem | Container approach |
| --- | --- |
| “Works on my machine” | Image bundles app + runtime |
| Repeatable releases | Immutable **tags** (`1.0`, `1.1`) |
| GCP deployment | Push to **Artifact Registry**, pull from GCP services |

**Your role:** Build `cloudmart-storefront:1.0`, publish it to Artifact Registry, run it on port **8080**, verify the page, then remove containers and images.

---

## Learning objectives

When you finish this lab, you should be able to:

- Compare **containers** and **virtual machines**
- Explain **image**, **container**, **tag**, and **registry**
- Run containers with `docker run` (ports, detach, names)
- Write a multi-step **Dockerfile** (`FROM`, `COPY`, `EXPOSE`, `CMD`)
- Build and tag images with `docker build` and `docker tag`
- Create a **Docker** Artifact Registry repository and push images
- Inspect logs and open a shell with `docker logs` and `docker exec`
- Remove containers and images to free space

---

## What you will build

| Item | Value |
| --- | --- |
| Image name (local) | `cloudmart-storefront:1.0` |
| Container name | `cloudmart-storefront-run` |
| Listen port | Host `8080` → container `8080` |
| Base image | `nginx:alpine` (or similar) |
| Registry repo | `cloudmart-docker` |
| Remote image | `asia-south1-docker.pkg.dev/PROJECT_ID/cloudmart-docker/cloudmart-storefront:1.0` |

**Cost note:** Artifact Registry storage is small for one image. Delete the repository in Part 8 if your instructor allows.

---

## Prerequisites

Before you start, confirm:

- [ ] Training **Google Cloud project** with **billing enabled**
- [ ] Permission to create **Artifact Registry** repos and use **Cloud Shell**
- [ ] **Artifact Registry API** and **Cloud Shell** access
- [ ] Slides from **Presentation 09** reviewed (optional but helpful)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="asia-south1"
export REPO="cloudmart-docker"
export IMAGE_NAME="cloudmart-storefront"
export IMAGE_TAG="1.0"
export CONTAINER="cloudmart-storefront-run"

gcloud config set project "$PROJECT_ID"
```

**Where to run Docker:** Use **Cloud Shell** for this lab (Docker is preinstalled). Local Docker Desktop is fine if you configure `gcloud auth configure-docker` for your registry region.

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1 | Concepts |
| 2–3 | Run sample containers |
| 4–5 | Dockerfile and build |
| 6–7 | Registry push and run |
| 8 | Cleanup |

---

## Part 1 — Container and Docker fundamentals

**Time estimate:** ~10 minutes

### Step 1 — VMs vs containers

| Virtual machine | Container |
| --- | --- |
| Full OS per VM | Shares host kernel |
| Heavier, slower boot | Lightweight, fast start |
| Hypervisor / Compute Engine | **Docker** / container runtime |

A **container image** is a read-only template; a **running container** is an instance of that image with a writable layer.

---

### Step 2 — Docker workflow (this lab)

```text
Dockerfile  --build-->  Image  --run-->  Container
                              |
                              +--tag/push-->  Artifact Registry
```

---

## Part 2 — Prepare Cloud Shell and verify Docker

**Time estimate:** ~5 minutes

### Step 3 — Open Cloud Shell and enable APIs

```bash
gcloud services enable artifactregistry.googleapis.com
docker version
```

**Expected result:** Client and Server versions print without error.

If Server fails in a restricted environment, ask your instructor; this lab assumes Cloud Shell **Docker daemon** is available.

---

### Step 4 — Create a working directory

```bash
mkdir -p ~/cloudmart-docker && cd ~/cloudmart-docker
```

---

## Part 3 — Run and inspect containers

**Time estimate:** ~15 minutes

### Step 5 — Run the hello-world container

```bash
docker run --rm hello-world
```

**Expected result:** Message confirming Docker pulled and ran the image.

---

### Step 6 — Run nginx temporarily with port mapping

```bash
docker run --rm -d -p 8080:80 --name cloudmart-nginx-test nginx:alpine
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
docker stop cloudmart-nginx-test
```

**Why:** `-p 8080:80` maps **host port 8080** to **container port 80**. `--rm` removes the container when it stops.

---

### Step 7 — Inspect images and containers

```bash
docker images
docker ps -a
```

| Command | Purpose |
| --- | --- |
| `docker images` | Local images |
| `docker ps` | Running containers |
| `docker ps -a` | All containers |
| `docker inspect IMAGE` | Metadata (JSON) |

---

## Part 4 — Author a Dockerfile for CloudMart

**Time estimate:** ~15 minutes

### Step 8 — Create the storefront HTML

```bash
cd ~/cloudmart-docker
mkdir -p html

cat > html/index.html <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CloudMart Storefront</title>
  <style>
    body { font-family: sans-serif; text-align: center; padding: 48px; background: #f4f7fb; }
    h1 { color: #1a73e8; }
    .card { background: #fff; padding: 32px; border-radius: 12px; max-width: 560px; margin: auto; }
  </style>
</head>
<body>
  <div class="card">
    <h1>CloudMart</h1>
    <p>Module 9 — Docker Lab</p>
    <p>Served from an nginx container image.</p>
    <p>Version: 1.0</p>
  </div>
</body>
</html>
EOF
```

---

### Step 9 — Write the Dockerfile

```bash
cat > Dockerfile <<'EOF'
# CloudMart storefront - lab image
FROM nginx:alpine
COPY html/index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

| Instruction | Role |
| --- | --- |
| `FROM` | Base image |
| `COPY` | Application files into image |
| `EXPOSE` | Documents the port (documentation; `-p` still required at run) |
| `CMD` | Process that keeps the container running |

**Best practice:** Keep layers small; order Dockerfile lines from least to most frequently changed when possible.

---

## Part 5 — Build, tag, and list images

**Time estimate:** ~10 minutes

### Step 10 — Build the image

```bash
docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" .
```

**Expected result:** `Successfully tagged cloudmart-storefront:1.0` (or your tag).

---

### Step 11 — Run the CloudMart image locally

```bash
docker run --rm -d -p 8080:80 --name "${CONTAINER}" "${IMAGE_NAME}:${IMAGE_TAG}"
curl -s http://localhost:8080 | head -5
```

Open Cloud Shell **Web preview** on port **8080** if available, or use `curl` output.

---

### Step 12 — Add a second tag

```bash
docker tag "${IMAGE_NAME}:${IMAGE_TAG}" "${IMAGE_NAME}:latest"
docker images | grep cloudmart
```

**Why tags:** `1.0` is immutable release identity; `latest` is a moving pointer (use carefully in production).

---

## Part 6 — Push images to Artifact Registry

**Time estimate:** ~15 minutes

### Step 13 — Create a Docker repository

```bash
gcloud artifacts repositories create "${REPO}" \
  --repository-format=docker \
  --location="${REGION}" \
  --description="CloudMart lab container images"
```

Skip if the repository already exists (instructor-provided).

---

### Step 14 — Configure Docker authentication

```bash
gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet
```

---

### Step 15 — Tag for Artifact Registry and push

```bash
REGISTRY="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}"
docker tag "${IMAGE_NAME}:${IMAGE_TAG}" "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

docker push "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
```

List images in the registry:

```bash
gcloud artifacts docker images list "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}"
```

---

## Part 7 — Pull, run, logs, and exec

**Time estimate:** ~10 minutes

### Step 16 — Remove local container and image (simulate clean host)

```bash
docker stop "${CONTAINER}" 2>/dev/null || true
docker rmi "${IMAGE_NAME}:${IMAGE_TAG}" "${IMAGE_NAME}:latest" 2>/dev/null || true
```

---

### Step 17 — Pull from registry and run

```bash
docker pull "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

docker run --rm -d -p 8080:80 --name "${CONTAINER}" \
  "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"

curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

---

### Step 18 — Logs and exec

```bash
docker logs "${CONTAINER}" --tail 20
docker exec -it "${CONTAINER}" cat /usr/share/nginx/html/index.html | head -3
```

Type `exit` if your shell opened interactively.

---

## Part 8 — Clean up lab resources

**Time estimate:** ~10 minutes

### Step 19 — Stop and remove containers

```bash
docker stop "${CONTAINER}" 2>/dev/null || true
docker ps -a
```

Remove any leftover lab containers:

```bash
docker container prune -f
```

---

### Step 20 — Remove local images

```bash
docker rmi "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}" 2>/dev/null || true
docker image prune -f
```

---

### Step 21 — Delete Artifact Registry repository (optional)

Only if this repo is lab-only and empty enough to delete:

```bash
gcloud artifacts repositories delete "${REPO}" \
  --location="${REGION}" --quiet
```

If deletion is blocked by policy, remove images via Console or ask your instructor.

---

### Step 22 — Remove project files

```bash
rm -rf ~/cloudmart-docker
```

---

## Validation checklist

- [ ] Ran `hello-world` and a sample `nginx` container
- [ ] Wrote Dockerfile with `FROM`, `COPY`, `EXPOSE`, `CMD`
- [ ] Built `cloudmart-storefront:1.0` and tested on port 8080
- [ ] Created Artifact Registry repo `cloudmart-docker`
- [ ] Pushed and pulled the image from the registry
- [ ] Used `docker logs` and `docker exec`
- [ ] Stopped containers and cleaned up images/repo

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | `docker version` |
| 2 | `docker images` showing `cloudmart-storefront` |
| 3 | `curl` or browser showing CloudMart HTML |
| 4 | `docker push` success / Artifact Registry image list |
| 5 | `docker logs` snippet |
| 6 | Proof of cleanup (no lab container running) |

---

## Knowledge check

1. What is the difference between an image and a container?  
   - A) Image is a template; container is a running instance  
   - B) They are the same  
   - C) Container is stored in Cloud SQL  

2. Which Dockerfile instruction copies files into the image?  
   - A) `COPY`  
   - B) `DELETE`  
   - C) `ROUTE`  

3. What does `-p 8080:80` mean on `docker run`?  
   - A) Map host port 8080 to container port 80  
   - B) Set CPU to 8080  
   - C) Create 80 VPC subnets  

4. Where should teams store images for GCP deployments?  
   - A) Artifact Registry  
   - B) Bigtable only  
   - C) Firewall rules  

5. Which command shows container stdout/stderr?  
   - A) `docker logs`  
   - B) `docker build`  
   - C) `bq query`  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — Image vs running container.  
2. **A** — `COPY`.  
3. **A** — Port mapping.  
4. **A** — Artifact Registry.  
5. **A** — `docker logs`.

</details>

---

## Optional challenge

1. Add a **`.dockerignore`** file and rebuild — note faster context upload.  
2. Build a **multi-stage** image (build in one stage, copy artifacts to `nginx:alpine`).  
3. Document three differences between this lab and **serverless container deploy** (concept only).

---

## Official references

- [Docker documentation overview](https://docs.docker.com/get-started/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Artifact Registry overview](https://cloud.google.com/artifact-registry/docs/overview)
- [Artifact Registry Docker images](https://cloud.google.com/artifact-registry/docs/docker)
- [Configure authentication for Docker](https://cloud.google.com/artifact-registry/docs/docker/authentication)
- Presentation 09 — *Docker and Containerization Fundamentals* (course materials)

---

## Lab outcome

You built a **CloudMart storefront** Docker image, ran it locally, published it to **Artifact Registry**, pulled it back, inspected **logs**, and **cleaned up** containers and images.

You can take other training labs in any order; none are required before or after this one.