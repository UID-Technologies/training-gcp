# Lab 01 — Deploy and Manage a Virtual Machine Using Google Compute Engine

| | |
| --- | --- |
| **Training day** | Day 1 — GCP Fundamentals |
| **Level** | Beginner |
| **Estimated time** | 90–120 minutes |
| **Primary service** | Google Compute Engine |
| **Lab ID** | `cloudmart-web-vm` |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [Lab architecture](#lab-architecture)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Create a VM (Google Cloud Console)](#part-1--create-a-vm-google-cloud-console)
8. [Part 2 — Install Nginx and deploy the CloudMart app](#part-2--install-nginx-and-deploy-the-cloudmart-app)
9. [Part 3 — Configure VPC firewall for HTTP](#part-3--configure-vpc-firewall-for-http)
10. [Part 4 — Provision and verify with Google Cloud CLI](#part-4--provision-and-verify-with-google-cloud-cli)
11. [Part 5 — VM lifecycle management](#part-5--vm-lifecycle-management)
12. [Part 6 — Troubleshooting practice](#part-6--troubleshooting-practice)
13. [Part 7 — Clean up lab resources](#part-7--clean-up-lab-resources)
14. [Validation checklist](#validation-checklist)
15. [Evidence to submit](#evidence-to-submit)
16. [Knowledge check](#knowledge-check)
17. [Optional challenge](#optional-challenge)
18. [Official references](#official-references)

---

## Lab description

In this hands-on lab, you provision, configure, access, monitor, and manage a Linux virtual machine (VM) on **Google Compute Engine**.

You deploy a sample **CloudMart** web application using the **Nginx** web server and expose it on the public internet with HTTP. Along the way you practice:

- Creating and configuring a VM (Console and `gcloud`)
- Connecting with SSH
- Installing and testing Nginx
- Creating a VPC firewall rule with network tags
- Stop, start, and reset operations
- Basic troubleshooting and **resource cleanup** (to avoid ongoing charges)

This lab supports **Day 1, Module 4** (infrastructure fundamentals) in the GCP training program.

**Standalone lab:** You do not need any other lab in this folder. All steps use only this project and the resources created here.

---

## Scenario and use case

### CloudMart development web server

**CloudMart** is building an e-commerce platform. The development team needs a cost-effective Linux server in Google Cloud to host an initial homepage before later moving to containers and Kubernetes.

**Your role:** Act as a cloud administrator and deliver an environment that meets these requirements:

| # | Requirement |
| --- | --- |
| 1 | Create a Linux VM in Google Cloud |
| 2 | Use a development-friendly, cost-conscious machine size |
| 3 | Install and configure Nginx |
| 4 | Deploy a sample CloudMart homepage |
| 5 | Allow customers to reach the site over **HTTP (TCP 80)** |
| 6 | Allow administrators to connect with **SSH** |
| 7 | Monitor the VM and perform basic lifecycle operations |
| 8 | Delete lab resources when finished |

The app runs on the VM in this lab; in production, teams often later containerize similar workloads on Google Kubernetes Engine.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain what Compute Engine provides and when to use it
- Create a VM with appropriate region, zone, machine type, and OS image
- Describe boot disks, internal IPs, and external IPs
- Connect to a Linux VM with SSH (Console and CLI)
- Install Nginx and serve custom HTML content
- Create a VPC ingress firewall rule using **target tags**
- Open the application in a browser using the VM’s external IP
- Stop, start, reset, and delete VM instances
- Run common Compute Engine tasks with `gcloud`

---

## Lab architecture

**Traffic path:** Browser → External IP → VPC firewall → Compute Engine VM → Nginx

### Configuration reference

| Component | Lab value |
| --- | --- |
| VM name | `cloudmart-web-vm` |
| Region / zone | `asia-south1` (Mumbai) / `asia-south1-a` |
| Machine type | `e2-micro` (E2 family) |
| OS | Debian GNU/Linux 12 (public image) |
| Boot disk | 20 GB, `pd-balanced` |
| VPC / subnet | `default` / default subnet in `asia-south1` |
| External IP | Ephemeral IPv4 |
| Network tag | `cloudmart-web` |
| Firewall rule | `allow-cloudmart-http` (TCP 80) |
| Web server | Nginx on port 80 |

**Cost note:** `e2-micro` free-tier benefits apply only in eligible US regions and under specific conditions. Storage, Mumbai region usage, and external IPv4 may incur charges. Always run **Part 7 — Clean up** when you are done.

---

## Prerequisites

Before you start, confirm the following:

- [ ] Active Google Cloud account and a **project with billing enabled**
- [ ] Permission to create Compute Engine VMs and VPC firewall rules (e.g. Compute Admin or equivalent custom role)
- [ ] Access to [Google Cloud Console](https://console.cloud.google.com/) and **Cloud Shell**
- [ ] Web browser and internet connection
- [ ] Comfort with basic Linux commands (`cd`, `sudo`, `curl`)

**VPC:** This lab uses the **`default`** VPC. If your organization does not create a default network automatically, ask your administrator or create a compatible VPC and subnet in `asia-south1` before continuing.

---

## How to work through this lab

You can use **either** path below; both teach the same outcomes.

| Path | When to use it |
| --- | --- |
| **A — Google Cloud Console** | Best for first-time learners; Parts 1–3 and 5–7 include Console steps. |
| **B — Google Cloud CLI (`gcloud`)** | Part 4 automates creation; individual steps also show CLI alternatives. |

**Suggested order:** Complete Parts **1 → 2 → 3** on the Console, then Part **4** (CLI literacy), Parts **5–6**, and Part **7** (cleanup).

> **Duplicate resources:** If you already created `cloudmart-web-vm` in Part 1, do **not** run the `gcloud compute instances create` command in Part 4 with the same name unless you delete the VM first.

---

## Part 1 — Create a VM (Google Cloud Console)

**Time estimate:** ~30 minutes

### Step 1 — Open Google Cloud Console and select your project

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Sign in with your Google account.
3. Click the **project selector** at the top of the page.
4. Select the project assigned for this training.
5. Confirm the project name appears in the top navigation bar.

**Why this matters:** A project is the billing and administrative boundary. Every resource you create in this lab belongs to this project.

---

### Step 2 — Enable the Compute Engine API

1. Open the **Navigation menu** (☰).
2. Go to **APIs & Services → Library**.
3. Search for **Compute Engine API**.
4. Open **Compute Engine API**.
5. Click **Enable** if it is not already enabled.
6. Wait until enablement completes.

Opening **Compute Engine** for the first time may also prompt you to enable the API.

**CLI alternative (Cloud Shell):**

```bash
gcloud config get-value project
gcloud services enable compute.googleapis.com
```

**Expected result:** Compute Engine API is enabled for your project.

---

### Step 3 — Open the VM creation workflow

1. Open the **Navigation menu**.
2. Select **Compute Engine → VM instances**.
3. Click **Create instance**.

Direct link: [Compute Engine — VM instances](https://console.cloud.google.com/compute/instances).

The creation form includes **Machine configuration**, **OS and storage**, **Networking**, **Security**, and related sections.

---

### Step 4 — Configure machine settings

In **Machine configuration**, enter:

| Field | Value |
| --- | --- |
| Name | `cloudmart-web-vm` |
| Region | `asia-south1 (Mumbai)` |
| Zone | `asia-south1-a` |
| Machine family | General purpose |
| Series | E2 |
| Machine type | `e2-micro` |

If **Zone** shows **Any**, choose **`asia-south1-a`** explicitly so Console and CLI exercises match.

**Concepts:**

- **Region:** Geographic area for your resources.
- **Zone:** Isolated location within a region (for fault isolation).
- **Machine type:** vCPU and memory (`e2-micro` is suitable for this lab, not for production workloads).

---

### Step 5 — Configure operating system and boot disk

1. Select **OS and storage** in the left panel (or scroll to that section).
2. Under **Boot disk**, click **Change**.
3. On **Public images**, set:

| Field | Value |
| --- | --- |
| Operating system | Debian |
| Version | Debian GNU/Linux 12 |
| Boot disk type | Balanced persistent disk |
| Size | 20 GB |

4. Click **Select**.

**Why this matters:** The boot disk holds the OS and your application files. Persistent Disk survives VM stop/start (unlike the ephemeral VM runtime).

---

### Step 6 — Configure networking

Select **Networking**.

**Firewall (on the VM form):**

- Leave **Allow HTTP traffic** unchecked for now.
- Leave **Allow HTTPS traffic** unchecked.

You will create a dedicated firewall rule in Part 3 to practice **target tags**.

**Network interface** — verify:

| Field | Value |
| --- | --- |
| Network | `default` |
| Subnetwork | Default subnet in `asia-south1` |
| Primary internal IPv4 | Ephemeral (automatic) |
| External IPv4 | Ephemeral |

**Network tags:** add

```text
cloudmart-web
```

This tag links the VM to the HTTP firewall rule you create later.

---

### Step 7 — Review security and advanced options

For this lab, keep defaults unless your organization specifies otherwise:

- Default **Shielded VM** settings
- **Confidential VM** disabled
- Standard provisioning model (not Spot)
- No service account key download
- Use the project default compute service account unless your lab policy requires a dedicated service account with minimal scopes

Do **not** enable the public HTTP checkbox on the VM form yet.

---

### Step 8 — Create the VM

1. Review the summary and estimated cost.
2. Click **Create**.
3. Wait until provisioning completes.
4. Return to **Compute Engine → VM instances**.

**Expected result:**

| Property | Expected value |
| --- | --- |
| Name | `cloudmart-web-vm` |
| Status | Running (green indicator) |
| Zone | `asia-south1-a` |
| Internal / External IP | Assigned automatically |
| SSH | Available in the **Connect** column |

---

### Step 9 — Connect with SSH and inspect the VM

1. On the VM instances page, find `cloudmart-web-vm`.
2. Click **SSH** in the **Connect** column.
3. Wait for the browser SSH window to open.
4. Confirm you have a Linux shell prompt.

In the SSH session, run:

```bash
whoami
hostname
pwd
cat /etc/os-release
```

Then inspect resources:

```bash
free -h
df -h
ip addr
```

**What you learned:** You connected to a cloud VM and verified OS, memory, disk, and network configuration.

---

## Part 2 — Install Nginx and deploy the CloudMart app

**Time estimate:** ~20 minutes  
**Where:** SSH session on `cloudmart-web-vm`

### Step 10 — Update package indexes

```bash
sudo apt-get update
```

---

### Step 11 — Install and start Nginx

```bash
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

**Expected result:** `Active: active (running)`. Press `q` if output uses a pager.

---

### Step 12 — Test Nginx locally

```bash
curl http://localhost
sudo ss -tlnp | grep ':80 '
```

**Expected result:** HTML from the default Nginx page; port 80 is listening.

**Note:** The site is not reachable from the internet until Part 3 configures the firewall.

---

### Step 13 — Deploy the CloudMart homepage

Create the sample page:

```bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>CloudMart - GCP Lab</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #f4f7fb;
            text-align: center;
            padding: 70px 20px;
        }
        h1 { color: #1a73e8; }
        .card {
            background: white;
            padding: 35px;
            border-radius: 12px;
            max-width: 600px;
            margin: auto;
            box-shadow: 0 4px 15px #ddd;
        }
    </style>
</head>
<body>
    <div class="card">
        <h1>Welcome to CloudMart</h1>
        <h2>Google Compute Engine Lab</h2>
        <p>Application successfully deployed on GCP.</p>
        <p>Web Server: Nginx</p>
        <p>Environment: Development</p>
        <p>Version: 1.0</p>
    </div>
</body>
</html>
EOF
```

Verify:

```bash
curl -I http://localhost
```

**Expected result:** `HTTP/1.1 200 OK` and `Server: nginx`.

---

## Part 3 — Configure VPC firewall for HTTP

**Time estimate:** ~15 minutes

### Step 14 — Understand the firewall requirement

VPC firewall rules control **ingress** and **egress** traffic. You need to allow **TCP port 80** to VMs with tag `cloudmart-web`.

| Setting | Value |
| --- | --- |
| Rule name | `allow-cloudmart-http` |
| Network | `default` |
| Direction | Ingress |
| Action | Allow |
| Targets | Specified target tags |
| Target tag | `cloudmart-web` |
| Source IPv4 ranges | `0.0.0.0/0` |
| Protocol / ports | TCP `80` |

`0.0.0.0/0` allows any IPv4 source—acceptable for a public demo site in a lab, but **not** a pattern for administrative or database ports in production.

---

### Step 15 — Create the rule (Console)

1. **Navigation menu → VPC network → Firewall** (label may appear as **Firewall policies** in some views).
2. Click **Create firewall rule**.
3. Enter the values from Step 14.
4. Under **Protocols and ports**, choose **Specified protocols and ports**, select **TCP**, and enter `80`.
5. Click **Create**.

Use **target tags**, not “all instances in the network.”

---

### Step 16 — Create the rule (CLI alternative)

In **Cloud Shell**:

```bash
gcloud compute firewall-rules create allow-cloudmart-http \
  --network=default \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=cloudmart-web
```

Verify:

```bash
gcloud compute firewall-rules describe allow-cloudmart-http
```

> Create the rule **once**—either in the Console **or** with CLI, not both.

---

### Step 17 — Access the app from your browser

1. Go to **Compute Engine → VM instances**.
2. Copy the **External IP** of `cloudmart-web-vm`.
3. In a new browser tab, open `http://EXTERNAL_IP` (replace with your IP).

**Expected result:** The CloudMart homepage loads over HTTP.

Production sites should use **HTTPS**; this lab uses HTTP for simplicity.

---

## Part 4 — Provision and verify with Google Cloud CLI

**Time estimate:** 20–25 minutes  
**Where:** Cloud Shell

### Step 18 — Configure Cloud Shell context

1. Click **Activate Cloud Shell** in the Console toolbar.
2. Run:

```bash
gcloud version
gcloud projects list
gcloud config set project YOUR_PROJECT_ID
gcloud config set compute/region asia-south1
gcloud config set compute/zone asia-south1-a
gcloud config list
```

Replace `YOUR_PROJECT_ID` with your training project ID.

---

### Step 19 — Create the VM with `gcloud` (optional if VM already exists)

Skip this step if `cloudmart-web-vm` already exists from Part 1.

```bash
gcloud compute instances create cloudmart-web-vm \
  --zone=asia-south1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=20GB \
  --boot-disk-type=pd-balanced \
  --network=default \
  --subnet=default \
  --network-tier=PREMIUM \
  --tags=cloudmart-web
```

| Flag | Purpose |
| --- | --- |
| `--zone` | Deployment zone |
| `--machine-type` | CPU and memory shape |
| `--image-family` / `--image-project` | OS image |
| `--boot-disk-size` / `--boot-disk-type` | Boot persistent disk |
| `--network` / `--subnet` | VPC attachment |
| `--tags` | Network tags for firewall targeting |

Add `--service-account` and `--scopes` only if your organization requires them.

---

### Step 20 — Verify the instance

```bash
gcloud compute instances list

gcloud compute instances describe cloudmart-web-vm \
  --zone=asia-south1-a

gcloud compute instances describe cloudmart-web-vm \
  --zone=asia-south1-a \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)'
```

---

### Step 21 — SSH from Cloud Shell

```bash
gcloud compute ssh cloudmart-web-vm --zone=asia-south1-a
```

On first connect, accept key generation if prompted. Run **Steps 10–13** if Nginx is not yet installed, then type `exit`.

---

### Step 22 — Test HTTP from Cloud Shell

Ensure the firewall rule from Part 3 exists, then:

```bash
EXTERNAL_IP=$(gcloud compute instances describe cloudmart-web-vm \
  --zone=asia-south1-a \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

echo "$EXTERNAL_IP"
curl -I "http://$EXTERNAL_IP"
```

**Expected result:** `HTTP/1.1 200 OK`.

---

## Part 5 — VM lifecycle management

**Time estimate:** ~15 minutes

### Step 23 — Stop the VM

**Console:** **Compute Engine → VM instances** → select `cloudmart-web-vm` → **Stop** → confirm.

**CLI:**

```bash
gcloud compute instances stop cloudmart-web-vm --zone=asia-south1-a
```

**When stopped:**

- vCPU and RAM are not billed for the VM runtime (storage may still bill).
- Boot disk remains attached.
- An **ephemeral** external IP may change after the next start.

---

### Step 24 — Start the VM

**Console:** Select the stopped VM → **Start**.

**CLI:**

```bash
gcloud compute instances start cloudmart-web-vm --zone=asia-south1-a
gcloud compute instances list
```

Re-check **External IP** before testing the website again.

---

### Step 25 — Reset the VM

**Console:** Instance **⋮** menu → **Reset**.

**CLI:**

```bash
gcloud compute instances reset cloudmart-web-vm --zone=asia-south1-a
```

Reset is a **hard** power cycle, not a graceful OS shutdown. For a normal OS reboot, use `sudo reboot` inside the VM.

---

### Step 26 — Inspect metrics

**Console:** Open the VM details page → **Observability** (or monitoring tab) → review CPU and network charts.

**CLI / OS:**

```bash
gcloud compute instances describe cloudmart-web-vm \
  --zone=asia-south1-a \
  --format="yaml(name,status,machineType,disks,networkInterfaces)"
```

In SSH:

```bash
uptime
free -h
df -h
top
```

Press `q` to leave `top`.

---

## Part 6 — Troubleshooting practice

Use this table when something fails during the lab:

| Symptom | Likely cause | What to try |
| --- | --- | --- |
| VM creation fails | API off, quota, IAM, billing | Enable API, check quotas and permissions |
| SSH fails | IAM, OS Login, firewall on port 22 | Verify roles and SSH access method |
| `curl localhost` fails | Nginx stopped | `sudo systemctl status nginx` |
| Browser cannot reach site | No firewall rule | Create or verify `allow-cloudmart-http` |
| Rule exists, still blocked | Wrong network tag | VM must have tag `cloudmart-web` |
| No external IP | No access config on NIC | Review network interface settings |
| Site worked before stop/start | External IP changed | Read current external IP in Console |
| HTTP 403/404 | Missing or wrong web root file | Check `/var/www/html/index.html` |

### Exercise A — Nginx stopped

```bash
sudo systemctl stop nginx
curl http://localhost
sudo systemctl start nginx
curl -I http://localhost
```

### Exercise B — Firewall rule disabled

In Cloud Shell:

```bash
gcloud compute firewall-rules update allow-cloudmart-http --disabled
```

Test the site in your browser (should fail). Restore:

```bash
gcloud compute firewall-rules update allow-cloudmart-http --no-disabled
```

---

## Part 7 — Clean up lab resources

**Time estimate:** 5–10 minutes

Unused VMs and disks can continue to generate charges.

### Step 27 — Delete the VM (Console)

1. **Compute Engine → VM instances**.
2. Select `cloudmart-web-vm`.
3. Click **Delete**.
4. Confirm **boot disk** deletion when prompted.
5. Confirm deletion.

### Step 28 — Delete the VM (CLI)

```bash
gcloud compute instances delete cloudmart-web-vm --zone=asia-south1-a
```

Confirm when prompted.

### Step 29 — Delete the firewall rule

**Console:** **VPC network → Firewall** → select `allow-cloudmart-http` → **Delete**.

**CLI:**

```bash
gcloud compute firewall-rules delete allow-cloudmart-http
```

### Step 30 — Verify cleanup

```bash
gcloud compute instances list
gcloud compute firewall-rules list --filter="name=allow-cloudmart-http"
```

Also check **Compute Engine → Disks** for orphaned boot disks.

Do **not** delete the shared `default` VPC or unrelated training resources.

---

## Validation checklist

Mark each item when complete:

- [ ] Selected the correct Google Cloud project
- [ ] Enabled Compute Engine API
- [ ] Created `cloudmart-web-vm`
- [ ] Configured Debian 12 and 20 GB boot disk
- [ ] Connected with SSH
- [ ] Installed and started Nginx
- [ ] Deployed the CloudMart HTML homepage
- [ ] Created `allow-cloudmart-http` firewall rule
- [ ] Opened the app via external IP in a browser
- [ ] Ran basic `gcloud` instance commands
- [ ] Tested stop and start
- [ ] Completed at least one troubleshooting exercise
- [ ] Deleted VM, rule, and verified no stray disks

---

## Evidence to submit

Capture screenshots **before** cleanup (if your instructor requires proof):

| # | Screenshot |
| --- | --- |
| 1 | VM instance running in Compute Engine |
| 2 | VM machine configuration |
| 3 | Successful SSH session |
| 4 | Nginx `active (running)` status |
| 5 | Firewall rule `allow-cloudmart-http` |
| 6 | CloudMart page in browser |
| 7 | Output of `gcloud compute instances list` |

---

## Knowledge check

1. What is the purpose of a Compute Engine machine type?  
   - A) Configure IAM permissions  
   - B) Define CPU and memory resources  
   - C) Create a VPC firewall rule  

2. Which port does HTTP use?  
   - A) 22  
   - B) 443  
   - C) 80  

3. What happens to a persistent boot disk when a VM is stopped?  
   - A) It is automatically deleted  
   - B) It remains attached to the stopped VM  
   - C) It becomes a Cloud Storage bucket  

4. Which command lists Compute Engine instances?  
   - A) `gcloud compute instances list`  
   - B) `gcloud vm show`  
   - C) `gcloud instances get`  

5. Why did we use the `cloudmart-web` network tag?  
   - A) To choose the Debian image  
   - B) To increase CPU capacity  
   - C) To target the VM with a firewall rule  

<details>
<summary><strong>Answer key</strong> (check after you answer)</summary>

1. **B** — Machine types define vCPU and memory.  
2. **C** — HTTP uses port 80.  
3. **B** — Persistent disks stay attached when the VM is stopped.  
4. **A** — `gcloud compute instances list`.  
5. **C** — Tags select which VMs a firewall rule applies to.

</details>

---

## Optional challenge

**Second web server:** Create `cloudmart-web-vm-02` in the same zone, install Nginx, customize the homepage to show **CloudMart Server 02**, and confirm both external IPs serve different content. This prepares you for instance templates, managed instance groups, and load balancing.

---

## Official references

Console labels and CLI flags may vary slightly by organization policy and Console version. Use Google’s documentation as the source of truth:

- [Create and start Compute Engine instances](https://cloud.google.com/compute/docs/instances/create-start-instance)
- [Create a VM from a public image](https://cloud.google.com/compute/docs/instances/create-vm-from-public-image)
- [Connect to Linux VMs using SSH](https://cloud.google.com/compute/docs/instances/connecting-to-instance)
- [Create and manage VPC firewall rules](https://cloud.google.com/firewall/docs/using-firewalls)

---

## Lab outcome

You provisioned a Linux VM on Google Cloud, installed Nginx, published a sample application, configured network access with firewall tags, managed the VM lifecycle, practiced troubleshooting, and cleaned up resources.

You can take other training labs in any order; none are required before or after this one.