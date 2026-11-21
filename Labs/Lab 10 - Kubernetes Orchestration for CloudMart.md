# Lab 10 — Kubernetes: Orchestration for CloudMart

| | |
| --- | --- |
| **Training module** | Module 10 — Kubernetes and container orchestration |
| **Companion presentation** | Presentation 10 — Introduction to Kubernetes and Container Orchestration |
| **Level** | Beginner to intermediate |
| **Estimated time** | 90–120 minutes |
| **Primary tools** | `kubectl`, GKE (Autopilot) |
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
7. [Part 1 — Kubernetes fundamentals](#part-1--kubernetes-fundamentals)
8. [Part 2 — Create a GKE cluster and configure kubectl](#part-2--create-a-gke-cluster-and-configure-kubectl)
9. [Part 3 — Namespaces and workload isolation](#part-3--namespaces-and-workload-isolation)
10. [Part 4 — Deploy CloudMart with a Deployment](#part-4--deploy-cloudmart-with-a-deployment)
11. [Part 5 — Expose the app with a Service](#part-5--expose-the-app-with-a-service)
12. [Part 6 — Scale, observe, and troubleshoot](#part-6--scale-observe-and-troubleshoot)
13. [Part 7 — ConfigMaps and rolling updates](#part-7--configmaps-and-rolling-updates)
14. [Part 8 — Clean up lab resources](#part-8--clean-up-lab-resources)
15. [Validation checklist](#validation-checklist)
16. [Evidence to submit](#evidence-to-submit)
17. [Knowledge check](#knowledge-check)
18. [Optional challenge](#optional-challenge)
19. [Official references](#official-references)

---

## Lab description

In this lab, you run **CloudMart** on **Kubernetes** using **Google Kubernetes Engine (GKE)**. You create a small **Autopilot** cluster, use **`kubectl`** to deploy a **Deployment**, expose it with a **Service**, **scale** replicas, apply a **ConfigMap**, and perform a simple **rolling update**. The lab teaches **orchestration concepts** from **Presentation 10**—not advanced GKE operations (see Presentation 11).

**Standalone lab:** You do not need any other lab in this folder. The cluster, namespace, and manifests are created and removed here. You may use any public container image (for example `nginx:alpine`); you do not need images from Lab 09.

---

## Scenario and use case

CloudMart runs multiple **storefront** containers across zones. Operations needs:

| Need | Kubernetes object |
| --- | --- |
| Run N identical web pods | **Deployment** |
| Stable traffic endpoint | **Service** |
| Team separation | **Namespace** |
| Non-secret configuration | **ConfigMap** |
| Replace app without downtime | **Rolling update** on Deployment |

**Your role:** Build a `cloudmart` namespace, run **two** web replicas, reach the app through a **LoadBalancer** IP, scale to **three** pods, update configuration, then **delete the cluster**.

---

## Learning objectives

When you finish this lab, you should be able to:

- Describe the control plane, nodes, **Pods**, **Deployments**, and **Services**
- Create a **GKE Autopilot** cluster and connect with `kubectl`
- Create and use a **namespace**
- Apply YAML manifests with `kubectl apply`
- Expose a Deployment with a **LoadBalancer** Service
- Scale with `kubectl scale` and inspect with `get`, `describe`, `logs`
- Mount configuration from a **ConfigMap**
- Delete the cluster to stop charges

---

## What you will build

| Item | Value |
| --- | --- |
| Cluster | `cloudmart-k8s-lab` (Autopilot) |
| Namespace | `cloudmart` |
| Deployment | `cloudmart-storefront` |
| Image | `nginx:alpine` (lab default) |
| Service | `cloudmart-storefront-lb` (type `LoadBalancer`) |
| Initial replicas | 2 |

**Cost note:** GKE Autopilot and LoadBalancer IPs incur charges. **Part 8 is required** immediately after the lab.

---

## Prerequisites

Before you start, confirm:

- [ ] Training project with **billing enabled**
- [ ] Roles to create GKE clusters (e.g. **Kubernetes Engine Admin**)
- [ ] **Cloud Shell** (includes `kubectl` and `gcloud`)
- [ ] Presentation **10** reviewed (recommended)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="asia-south1"
export CLUSTER="cloudmart-k8s-lab"
export NAMESPACE="cloudmart"

gcloud config set project "$PROJECT_ID"
```

**Instructor option:** Learners may use a **shared cluster** and skip Part 2 cluster creation—apply manifests only. Still delete your namespace in Part 8.

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1 | Concepts |
| 2 | GKE + `kubectl` |
| 3–5 | Namespace, Deployment, Service |
| 6–7 | Scale, logs, ConfigMap, update |
| 8 | Delete cluster |

Cluster creation can take **10–15 minutes**.

---

## Part 1 — Kubernetes fundamentals

**Time estimate:** ~10 minutes

### Step 1 — Core components

| Component | Role |
| --- | --- |
| **Control plane** | Schedules work, maintains desired state |
| **Node** | Worker machine running **kubelet** and container runtime |
| **Pod** | Smallest deployable unit; one or more containers |
| **Deployment** | Manages ReplicaSets and rolling updates |
| **Service** | Stable network name and load balancing to Pods |

```text
User --> Service (ClusterIP / LoadBalancer) --> Pod(s) in Deployment
```

---

### Step 2 — Declarative vs imperative

This lab uses **declarative** YAML and `kubectl apply` so manifests are reproducible. Imperative commands like `kubectl run` are useful for quick tests but not the main pattern here.

---

## Part 2 — Create a GKE cluster and configure kubectl

**Time estimate:** ~20 minutes (mostly wait time)

### Step 3 — Enable APIs

```bash
gcloud services enable container.googleapis.com
```

---

### Step 4 — Create an Autopilot cluster

**Console:** **Kubernetes Engine → Create → Autopilot** → name `cloudmart-k8s-lab`, location **region** `asia-south1`.

**CLI:**

```bash
gcloud container clusters create-auto "$CLUSTER" \
  --region="$REGION"
```

Wait until status is **RUNNING**:

```bash
gcloud container clusters list --filter="name:$CLUSTER"
```

---

### Step 5 — Configure `kubectl`

```bash
gcloud container clusters get-credentials "$CLUSTER" \
  --region="$REGION"

kubectl cluster-info
kubectl get nodes
```

**Expected result:** Nodes listed (Autopilot manages node pools for you).

---

## Part 3 — Namespaces and workload isolation

**Time estimate:** ~5 minutes

### Step 6 — Create the `cloudmart` namespace

```bash
mkdir -p ~/cloudmart-k8s && cd ~/cloudmart-k8s

kubectl create namespace "$NAMESPACE"
kubectl config set-context --current --namespace="$NAMESPACE"
kubectl get namespaces
```

**Why namespaces:** Separate dev/test resources and RBAC boundaries.

---

## Part 4 — Deploy CloudMart with a Deployment

**Time estimate:** ~15 minutes

### Step 7 — Write the Deployment manifest

```bash
cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-storefront
  namespace: cloudmart
  labels:
    app: cloudmart-storefront
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudmart-storefront
  template:
    metadata:
      labels:
        app: cloudmart-storefront
    spec:
      containers:
        - name: web
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              memory: "256Mi"
EOF
```

---

### Step 8 — Apply and verify Pods

```bash
kubectl apply -f deployment.yaml
kubectl get deployments -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
```

**Expected result:** `2/2` replicas ready; each Pod status **Running**.

```bash
kubectl describe deployment cloudmart-storefront -n "$NAMESPACE" | head -40
```

---

## Part 5 — Expose the app with a Service

**Time estimate:** ~15 minutes

### Step 9 — Create a LoadBalancer Service

```bash
cat > service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: cloudmart-storefront-lb
  namespace: cloudmart
  labels:
    app: cloudmart-storefront
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-storefront
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF

kubectl apply -f service.yaml
kubectl get svc -n "$NAMESPACE" -w
```

Press `Ctrl+C` when **EXTERNAL-IP** shows an IP (not `<pending>`).

---

### Step 10 — Test the storefront

```bash
EXTERNAL_IP=$(kubectl get svc cloudmart-storefront-lb -n "$NAMESPACE" \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://$EXTERNAL_IP"
curl -s -o /dev/null -w "%{http_code}\n" "http://$EXTERNAL_IP"
```

**Expected result:** HTTP `200` (default nginx welcome page).

---

## Part 6 — Scale, observe, and troubleshoot

**Time estimate:** ~15 minutes

### Step 11 — Scale the Deployment

```bash
kubectl scale deployment cloudmart-storefront -n "$NAMESPACE" --replicas=3
kubectl get pods -n "$NAMESPACE"
```

**Expected result:** Three Running pods.

---

### Step 12 — Logs and exec

```bash
POD=$(kubectl get pods -n "$NAMESPACE" -l app=cloudmart-storefront \
  -o jsonpath='{.items[0].metadata.name}')

kubectl logs "$POD" -n "$NAMESPACE" --tail=20
kubectl exec -it "$POD" -n "$NAMESPACE" -- nginx -v
```

---

### Step 13 — Troubleshooting practice

| Symptom | Command | What to check |
| --- | --- | --- |
| Pods not Ready | `kubectl describe pod POD` | Events, image pull, resources |
| No external IP | `kubectl describe svc cloudmart-storefront-lb` | LoadBalancer provisioning |
| 502 / no response | `kubectl get endpoints` | Endpoints match running Pods |

Delete one pod and watch recovery:

```bash
kubectl delete pod "$POD" -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -w
```

The Deployment creates a replacement pod.

---

## Part 7 — ConfigMaps and rolling updates

**Time estimate:** ~15 minutes

### Step 14 — Create a ConfigMap

```bash
kubectl create configmap cloudmart-config -n "$NAMESPACE" \
  --from-literal=STORE_NAME="CloudMart Kubernetes Lab"
```

Verify:

```bash
kubectl get configmap cloudmart-config -n "$NAMESPACE" -o yaml
```

---

### Step 15 — Patch Deployment to use the ConfigMap as env vars

```bash
kubectl patch deployment cloudmart-storefront -n "$NAMESPACE" --patch '
spec:
  template:
    spec:
      containers:
        - name: web
          image: nginx:alpine
          env:
            - name: STORE_NAME
              valueFrom:
                configMapKeyRef:
                  name: cloudmart-config
                  key: STORE_NAME
'

kubectl rollout status deployment/cloudmart-storefront -n "$NAMESPACE"
```

---

### Step 16 — Change ConfigMap and roll restart (simple pattern)

```bash
kubectl create configmap cloudmart-config -n "$NAMESPACE" \
  --from-literal=STORE_NAME="CloudMart v2" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/cloudmart-storefront -n "$NAMESPACE"
kubectl rollout status deployment/cloudmart-storefront -n "$NAMESPACE"
```

Confirm env inside a new pod:

```bash
POD=$(kubectl get pods -n "$NAMESPACE" -l app=cloudmart-storefront \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -n "$NAMESPACE" -- printenv STORE_NAME
```

**Optional:** Serve custom HTML via ConfigMap volume in the **optional challenge**—not required for pass criteria.

---

## Part 8 — Clean up lab resources

**Time estimate:** ~15 minutes

### Step 17 — Delete namespace workloads

```bash
kubectl delete namespace "$NAMESPACE" --wait
```

---

### Step 18 — Delete the GKE cluster

```bash
gcloud container clusters delete "$CLUSTER" \
  --region="$REGION" --quiet
```

Verify:

```bash
gcloud container clusters list --filter="name:$CLUSTER"
```

---

### Step 19 — Remove local manifests

```bash
rm -rf ~/cloudmart-k8s
```

---

## Validation checklist

- [ ] Created GKE Autopilot cluster `cloudmart-k8s-lab`
- [ ] Connected with `kubectl get-credentials`
- [ ] Created namespace `cloudmart`
- [ ] Applied Deployment with 2+ replicas
- [ ] Created LoadBalancer Service and reached HTTP 200
- [ ] Scaled Deployment to 3 replicas
- [ ] Used `kubectl logs` and `describe`
- [ ] Created ConfigMap and rolled out updated Deployment
- [ ] Deleted namespace and cluster

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | `kubectl get nodes` |
| 2 | `kubectl get pods -n cloudmart` (3 replicas) |
| 3 | `kubectl get svc` with EXTERNAL-IP |
| 4 | `curl` HTTP 200 to LoadBalancer |
| 5 | `kubectl describe deployment` events |
| 6 | `printenv STORE_NAME` in pod |
| 7 | Empty cluster list after delete |

---

## Knowledge check

1. What is the smallest schedulable unit in Kubernetes?  
   - A) Pod  
   - B) VM  
   - C) Firewall rule  

2. Which object provides a stable way to reach a set of Pods?  
   - A) Service  
   - B) ConfigMap only  
   - C) PersistentVolume  

3. What does a Deployment manage?  
   - A) Desired Pod replicas and rolling updates  
   - B) Only VPC routes  
   - C) BigQuery datasets  

4. Which command applies manifest changes idempotently?  
   - A) `kubectl apply`  
   - B) `docker build`  
   - C) `bq load`  

5. How do you stop GKE charges after this lab?  
   - A) Delete the cluster  
   - B) Close Cloud Shell only  
   - C) Delete kubectl config file only  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — Pod.  
2. **A** — Service.  
3. **A** — Deployment / ReplicaSet behavior.  
4. **A** — `kubectl apply`.  
5. **A** — Delete the cluster.

</details>

---

## Optional challenge

1. Add a **ConfigMap** volume that serves custom `index.html` for CloudMart (mount under `/usr/share/nginx/html`).  
2. Create a **readinessProbe** HTTP GET on `/`.  
3. Compare **ClusterIP** vs **LoadBalancer** Service types in one paragraph.

---

## Official references

- [Kubernetes concepts](https://kubernetes.io/docs/concepts/)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [GKE Autopilot overview](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)
- [Create an Autopilot cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-an-autopilot-cluster)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- Presentation 10 — *Introduction to Kubernetes and Container Orchestration* (course materials)

---

## Lab outcome

You created a **GKE Autopilot** cluster, deployed and **scaled** CloudMart workloads, exposed them with a **LoadBalancer Service**, updated configuration with **ConfigMaps**, and **deleted** the cluster when finished.

You can take other training labs in any order; none are required before or after this one.