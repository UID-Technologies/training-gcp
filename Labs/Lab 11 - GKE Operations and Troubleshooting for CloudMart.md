# Lab 11 — GKE: Operations and Troubleshooting for CloudMart

| | |
| --- | --- |
| **Training module** | Module 11 — Google Kubernetes Engine |
| **Companion presentation** | Presentation 11 — GKE Architecture, Deployment, Operations, and Troubleshooting |
| **Level** | Intermediate |
| **Estimated time** | 90–120 minutes |
| **Primary tools** | GKE Standard, `kubectl`, Cloud Logging |
| **Lab prefix** | `cloudmart` |
| **Zone** | `asia-south1-a` |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [What you will build](#what-you-will-build)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — GKE architecture](#part-1--gke-architecture)
8. [Part 2 — Create a Standard GKE cluster](#part-2--create-a-standard-gke-cluster)
9. [Part 3 — Deploy CloudMart with health probes](#part-3--deploy-cloudmart-with-health-probes)
10. [Part 4 — Expose and verify the application](#part-4--expose-and-verify-the-application)
11. [Part 5 — Horizontal Pod Autoscaler (HPA)](#part-5--horizontal-pod-autoscaler-hpa)
12. [Part 6 — GKE operations (rollouts and cluster maintenance)](#part-6--gke-operations-rollouts-and-cluster-maintenance)
13. [Part 7 — Troubleshooting exercises](#part-7--troubleshooting-exercises)
14. [Part 8 — Logging and the GKE Console](#part-8--logging-and-the-gke-console)
15. [Part 9 — Clean up lab resources](#part-9--clean-up-lab-resources)
16. [Validation checklist](#validation-checklist)
17. [Evidence to submit](#evidence-to-submit)
18. [Knowledge check](#knowledge-check)
19. [Optional challenge](#optional-challenge)
20. [Official references](#official-references)

---

## Lab description

In this lab, you operate a **Google Kubernetes Engine (GKE) Standard** cluster for **CloudMart production-style** workloads. You deploy an application with **liveness and readiness probes**, expose it with a **LoadBalancer**, configure a **Horizontal Pod Autoscaler (HPA)**, practice **rollout** commands, **troubleshoot** a failed image update, and review **logs** in Cloud Logging. This aligns with **Presentation 11** (architecture, deployment, operations, troubleshooting).

**Standalone lab:** You do not need any other lab in this folder. Cluster name **`cloudmart-gke-ops-lab`** is unique to this guide. Kubernetes basics are explained in the steps; you do not need to complete Lab 10 first.

---

## Scenario and use case

CloudMart’s SRE team runs checkout and storefront services on **GKE**. They need:

| Operations task | Lab practice |
| --- | --- |
| Healthy deployments | **Liveness** and **readiness** probes |
| Traffic during releases | **Rolling update** and **rollout undo** |
| Handle load spikes | **HPA** (concept + resource) |
| Fix bad releases | **CrashLoopBackOff** diagnosis |
| Audit issues | **Events**, **logs**, GKE Console |

**Your role:** Stand up a zonal **Standard** cluster, run a resilient Deployment, break and fix a release, then **delete the cluster**.

---

## Learning objectives

When you finish this lab, you should be able to:

- Compare **GKE Autopilot** and **GKE Standard** (when this lab uses Standard)
- Describe **control plane**, **node pool**, and worker **nodes**
- Deploy workloads with **resource requests** and **health probes**
- Create and inspect an **HPA**
- Use `kubectl rollout history`, `kubectl rollout undo`
- Troubleshoot **ImagePullBackOff** / **CrashLoopBackOff** with `describe` and events
- Find workload messages in **Cloud Logging**
- **Delete** the GKE cluster to stop charges

---

## What you will build

| Item | Value |
| --- | --- |
| Cluster | `cloudmart-gke-ops-lab` (Standard, zonal) |
| Node pool | Default pool, 2 × `e2-small` nodes |
| Namespace | `cloudmart-ops` |
| Deployment | `cloudmart-api` |
| Service | `cloudmart-api-lb` (LoadBalancer) |
| HPA | `cloudmart-api-hpa` (2–5 replicas) |

**Cost note:** Standard GKE charges for **nodes** and **cluster management**. **Part 9 is required** when you finish.

---

## Prerequisites

- [ ] Training project with **billing enabled**
- [ ] **Kubernetes Engine Admin** (or equivalent) on the project
- [ ] **Cloud Shell** with `kubectl` and `gcloud`
- [ ] Presentation **11** reviewed (recommended)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export ZONE="asia-south1-a"
export CLUSTER="cloudmart-gke-ops-lab"
export NAMESPACE="cloudmart-ops"

gcloud config set project "$PROJECT_ID"
```

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1–2 | GKE architecture and cluster |
| 3–4 | Probes and Service |
| 5 | HPA |
| 6–7 | Rollouts and troubleshooting |
| 8 | Logging |
| 9 | Delete cluster |

Zonal cluster creation often takes **5–10 minutes**.

---

## Part 1 — GKE architecture

**Time estimate:** ~10 minutes

### Step 1 — Autopilot vs Standard

| Mode | You manage | Best for |
| --- | --- | --- |
| **Autopilot** | Workloads (Google sizes nodes) | Simpler ops, hands-off nodes |
| **Standard** | **Node pools**, machine types, scaling | Custom software on nodes, GPUs, legacy needs |

**Control plane:** Google-managed (API server, scheduler, etc.). **Nodes:** run your Pods.

---

### Step 2 — Regional vs zonal

| Topology | Tradeoff |
| --- | --- |
| **Zonal** (this lab) | One zone; fewer nodes; lower cost for training |
| **Regional** | Control plane replicas across zones; higher resilience |

---

### Step 3 — Open GKE in the Console

1. **Kubernetes Engine → Clusters**.
2. Note **mode**, **location**, and **node pools** on any sample clusters (read-only).
3. Open **Features** documentation links if your instructor provides them.

---

## Part 2 — Create a Standard GKE cluster

**Time estimate:** ~15 minutes

### Step 4 — Enable APIs

```bash
gcloud services enable container.googleapis.com \
  compute.googleapis.com \
  logging.googleapis.com
```

---

### Step 5 — Create the cluster

**Console:** **Create → Standard** → name `cloudmart-gke-ops-lab`, **Zonal** `asia-south1-a`, **Number of nodes:** 2, **Machine type:** `e2-small`, **Release channel:** Regular.

**CLI:**

```bash
gcloud container clusters create "$CLUSTER" \
  --zone="$ZONE" \
  --num-nodes=2 \
  --machine-type=e2-small \
  --release-channel=regular \
  --enable-ip-alias
```

Wait until `RUNNING`:

```bash
gcloud container clusters list --filter="name:$CLUSTER"
```

---

### Step 6 — Connect and inspect nodes

```bash
gcloud container clusters get-credentials "$CLUSTER" --zone="$ZONE"

kubectl get nodes -o wide
kubectl top nodes 2>/dev/null || echo "Metrics may take a few minutes to appear."
```

```bash
gcloud container node-pools list --cluster="$CLUSTER" --zone="$ZONE"
```

**Expected result:** Two nodes in `Ready` state.

---

### Step 7 — Create namespace

```bash
mkdir -p ~/cloudmart-gke-ops && cd ~/cloudmart-gke-ops

kubectl create namespace "$NAMESPACE"
```

---

## Part 3 — Deploy CloudMart with health probes

**Time estimate:** ~15 minutes

### Step 8 — Deployment manifest with probes and resources

Probes keep unhealthy Pods out of load balancing; **requests** are required for meaningful **HPA**.

```bash
cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-api
  namespace: cloudmart-ops
  labels:
    app: cloudmart-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudmart-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: cloudmart-api
    spec:
      containers:
        - name: web
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
EOF

kubectl apply -f deployment.yaml
kubectl get pods -n "$NAMESPACE" -w
```

Stop the watch with `Ctrl+C` when all pods are **Running** and **Ready** (`1/1`).

---

### Step 9 — Verify probe behavior

```bash
kubectl describe deployment cloudmart-api -n "$NAMESPACE" | grep -A6 "Liveness\|Readiness"
```

---

## Part 4 — Expose and verify the application

**Time estimate:** ~10 minutes

### Step 10 — LoadBalancer Service

```bash
cat > service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: cloudmart-api-lb
  namespace: cloudmart-ops
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-api
  ports:
    - port: 80
      targetPort: 80
EOF

kubectl apply -f service.yaml
kubectl get svc -n "$NAMESPACE"
```

Wait for **EXTERNAL-IP**, then test:

```bash
EXTERNAL_IP=$(kubectl get svc cloudmart-api-lb -n "$NAMESPACE" \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s -o /dev/null -w "%{http_code}\n" "http://$EXTERNAL_IP"
```

---

## Part 5 — Horizontal Pod Autoscaler (HPA)

**Time estimate:** ~15 minutes

### Step 11 — Create HPA

```bash
kubectl autoscale deployment cloudmart-api -n "$NAMESPACE" \
  --cpu-percent=50 \
  --min=2 \
  --max=5 \
  --name=cloudmart-api-hpa
```

Inspect:

```bash
kubectl get hpa -n "$NAMESPACE"
kubectl describe hpa cloudmart-api-hpa -n "$NAMESPACE"
```

**Note:** Without sustained CPU load, replicas may stay at **2**. That is expected in this lab.

---

### Step 12 — Optional quick load (instructor only)

If allowed, generate CPU load inside one pod:

```bash
POD=$(kubectl get pods -n "$NAMESPACE" -l app=cloudmart-api \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n "$NAMESPACE" "$POD" -- sh -c \
  "wget -O /dev/null http://localhost/ &" 
```

Re-check `kubectl get hpa -n "$NAMESPACE"` after 2–3 minutes. Stop artificial load afterward.

---

## Part 6 — GKE operations (rollouts and cluster maintenance)

**Time estimate:** ~15 minutes

### Step 13 — Controlled image update

```bash
kubectl set image deployment/cloudmart-api -n "$NAMESPACE" \
  web=nginx:1.27-alpine

kubectl rollout status deployment/cloudmart-api -n "$NAMESPACE"
kubectl rollout history deployment/cloudmart-api -n "$NAMESPACE"
```

**Expected result:** Rollout completes with zero `maxUnavailable` (per strategy).

---

### Step 14 — Cluster maintenance concepts (read-only)

List upgrade versions (do **not** upgrade during class unless instructed):

```bash
gcloud container get-server-config --zone="$ZONE" \
  --format="yaml(validMasterVersions,validNodeVersions)" | head -30
```

Discuss: **surge upgrades** for nodes, **release channels**, and maintenance windows in the Console.

---

## Part 7 — Troubleshooting exercises

**Time estimate:** ~20 minutes

### Step 15 — Break the deployment (bad image tag)

```bash
kubectl set image deployment/cloudmart-api -n "$NAMESPACE" \
  web=nginx:does-not-exist

kubectl get pods -n "$NAMESPACE" -w
```

**Expected result:** New pods show **ErrImagePull** or **ImagePullBackOff**.

---

### Step 16 — Diagnose

```bash
POD=$(kubectl get pods -n "$NAMESPACE" -l app=cloudmart-api \
  --field-selector=status.phase!=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

kubectl describe pod "$POD" -n "$NAMESPACE" | tail -25
kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | tail -10
```

Identify the **Failed to pull image** message.

---

### Step 17 — Recover with rollout undo

```bash
kubectl rollout undo deployment/cloudmart-api -n "$NAMESPACE"
kubectl rollout status deployment/cloudmart-api -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE"
curl -s -o /dev/null -w "%{http_code}\n" "http://$EXTERNAL_IP"
```

**Expected result:** Pods healthy; HTTP `200` again.

---

### Step 18 — Troubleshooting reference

| Symptom | Likely cause | Commands |
| --- | --- | --- |
| `ImagePullBackOff` | Wrong tag or registry auth | `describe pod`, fix image |
| `CrashLoopBackOff` | App exits on start | `logs`, probes, command |
| `Pending` | Insufficient CPU/memory | `describe pod`, node capacity |
| Service no endpoints | Selector mismatch | `get endpoints`, labels |

---

## Part 8 — Logging and the GKE Console

**Time estimate:** ~10 minutes

### Step 19 — GKE Console workload view

1. **Kubernetes Engine → Workloads**.
2. Select cluster **`cloudmart-gke-ops-lab`**.
3. Open **cloudmart-api** → **Logs** and **Events**.

---

### Step 20 — Query Cloud Logging

```bash
gcloud logging read \
  'resource.type="k8s_container"
   resource.labels.namespace_name="cloudmart-ops"
   resource.labels.cluster_name="cloudmart-gke-ops-lab"' \
  --limit=10 \
  --format="table(timestamp,textPayload)"
```

Adjust the filter if your environment uses structured JSON logs instead of `textPayload`.

---

## Part 9 — Clean up lab resources

**Time estimate:** ~15 minutes

### Step 21 — Delete namespace

```bash
kubectl delete namespace "$NAMESPACE" --wait
```

---

### Step 22 — Delete cluster

```bash
gcloud container clusters delete "$CLUSTER" \
  --zone="$ZONE" --quiet
```

Verify:

```bash
gcloud container clusters list --filter="name:$CLUSTER"
```

---

### Step 23 — Remove local files

```bash
rm -rf ~/cloudmart-gke-ops
```

---

## Validation checklist

- [ ] Created Standard GKE cluster with visible nodes
- [ ] Deployed `cloudmart-api` with liveness and readiness probes
- [ ] LoadBalancer returns HTTP 200
- [ ] Created and described HPA
- [ ] Performed rollout update and `rollout history`
- [ ] Reproduced image pull failure and used `rollout undo`
- [ ] Viewed logs in Console or `gcloud logging read`
- [ ] Deleted cluster in Part 9

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | GKE cluster details (Standard, node pool) |
| 2 | `kubectl get nodes` |
| 3 | Deployment showing probe configuration |
| 4 | `kubectl get hpa` |
| 5 | `describe pod` showing ImagePullBackOff |
| 6 | Successful `rollout undo` and HTTP 200 |
| 7 | Sample log entries |
| 8 | No cluster after cleanup |

---

## Knowledge check

1. Who manages worker nodes in **GKE Standard**?  
   - A) You configure node pools; Google manages the control plane  
   - B) You manage the control plane only  
   - C) Neither; only Cloud Run manages nodes  

2. What does a **readiness** probe control?  
   - A) Whether the Pod receives Service traffic  
   - B) VPC firewall rules  
   - C) BigQuery slot usage  

3. Which command reverts a failed Deployment rollout?  
   - A) `kubectl rollout undo`  
   - B) `docker stop`  
   - C) `gcloud sql import`  

4. HPA scales based on metrics such as:  
   - A) CPU utilization (when configured)  
   - B) Number of GCS buckets  
   - C) IAM role count  

5. How do you stop GKE node charges after this lab?  
   - A) Delete the cluster  
   - B) Delete only the Deployment  
   - C) Log out of Console  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — Standard mode node pools.  
2. **A** — Readiness and Service endpoints.  
3. **A** — `rollout undo`.  
4. **A** — CPU (in this lab’s HPA).  
5. **A** — Delete the cluster.

</details>

---

## Optional challenge

1. Add a **PodDisruptionBudget** for `cloudmart-api`.  
2. Create an **Ingress** with a static IP (if instructor provides DNS).  
3. Summarize **Workload Identity** in two sentences for CloudMart accessing Cloud Storage.

---

## Official references

- [GKE overview](https://cloud.google.com/kubernetes-engine/docs/concepts/kubernetes-engine-overview)
- [GKE Autopilot vs Standard](https://cloud.google.com/kubernetes-engine/docs/resources/autopilot-standard-feature-comparison)
- [Create a cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)
- [Configure liveness and readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [kubectl rollout](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
- [View GKE logs](https://cloud.google.com/kubernetes-engine/docs/how-to/view-logs)
- [Troubleshoot workloads](https://cloud.google.com/kubernetes-engine/docs/troubleshooting)
- Presentation 11 — *GKE Architecture, Deployment, Operations, and Troubleshooting* (course materials)

---

## Lab outcome

You operated a **GKE Standard** cluster for CloudMart, deployed workloads with **health probes**, configured **HPA**, practiced **rollouts**, **troubleshot** a failed release, reviewed **logs**, and **deleted** the cluster.

You can take other training labs in any order; none are required before or after this one.