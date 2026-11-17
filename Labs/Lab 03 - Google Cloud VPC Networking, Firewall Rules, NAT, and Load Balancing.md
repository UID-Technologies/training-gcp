# Lab 03 — Google Cloud VPC Networking, Firewall Rules, NAT, and Load Balancing

| | |
| --- | --- |
| **Training module** | Module 3 — Google Cloud Networking |
| **Level** | Beginner to intermediate |
| **Estimated time** | 120–150 minutes |
| **Primary services** | VPC, Compute Engine, Cloud NAT, Cloud Load Balancing |
| **Lab prefix** | `cloudmart` |
| **Region** | `asia-south1` (Mumbai) |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [Network design](#network-design)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Virtual Private Cloud (VPC) fundamentals](#part-1--virtual-private-cloud-vpc-fundamentals)
8. [Part 2 — Create the CloudMart VPC and subnets](#part-2--create-the-cloudmart-vpc-and-subnets)
9. [Part 3 — Subnets and IP addressing](#part-3--subnets-and-ip-addressing)
10. [Part 4 — VPC firewall rules](#part-4--vpc-firewall-rules)
11. [Part 5 — Routes and network connectivity](#part-5--routes-and-network-connectivity)
12. [Part 6 — External and internal IP addresses](#part-6--external-and-internal-ip-addresses)
13. [Part 7 — Cloud NAT overview and configuration](#part-7--cloud-nat-overview-and-configuration)
14. [Part 8 — Load balancing concepts (HTTP load balancer)](#part-8--load-balancing-concepts-http-load-balancer)
15. [Part 9 — Clean up lab resources](#part-9--clean-up-lab-resources)
16. [Validation checklist](#validation-checklist)
17. [Evidence to submit](#evidence-to-submit)
18. [Knowledge check](#knowledge-check)
19. [Optional challenge](#optional-challenge)
20. [Official references](#official-references)

---

## Lab description

In this hands-on lab, you design and build a **custom Virtual Private Cloud (VPC)** for **CloudMart**. You create **subnets**, **firewall rules**, and **routes**; deploy VMs with **internal** and **external** IP addressing; enable **outbound internet** for private workloads using **Cloud NAT**; and implement a basic **external HTTP load balancer** in front of web servers.

This lab aligns with **Module 3: Google Cloud Networking**.

**Standalone lab:** You do not need any other lab in this folder. This lab creates its own VPC and VMs; it does not use resources from other labs.

---

## Scenario and use case

### CloudMart network segmentation

CloudMart is preparing a **three-tier style** layout for its e-commerce platform:

| Tier | Requirement | Network placement |
| --- | --- | --- |
| **Web** | Accept customer HTTP traffic | Public-facing subnet; load balancer front end |
| **Application** | Call Google APIs and package repos; **no** direct inbound internet | Private subnet; egress via **Cloud NAT** |
| **Administration** | SSH from trusted paths only | Controlled by firewall rules and tags |

**Your role:** As a network administrator, you must:

1. Replace ad hoc `default` VPC usage with a purpose-built **`cloudmart-vpc`**.
2. Carve **regional subnets** with clear CIDR blocks.
3. Apply **least-privilege firewall rules** using **target tags** and **service accounts** where appropriate.
4. Verify **routing** to the internet and between subnets.
5. Demonstrate **private VMs** reaching the internet only through NAT.
6. Distribute customer traffic across multiple web servers with an **HTTP load balancer**.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain VPC networks, regions, subnets, and IP ranges in Google Cloud
- Create a **custom mode** VPC and regional subnets
- Plan **internal IPv4** addressing and optional **secondary IP ranges** (conceptual)
- Create **ingress** firewall rules with priorities, tags, and source filters
- Read the **system-generated routes** and understand default internet access
- Differentiate **internal (RFC 1918)** and **external (public)** IP addresses on VMs
- Configure **Cloud Router** and **Cloud NAT** for private subnet egress
- Describe load balancer types and build a simple **global external HTTP(S) load balancer** backend

---

## Network design

### Architecture diagram (logical)

```text
Internet
    |
    v
[ Global HTTP(S) Load Balancer — static anycast IP ]
    |
    +--> Backend VMs (public subnet, tag: cloudmart-web)
    |
[ Ephemeral external IP — optional admin jump host ]

cloudmart-vpc (custom, regional routing)
|
+-- cloudmart-public-asia-south1   10.10.1.0/24   (asia-south1)
|      cloudmart-web-01, cloudmart-web-02, (optional bastion)
|
+-- cloudmart-private-asia-south1  10.10.2.0/24   (asia-south1)
       cloudmart-app-private-vm (no external IP)

Cloud Router (asia-south1) + Cloud NAT --> outbound internet for private subnet
```

### Resource naming reference

| Resource | Name |
| --- | --- |
| VPC | `cloudmart-vpc` |
| Public subnet | `cloudmart-public-asia-south1` (`10.10.1.0/24`) |
| Private subnet | `cloudmart-private-asia-south1` (`10.10.2.0/24`) |
| Web VMs | `cloudmart-web-01`, `cloudmart-web-02` |
| Private app VM | `cloudmart-app-private-vm` |
| Router | `cloudmart-router` |
| NAT | `cloudmart-nat` |
| Instance group | `cloudmart-web-ig` |
| Health check | `cloudmart-http-hc` |
| Load balancer IP | `cloudmart-lb-ip` (global static) |

**Cost note:** VMs, external IPs, load balancers, and NAT gateways can incur charges. Run **Part 9 — Clean up** when finished.

---

## Prerequisites

Before you start, confirm:

- [ ] Training project with **billing enabled**
- [ ] Permissions to create VPC networks, subnets, firewall rules, routers, NAT, VMs, and load balancers (e.g. Compute Network Admin / Editor on a sandbox project)
- [ ] **Compute Engine API** enabled
- [ ] Basic familiarity with Google Cloud Console and `gcloud` (from course material or prior experience)
- [ ] Cloud Shell or local `gcloud` configured with the correct project

Set your project ID for commands:

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
gcloud config set project "$PROJECT_ID"
gcloud config set compute/region asia-south1
gcloud config set compute/zone asia-south1-a
```

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1–2 | VPC and subnets |
| 3–4 | Addressing and firewalls |
| 5–6 | Routes and IP types on VMs |
| 7 | Cloud NAT |
| 8 | HTTP load balancer |
| 9 | Cleanup |

This lab uses a dedicated VPC (`cloudmart-vpc`). It does not modify the `default` network unless you choose to.

---

## Part 1 — Virtual Private Cloud (VPC) fundamentals

**Time estimate:** ~10 minutes

### Step 1 — Compare default and custom VPCs

1. Open [Google Cloud Console](https://console.cloud.google.com/) → **VPC network → VPC networks**.
2. Locate the **`default`** network (auto-created subnets in each region).
3. Note: Many training projects still have a `default` network; production teams often use **custom VPCs** for control.

| Mode | Behavior |
| --- | --- |
| **Auto mode** | One subnet per region (Google-chosen CIDR). Simple but less control. |
| **Custom mode** | No subnets until you create them. **Used in this lab.** |

**Global VPC, regional subnets:** A VPC spans all regions; each **subnet** is regional and has one **primary IPv4 range** (and optional secondary ranges for GKE aliases).

---

### Step 2 — Key vocabulary

| Term | Meaning |
| --- | --- |
| **VPC network** | Software-defined private network in Google Cloud |
| **Subnet** | Regional IP segment; VMs get **internal IPs** from the subnet range |
| **Firewall rule** | Allow/deny traffic **to/from VMs** in the VPC (not a wall between VPC and internet by itself) |
| **Route** | Next-hop decision for packets (default route to internet gateway) |
| **External IP** | Public address mapped to a VM NIC (ephemeral or static) |
| **Cloud NAT** | Outbound SNAT for VMs **without** public IPs |

---

## Part 2 — Create the CloudMart VPC and subnets

**Time estimate:** ~15 minutes

### Step 3 — Create custom VPC (Console)

1. **VPC network → VPC networks → Create VPC network**.
2. **Name:** `cloudmart-vpc`.
3. **Subnet creation mode:** **Custom**.
4. Do **not** add subnets on this screen yet if the wizard separates steps; otherwise continue to Step 4.
5. **Dynamic routing mode:** **Regional** (default for many designs).
6. **Create**.

### Step 4 — Create subnets (Console)

Create **two** subnets in **`asia-south1`**:

**Public-facing subnet**

| Field | Value |
| --- | --- |
| Name | `cloudmart-public-asia-south1` |
| Region | `asia-south1` |
| IP address range | `10.10.1.0/24` |
| Private Google Access | On (recommended) |
| Flow logs | Off (enable in production) |

**Private application subnet**

| Field | Value |
| --- | --- |
| Name | `cloudmart-private-asia-south1` |
| Region | `asia-south1` |
| IP address range | `10.10.2.0/24` |
| Private Google Access | On |
| Flow logs | Off |

**CLI equivalent**

```bash
gcloud compute networks create cloudmart-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

gcloud compute networks subnets create cloudmart-public-asia-south1 \
  --network=cloudmart-vpc \
  --region=asia-south1 \
  --range=10.10.1.0/24 \
  --enable-private-ip-google-access

gcloud compute networks subnets create cloudmart-private-asia-south1 \
  --network=cloudmart-vpc \
  --region=asia-south1 \
  --range=10.10.2.0/24 \
  --enable-private-ip-google-access
```

**Expected result:** One VPC and two subnets visible under **VPC network → VPC networks → cloudmart-vpc**.

---

## Part 3 — Subnets and IP addressing

**Time estimate:** ~10 minutes

### Step 5 — Understand CIDR planning

| Subnet | CIDR | Usable design intent |
| --- | --- | --- |
| Public | `10.10.1.0/24` | Up to ~254 VM IPs for web tier and load balancer backends |
| Private | `10.10.2.0/24` | Application tier; no public IPs on NICs |

Google reserves some addresses in each subnet; plan headroom for **GKE secondary ranges** in later modules.

---

### Step 6 — Inspect subnet details

Console: **VPC network → VPC networks → cloudmart-vpc → Subnets** → open each subnet.

CLI:

```bash
gcloud compute networks subnets describe cloudmart-public-asia-south1 \
  --region=asia-south1 \
  --format="yaml(name,ipCidrRange,privateIpGoogleAccess,gatewayAddress)"

gcloud compute networks subnets describe cloudmart-private-asia-south1 \
  --region=asia-south1
```

**Private Google Access:** Allows VMs **without** public IPs to reach **certain** Google APIs and services via **internal paths** (complements Cloud NAT for general internet).

---

## Part 4 — VPC firewall rules

**Time estimate:** ~20 minutes

VPC firewall rules are **stateful** at the VM level. This lab uses **target tags** to scope rules to specific VMs.

### Step 7 — Allow internal communication

Allow VM-to-VM traffic inside `10.10.0.0/16` (adjust if you expand addressing).

**Console:** **VPC network → Firewall → Create firewall rule**

| Field | Value |
| --- | --- |
| Name | `cloudmart-allow-internal` |
| Network | `cloudmart-vpc` |
| Priority | `1000` |
| Direction | Ingress |
| Action | Allow |
| Targets | All instances in the network |
| Source IPv4 ranges | `10.10.0.0/16` |
| Protocols | tcp, udp, icmp |

**CLI:**

```bash
gcloud compute firewall-rules create cloudmart-allow-internal \
  --network=cloudmart-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --source-ranges=10.10.0.0/16 \
  --rules=tcp,udp,icmp
```

---

### Step 8 — Allow HTTP to web servers (tagged)

| Field | Value |
| --- | --- |
| Name | `cloudmart-allow-http` |
| Network | `cloudmart-vpc` |
| Targets | Specified target tags |
| Target tags | `cloudmart-web` |
| Source IPv4 ranges | `0.0.0.0/0` |
| Protocols | `tcp:80` |

**CLI:**

```bash
gcloud compute firewall-rules create cloudmart-allow-http \
  --network=cloudmart-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --target-tags=cloudmart-web \
  --source-ranges=0.0.0.0/0 \
  --rules=tcp:80
```

**Production note:** Restrict source ranges when possible; use **Identity-Aware Proxy** or **load balancer-only** exposure for admin paths.

---

### Step 9 — Allow Google load balancer health checks

Health check probes originate from Google-owned ranges.

| Field | Value |
| --- | --- |
| Name | `cloudmart-allow-lb-health` |
| Targets | `cloudmart-web` |
| Source ranges | `130.211.0.0/22`, `35.191.0.0/16` |
| Protocols | `tcp:80` |

**CLI:**

```bash
gcloud compute firewall-rules create cloudmart-allow-lb-health \
  --network=cloudmart-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --target-tags=cloudmart-web \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --rules=tcp:80
```

---

### Step 10 — Allow SSH for lab administration (optional, narrow)

Prefer **IAP TCP forwarding** (`35.235.240.0/20`) in production. For sandbox lab SSH:

| Field | Value |
| --- | --- |
| Name | `cloudmart-allow-ssh-iap` |
| Targets | `cloudmart-admin` |
| Source ranges | `35.235.240.0/20` |
| Protocols | `tcp:22` |

Create only if you tag an admin VM with `cloudmart-admin`. Otherwise use Console SSH with IAP enabled on the project.

```bash
gcloud compute firewall-rules create cloudmart-allow-ssh-iap \
  --network=cloudmart-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --target-tags=cloudmart-admin \
  --source-ranges=35.235.240.0/20 \
  --rules=tcp:22
```

---

### Step 11 — Review implicit deny

There is no explicit “deny all” rule. **Lower priority** rules and the **default deny** for unmatched ingress apply. List rules:

```bash
gcloud compute firewall-rules list --filter="network:cloudmart-vpc" \
  --format="table(name,direction,priority,disabled)"
```

---

## Part 5 — Routes and network connectivity

**Time estimate:** ~15 minutes

### Step 12 — View routes

1. **VPC network → Routes**.
2. Filter by network `cloudmart-vpc`.

You should see **default internet gateway routes** per subnet (`default-internet-gateway`).

CLI:

```bash
gcloud compute routes list --filter="network:cloudmart-vpc"
```

| Route type | Purpose |
| --- | --- |
| **Subnet route** | Reach VMs in each subnet |
| **Default route (0.0.0.0/0)** | Internet egress when VM has external IP or via **NAT** |

---

### Step 13 — Optional: static route (conceptual)

**Do not leave unnecessary custom routes in the project.**

Create a **disabled** or temporary custom route only if your instructor demonstrates on-prem simulation:

```bash
# Example only — delete after discussion
gcloud compute routes create cloudmart-demo-route \
  --network=cloudmart-vpc \
  --destination-range=203.0.113.0/24 \
  --next-hop-gateway=default-internet-gateway \
  --priority=900
```

Delete:

```bash
gcloud compute routes delete cloudmart-demo-route --quiet
```

**Connectivity test (after VMs exist):** From a VM with outbound access, `ping` and `traceroute` to `8.8.8.8` illustrate path; ICMP may be blocked by corporate policy—use `curl` if needed.

---

## Part 6 — External and internal IP addresses

**Time estimate:** ~25 minutes

### Step 14 — Create web servers in the public subnet

Create **two** small VMs for load balancing. Use a **startup script** to install Nginx and return instance name in the page.

**Console highlights**

| Field | `cloudmart-web-01` | `cloudmart-web-02` |
| --- | --- | --- |
| Zone | `asia-south1-a` | `asia-south1-b` |
| Machine type | `e2-micro` | `e2-micro` |
| Network | `cloudmart-vpc` | `cloudmart-vpc` |
| Subnetwork | `cloudmart-public-asia-south1` | `cloudmart-public-asia-south1` |
| External IPv4 | Ephemeral | Ephemeral |
| Network tags | `cloudmart-web` | `cloudmart-web` |

**Startup script (both VMs)**

```bash
#!/bin/bash
apt-get update
apt-get install -y nginx
HOSTNAME=$(hostname)
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html><html><body style="font-family:sans-serif;text-align:center;padding:40px;">
<h1>CloudMart Web</h1>
<p>Server: ${HOSTNAME}</p>
<p>Module 3 — VPC Lab</p>
</body></html>
EOF
systemctl enable --now nginx
```

**CLI — first VM**

```bash
gcloud compute instances create cloudmart-web-01 \
  --zone=asia-south1-a \
  --machine-type=e2-micro \
  --subnet=cloudmart-public-asia-south1 \
  --tags=cloudmart-web \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nginx
HOSTNAME=$(hostname)
echo "<h1>CloudMart Web</h1><p>Server: ${HOSTNAME}</p>" > /var/www/html/index.html
systemctl enable --now nginx'
```

Repeat for `cloudmart-web-02` in `asia-south1-b` (edit script similarly).

---

### Step 15 — Create private application VM (no external IP)

| Field | Value |
| --- | --- |
| Name | `cloudmart-app-private-vm` |
| Zone | `asia-south1-a` |
| Subnet | `cloudmart-private-asia-south1` |
| External IP | **None** (`--no-address`) |

```bash
gcloud compute instances create cloudmart-app-private-vm \
  --zone=asia-south1-a \
  --machine-type=e2-micro \
  --subnet=cloudmart-private-asia-south1 \
  --no-address \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

---

### Step 16 — Compare IP addressing

List VMs:

```bash
gcloud compute instances list \
  --filter="name~'cloudmart'" \
  --format="table(name,zone.basename(),networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP:label=EXTERNAL)"
```

| VM | Internal IP | External IP |
| --- | --- | --- |
| `cloudmart-web-01` | `10.10.1.x` | Ephemeral public (if configured) |
| `cloudmart-web-02` | `10.10.1.y` | Ephemeral public |
| `cloudmart-app-private-vm` | `10.10.2.x` | *(none)* |

**Test public web VM**

```bash
curl -s "http://EXTERNAL_IP_OF_WEB_01" | head
```

**Test private VM inbound from internet**

From your laptop, try to SSH or curl the **internal** IP — it should **fail** without a tunnel/bastion.

**Test private to public subnet**

SSH to `cloudmart-web-01` and ping the private VM internal IP (ICMP allowed by internal rule):

```bash
ping -c 3 PRIVATE_VM_INTERNAL_IP
```

---

## Part 7 — Cloud NAT overview and configuration

**Time estimate:** ~20 minutes

### Step 17 — Concepts

| Without NAT | With Cloud NAT |
| --- | --- |
| Private VM cannot reach `apt` repos on the public internet | Outbound connections SNAT to NAT's address |
| Inbound connections still blocked | Does **not** expose inbound ports |

Cloud NAT requires a **Cloud Router** in the **same region** as the subnets.

---

### Step 18 — Create Cloud Router

```bash
gcloud compute routers create cloudmart-router \
  --network=cloudmart-vpc \
  --region=asia-south1
```

Console: **Network connectivity → Cloud Routers → Create**.

---

### Step 19 — Create Cloud NAT

Apply NAT to **private subnet only** (recommended pattern):

```bash
gcloud compute routers nats create cloudmart-nat \
  --router=cloudmart-router \
  --region=asia-south1 \
  --nat-custom-subnet-ip-ranges=cloudmart-private-asia-south1 \
  --auto-allocate-nat-external-ips \
  --enable-logging
```

Console: Edit router → **Network Address Translation (NAT)** → create NAT gateway with subnet `cloudmart-private-asia-south1`.

---

### Step 20 — Verify outbound internet from private VM

SSH to **`cloudmart-web-01`**, then SSH hop to private VM **using internal IP**, or use **IAP**:

```bash
gcloud compute ssh cloudmart-app-private-vm --zone=asia-south1-a --tunnel-through-iap
```

On the private VM:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://www.google.com
sudo apt-get update
```

**Expected result:** HTTP `200` or successful `apt-get update` after NAT propagates (may take 1–2 minutes).

If IAP SSH fails, ensure firewall rule for `tcp:22` from `35.235.240.0/20` exists and OS Login/IAP is allowed on the project.

---

## Part 8 — Load balancing concepts (HTTP load balancer)

**Time estimate:** ~30–40 minutes

### Step 21 — Load balancing concepts

| Type | Layer | Typical use |
| --- | --- | --- |
| **Global external Application LB** | HTTP/HTTPS (L7) | Web apps, URL maps, CDN integration |
| **Regional external/Internal LB** | TCP/UDP or HTTP | Regional apps, lowest latency in one region |
| **Internal TCP/UDP LB** | L4 | Private service VIPs |
| **Network LB (passthrough)** | L4 | TLS pass-through, extreme performance cases |

This lab builds a **global external Application Load Balancer (HTTP)** with an **unmanaged zonal instance group** spanning two zones—enough to see **health checks**, **backends**, and a **single front-end IP**.

---

### Step 22 — Reserve global static IP

```bash
gcloud compute addresses create cloudmart-lb-ip \
  --ip-version=IPV4 \
  --global
```

Note the address:

```bash
gcloud compute addresses describe cloudmart-lb-ip --global --format="get(address)"
```

---

### Step 23 — Create health check

```bash
gcloud compute health-checks create http cloudmart-http-hc \
  --port=80 \
  --request-path=/ \
  --check-interval=10s \
  --timeout=5s \
  --unhealthy-threshold=3 \
  --healthy-threshold=2
```

---

### Step 24 — Create instance group and add VMs

```bash
gcloud compute instance-groups unmanaged create cloudmart-web-ig \
  --zone=asia-south1-a

gcloud compute instance-groups unmanaged add-instances cloudmart-web-ig \
  --zone=asia-south1-a \
  --instances=cloudmart-web-01

# Second VM is in asia-south1-b — use a second group OR move web-02 to -a for simplicity.
# For two zones, create backends in a MIG later; for this lab, add web-02 to same zone group
# only if both VMs share zone. Recommended: recreate web-02 in asia-south1-a OR use NEG/MIG.
```

**Simplifying note for learners:** If `cloudmart-web-02` is in `asia-south1-b`, either:

- **Option A (recommended):** Delete and recreate `cloudmart-web-02` in `asia-south1-a` and add both to `cloudmart-web-ig`, or  
- **Option B:** Create a **regional** backend with instance groups in each zone (advanced).

**Option A commands:**

```bash
gcloud compute instances delete cloudmart-web-02 --zone=asia-south1-b --quiet

gcloud compute instances create cloudmart-web-02 \
  --zone=asia-south1-a \
  --machine-type=e2-micro \
  --subnet=cloudmart-public-asia-south1 \
  --tags=cloudmart-web \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nginx
HOSTNAME=$(hostname)
echo "<h1>CloudMart Web</h1><p>Server: ${HOSTNAME}</p>" > /var/www/html/index.html
systemctl enable --now nginx'

gcloud compute instance-groups unmanaged add-instances cloudmart-web-ig \
  --zone=asia-south1-a \
  --instances=cloudmart-web-02
```

---

### Step 25 — Backend service, URL map, proxy, forwarding rule

```bash
gcloud compute backend-services create cloudmart-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=cloudmart-http-hc \
  --global

gcloud compute backend-services add-backend cloudmart-backend \
  --instance-group=cloudmart-web-ig \
  --instance-group-zone=asia-south1-a \
  --global

gcloud compute url-maps create cloudmart-url-map \
  --default-service=cloudmart-backend

gcloud compute target-http-proxies create cloudmart-http-proxy \
  --url-map=cloudmart-url-map

gcloud compute forwarding-rules create cloudmart-http-rule \
  --address=cloudmart-lb-ip \
  --global \
  --target-http-proxy=cloudmart-http-proxy \
  --ports=80
```

Wait several minutes for the load balancer to become healthy.

---

### Step 26 — Test load distribution

```bash
LB_IP=$(gcloud compute addresses describe cloudmart-lb-ip --global --format="get(address)")
echo "http://$LB_IP"
curl -s "http://$LB_IP"
```

Refresh the browser multiple times. You may see **`cloudmart-web-01`** or **`cloudmart-web-02`** in the HTML.

Check backend health:

```bash
gcloud compute backend-services get-health cloudmart-backend --global
```

**Concept:** Unhealthy backends are removed from rotation automatically.

---

### Step 27 — Compare direct VM IP vs load balancer IP

| Access path | When to use |
| --- | --- |
| `http://VM_EXTERNAL_IP` | Debugging single instance |
| `http://LOAD_BALANCER_IP` | Customer-facing entry; scales out; health-checked |

For production, terminate **HTTPS** at the load balancer with **Google-managed certificates**.

---

## Part 9 — Clean up lab resources

**Time estimate:** ~15 minutes

Delete in an order that respects dependencies.

### Step 28 — Delete load balancer components

```bash
gcloud compute forwarding-rules delete cloudmart-http-rule --global --quiet

gcloud compute target-http-proxies delete cloudmart-http-proxy --quiet

gcloud compute url-maps delete cloudmart-url-map --quiet

gcloud compute backend-services delete cloudmart-backend --global --quiet

gcloud compute health-checks delete cloudmart-http-hc --quiet

gcloud compute instance-groups unmanaged delete cloudmart-web-ig \
  --zone=asia-south1-a --quiet

gcloud compute addresses delete cloudmart-lb-ip --global --quiet
```

### Step 29 — Delete NAT, router, VMs

```bash
gcloud compute routers nats delete cloudmart-nat \
  --router=cloudmart-router --region=asia-south1 --quiet

gcloud compute routers delete cloudmart-router --region=asia-south1 --quiet

gcloud compute instances delete cloudmart-web-01 cloudmart-web-02 \
  cloudmart-app-private-vm --zone=asia-south1-a --quiet
```

Adjust zones if any VM remains in `asia-south1-b`.

### Step 30 — Delete firewall rules, subnets, VPC

```bash
for rule in cloudmart-allow-internal cloudmart-allow-http cloudmart-allow-lb-health cloudmart-allow-ssh-iap; do
  gcloud compute firewall-rules delete "$rule" --quiet 2>/dev/null || true
done

gcloud compute networks subnets delete cloudmart-private-asia-south1 \
  --region=asia-south1 --quiet

gcloud compute networks subnets delete cloudmart-public-asia-south1 \
  --region=asia-south1 --quiet

gcloud compute networks delete cloudmart-vpc --quiet
```

### Step 31 — Verify cleanup

```bash
gcloud compute networks list --filter="name:cloudmart"
gcloud compute instances list --filter="name~cloudmart"
```

No `cloudmart-*` networking artifacts should remain.

---

## Validation checklist

- [ ] Created custom VPC `cloudmart-vpc`
- [ ] Created public and private subnets with documented CIDRs
- [ ] Enabled Private Google Access on subnets
- [ ] Created internal, HTTP, and health-check firewall rules
- [ ] Reviewed routes including default internet gateway
- [ ] Deployed web VMs with external IPs and private VM without external IP
- [ ] Verified internal connectivity between subnets
- [ ] Configured Cloud Router and Cloud NAT for private egress
- [ ] Verified outbound internet from private VM via NAT
- [ ] Created global HTTP load balancer and observed traffic split
- [ ] Deleted all lab networking resources

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | VPC networks page showing `cloudmart-vpc` and subnets |
| 2 | Firewall rules list for `cloudmart-vpc` |
| 3 | Routes list filtered to `cloudmart-vpc` |
| 4 | VM list showing internal and external IPs |
| 5 | NAT configuration on `cloudmart-router` |
| 6 | `curl` output from load balancer IP (two different backend names) |
| 7 | Backend health check status |
| 8 | Proof of cleanup (empty resource list) |

---

## Knowledge check

1. What is the scope of a **subnet** in Google Cloud?  
   - A) Global across all regions  
   - B) Regional within a VPC  
   - C) Zonal only  

2. Which resource provides **outbound** internet for VMs with only internal IPs?  
   - A) Cloud NAT  
   - B) Ephemeral external IP on every VM  
   - C) DNS only  

3. Which source ranges should be allowed for **Google load balancer health checks** to web servers on port 80?  
   - A) `0.0.0.0/0` only  
   - B) `130.211.0.0/22` and `35.191.0.0/16`  
   - C) `10.0.0.0/8` only  

4. What does a **default route** to the internet gateway enable?  
   - A) Outbound internet for eligible VMs in the subnet  
   - B) Automatic HTTPS certificates  
   - C) Cross-project IAM binding  

5. Which IP address should customers use in production for a multi-VM web tier?  
   - A) Individual VM ephemeral external IPs  
   - B) Load balancer front-end IP  
   - C) Private RFC 1918 address only  

<details>
<summary><strong>Answer key</strong></summary>

1. **B** — Subnets are regional.  
2. **A** — Cloud NAT handles outbound SNAT.  
3. **B** — Google health check probe ranges.  
4. **A** — Internet egress path (with NAT or external IP as designed).  
5. **B** — Use the load balancer VIP for stable, scalable entry.

</details>

---

## Optional challenge

1. Add an **HTTPS** front end with a **Google-managed certificate** and a test domain (if you own one).  
2. Create **secondary IP ranges** on the private subnet for a future **GKE** cluster alias demonstration.  
3. Configure **VPC Flow Logs** on one subnet and query a sample connection in Cloud Logging.  
4. Design a **bastion-less** admin model using **IAP TCP forwarding** only.

---

## Official references

- [VPC network overview](https://cloud.google.com/vpc/docs/vpc)
- [Subnets](https://cloud.google.com/vpc/docs/subnets)
- [VPC firewall rules](https://cloud.google.com/vpc/docs/firewalls)
- [Routes](https://cloud.google.com/vpc/docs/routes)
- [IP addresses](https://cloud.google.com/vpc/docs/ip-addresses)
- [Private Google Access](https://cloud.google.com/vpc/docs/private-google-access)
- [Cloud NAT overview](https://cloud.google.com/nat/docs/overview)
- [External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)
- [Health checks](https://cloud.google.com/load-balancing/docs/health-checks)

---

## Lab outcome

You built a segmented **CloudMart VPC**, applied firewall and routing knowledge, compared internal and external IP behavior, enabled **Cloud NAT** for private egress, and fronted web servers with a **global HTTP load balancer**.

You can take other training labs in any order; none are required before or after this one.