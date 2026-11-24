# Lab 13 — Terraform: Infrastructure as Code for CloudMart

| | |
| --- | --- |
| **Training module** | Module 13 — Infrastructure as Code |
| **Companion presentation** | Presentation 13 — Introduction to Infrastructure as Code |
| **Level** | Beginner to intermediate |
| **Estimated time** | 75–90 minutes |
| **Primary tools** | Terraform (Google provider) |
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
7. [Part 1 — Infrastructure as Code fundamentals](#part-1--infrastructure-as-code-fundamentals)
8. [Part 2 — Install Terraform and authenticate](#part-2--install-terraform-and-authenticate)
9. [Part 3 — Create the Terraform project layout](#part-3--create-the-terraform-project-layout)
10. [Part 4 — Define networking resources](#part-4--define-networking-resources)
11. [Part 5 — Define compute, storage, and firewall](#part-5--define-compute-storage-and-firewall)
12. [Part 6 — Plan, apply, and verify](#part-6--plan-apply-and-verify)
13. [Part 7 — Modify infrastructure and plan again](#part-7--modify-infrastructure-and-plan-again)
14. [Part 8 — State, outputs, and backend concepts](#part-8--state-outputs-and-backend-concepts)
15. [Part 9 — Destroy infrastructure](#part-9--destroy-infrastructure)
16. [Validation checklist](#validation-checklist)
17. [Evidence to submit](#evidence-to-submit)
18. [Knowledge check](#knowledge-check)
19. [Optional challenge](#optional-challenge)
20. [Official references](#official-references)

---

## Lab description

In this lab, you use **Terraform** to define and provision **CloudMart** infrastructure on Google Cloud: a **VPC**, **subnet**, **firewall rule**, a small **Compute Engine** VM, and a **Cloud Storage** bucket. You follow the Terraform workflow—**init**, **fmt**, **validate**, **plan**, **apply**, and **destroy**—matching **Presentation 13**.

**Standalone lab:** You do not need other labs in this folder. All configuration lives in a single Terraform directory created in Cloud Shell.

---

## Scenario and use case

CloudMart operations wants **repeatable dev environments** instead of one-off Console clicks.

| Manual Console | Terraform (IaC) |
| --- | --- |
| Hard to reproduce across teams | Same `.tf` files every time |
| Limited change history | Version control + plan review |
| Drift over time | Desired state in configuration |

**Your role:** Codify a minimal **dev stack** for CloudMart, apply it, make a small change, then **tear it down** with `terraform destroy`.

---

## Learning objectives

When you finish this lab, you should be able to:

- Explain **IaC**, **providers**, **resources**, and **state**
- Structure Terraform files (`providers.tf`, `variables.tf`, `main.tf`, `outputs.tf`)
- Configure the **Google provider** for your project
- Declare dependencies between resources (subnet → VPC)
- Run **`terraform plan`** and interpret a plan summary
- Apply and destroy infrastructure safely
- Describe why **remote state** (GCS backend) matters for teams

---

## What you will build

| Resource | Terraform name / GCP name |
| --- | --- |
| VPC | `cloudmart-vpc` (custom mode) |
| Subnet | `cloudmart-subnet` (`10.10.1.0/24`) |
| Firewall | `cloudmart-allow-ssh-iap` (IAP range, optional tag) |
| VM | `cloudmart-tf-vm` (`e2-micro`, Debian 12) |
| Bucket | `PROJECT_ID-cloudmart-tf-lab` (globally unique) |

**Cost note:** VM, disk, and storage incur charges until **`terraform destroy`**.

---

## Prerequisites

- [ ] Training project with **billing enabled**
- [ ] **Editor** or granular roles for Compute, VPC, Storage
- [ ] **Cloud Shell** (Terraform preinstalled in many sessions; verify in Part 2)
- [ ] Presentation **13** reviewed (recommended)

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="asia-south1"
export ZONE="asia-south1-a"

gcloud config set project "$PROJECT_ID"
```

---

## How to work through this lab

| Part | Focus |
| --- | --- |
| 1–2 | Concepts, Terraform CLI |
| 3–5 | HCL resources |
| 6–7 | Plan / apply / change |
| 8 | State and GCS backend (concept) |
| 9 | **Destroy** |

---

## Part 1 — Infrastructure as Code fundamentals

**Time estimate:** ~10 minutes

### Step 1 — Manual vs automated

**Presentation 13** contrasts manual Console work with version-controlled Terraform for **Dev / QA / Prod** consistency.

### Step 2 — Terraform building blocks

| Block | Purpose |
| --- | --- |
| **Provider** | Connects to Google Cloud APIs |
| **Resource** | Creates or manages infrastructure |
| **Data source** | Reads existing infrastructure |
| **Variable** | Parameterizes configuration |
| **Output** | Exposes values after apply |

**Declarative model:** You describe **desired state**; Terraform calculates the diff.

---

## Part 2 — Install Terraform and authenticate

**Time estimate:** ~10 minutes

### Step 3 — Verify Terraform

```bash
terraform version
```

Cloud Shell usually includes Terraform. If missing, follow [Install Terraform](https://developer.hashicorp.com/terraform/install) or ask your instructor.

---

### Step 4 — Application Default Credentials

Terraform uses your Cloud Shell credentials:

```bash
gcloud auth application-default login
gcloud auth list
```

**Best practice:** Do not hardcode service account keys in `.tf` files.

---

## Part 3 — Create the Terraform project layout

**Time estimate:** ~10 minutes

### Step 5 — Create directories

```bash
mkdir -p ~/cloudmart-terraform && cd ~/cloudmart-terraform
```

### Step 6 — `versions.tf` and provider

```bash
cat > versions.tf <<'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}
EOF

cat > providers.tf <<'EOF'
provider "google" {
  project = var.project_id
  region  = var.region
}
EOF
```

---

### Step 7 — Variables and tfvars

```bash
cat > variables.tf <<'EOF'
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "region" {
  type        = string
  default     = "asia-south1"
}

variable "zone" {
  type        = string
  default     = "asia-south1-a"
}

variable "bucket_name" {
  type        = string
  description = "Globally unique GCS bucket name"
}
EOF

cat > terraform.tfvars <<EOF
project_id  = "${PROJECT_ID}"
region      = "${REGION}"
zone        = "${ZONE}"
bucket_name = "${PROJECT_ID}-cloudmart-tf-lab"
EOF
```

Replace `PROJECT_ID` in `terraform.tfvars` or edit the file after creation.

---

### Step 8 — Outputs (placeholder file)

```bash
cat > outputs.tf <<'EOF'
output "vpc_name" {
  value       = google_compute_network.cloudmart_vpc.name
  description = "VPC network name"
}

output "vm_internal_ip" {
  value       = google_compute_instance.cloudmart_tf_vm.network_interface[0].network_ip
  description = "Internal IP of the lab VM"
}

output "bucket_url" {
  value       = google_storage_bucket.cloudmart_tf_lab.url
  description = "Cloud Storage bucket URL"
}
EOF
```

---

## Part 4 — Define networking resources

**Time estimate:** ~15 minutes

### Step 9 — `main.tf` — VPC and subnet

```bash
cat > main.tf <<'EOF'
resource "google_compute_network" "cloudmart_vpc" {
  name                    = "cloudmart-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "cloudmart_subnet" {
  name          = "cloudmart-subnet"
  ip_cidr_range = "10.10.1.0/24"
  region        = var.region
  network       = google_compute_network.cloudmart_vpc.id
}

resource "google_compute_firewall" "cloudmart_allow_ssh_iap" {
  name    = "cloudmart-allow-ssh-iap"
  network = google_compute_network.cloudmart_vpc.name

  direction = "INGRESS"
  priority  = 1000

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"]
  target_tags   = ["cloudmart-tf"]
}
EOF
```

**Dependency:** Subnet references `cloudmart_vpc.id`—Terraform orders create/delete correctly.

---

## Part 5 — Define compute, storage, and firewall

**Time estimate:** ~10 minutes

### Step 10 — Append VM and bucket to `main.tf`

```bash
cat >> main.tf <<'EOF'

resource "google_compute_instance" "cloudmart_tf_vm" {
  name         = "cloudmart-tf-vm"
  machine_type = "e2-micro"
  zone         = var.zone

  tags = ["cloudmart-tf"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 20
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.cloudmart_subnet.id
    access_config {
      # Ephemeral public IP for lab SSH/testing
    }
  }

  metadata_startup_script = <<-EOT
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    echo "CloudMart Terraform Lab" > /var/www/html/index.html
    systemctl enable --now nginx
  EOT
}

resource "google_storage_bucket" "cloudmart_tf_lab" {
  name     = var.bucket_name
  location = var.region

  uniform_bucket_level_access = true

  force_destroy = true
}
EOF
```

`force_destroy = true` allows bucket deletion in the lab when non-empty (use only in training projects).

---

## Part 6 — Plan, apply, and verify

**Time estimate:** ~20 minutes

### Step 11 — Initialize and validate

```bash
cd ~/cloudmart-terraform

terraform fmt -recursive
terraform init
terraform validate
```

---

### Step 12 — Plan and apply

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

Type `yes` if prompted (when not using saved plan file, use `terraform apply` and confirm).

---

### Step 13 — Verify outputs and GCP

```bash
terraform output
```

Console:

- **VPC network** → `cloudmart-vpc`
- **VM instances** → `cloudmart-tf-vm`
- **Cloud Storage** → bucket name from tfvars

Optional:

```bash
gcloud compute instances list --filter="name:cloudmart-tf-vm"
```

---

## Part 7 — Modify infrastructure and plan again

**Time estimate:** ~10 minutes

### Step 14 — Change VM machine type or disk size

Edit `main.tf`: change `machine_type` to `e2-small` **or** increase boot disk to `25`.

```bash
terraform plan
terraform apply
```

**Expected result:** Plan shows **1 to change** (in-place or replacement depending on attribute).

Demonstrates **drift detection** when someone changes Console manually—run `terraform plan` after a manual edit if instructor demos drift.

---

## Part 8 — State, outputs, and backend concepts

**Time estimate:** ~10 minutes

### Step 15 — Inspect local state

```bash
terraform state list
terraform state show google_compute_network.cloudmart_vpc | head -20
```

**State** maps Terraform addresses to real resource IDs. **Do not commit secrets**; protect state files.

---

### Step 16 — Remote backend (concept)

**Presentation 13** recommends a **GCS backend** for teams:

```hcl
terraform {
  backend "gcs" {
    bucket = "YOUR-TERRAFORM-STATE-BUCKET"
    prefix = "cloudmart/dev"
  }
}
```

You **do not** enable this backend in the lab (requires a pre-created state bucket). Discuss **state locking** and **IAM** on the bucket.

---

## Part 9 — Destroy infrastructure

**Time estimate:** ~10 minutes

### Step 17 — Destroy all managed resources

```bash
terraform destroy
```

Confirm with `yes`.

---

### Step 18 — Verify cleanup

```bash
gcloud compute networks list --filter="name:cloudmart-vpc"
gcloud compute instances list --filter="name:cloudmart-tf"
gcloud storage buckets list --filter="name:cloudmart-tf-lab"
```

Remove project files if desired:

```bash
rm -rf ~/cloudmart-terraform
```

---

## Validation checklist

- [ ] Ran `terraform init`, `validate`, `plan`, `apply`
- [ ] Created VPC, subnet, firewall, VM, and bucket
- [ ] Read `terraform output`
- [ ] Made a change and applied an update
- [ ] Listed resources in `terraform state list`
- [ ] Ran **`terraform destroy`** successfully

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | `terraform plan` summary (create) |
| 2 | `terraform apply` success |
| 3 | `terraform output` |
| 4 | Console screenshot of `cloudmart-tf-vm` |
| 5 | `terraform plan` after a config change |
| 6 | `terraform destroy` success |

---

## Knowledge check

1. What language does Terraform use for configuration?  
   - A) HCL  
   - B) SQL  
   - C) YAML only  

2. Which command shows planned changes without applying?  
   - A) `terraform plan`  
   - B) `terraform destroy`  
   - C) `kubectl apply`  

3. What file tracks managed resource IDs?  
   - A) Terraform state  
   - B) Dockerfile  
   - C) package.json  

4. Why reference `google_compute_network.cloudmart_vpc.id` in the subnet?  
   - A) Implicit dependency ordering  
   - B) To enable BigQuery  
   - C) To create DNS records automatically  

5. Why use a remote GCS backend in production teams?  
   - A) Shared state, locking, and collaboration  
   - B) Replace Cloud Build  
   - C) Delete VMs faster  

<details>
<summary><strong>Answer key</strong></summary>

1. **A** — HCL.  
2. **A** — `terraform plan`.  
3. **A** — State file.  
4. **A** — Dependencies.  
5. **A** — Centralized state.

</details>

---

## Optional challenge

1. Add a **data** source `google_client_config` and output `project`.  
2. Split networking into a **module** `modules/networking` and call it from root (Presentation 13 pattern).  
3. Configure a **GCS backend** in a dedicated state bucket (instructor-provided).

---

## Official references

- [Terraform on Google Cloud](https://cloud.google.com/docs/terraform)
- [Google provider documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Terraform workflow](https://developer.hashicorp.com/terraform/intro/core-workflow)
- [Backend type: gcs](https://developer.hashicorp.com/terraform/language/settings/backends/gcs)
- Presentation 13 — *Introduction to Infrastructure as Code* (course materials)

---

## Lab outcome

You defined **CloudMart** infrastructure as code, provisioned it with Terraform, updated configuration, inspected **state**, and **destroyed** all resources.

You can take other training labs in any order; none are required before or after this one.