# Lab 02 — Google Cloud Console, CLI, and Identity and Access Management

| | |
| --- | --- |
| **Training module** | Module 2 — GCP Console, CLI, and Identity Management |
| **Level** | Beginner |
| **Estimated time** | 90–120 minutes |
| **Primary services** | Google Cloud Console, Cloud Shell, Cloud SDK (`gcloud`), IAM |
| **Lab prefix** | `cloudmart` |

---

## Table of contents

1. [Lab description](#lab-description)
2. [Scenario and use case](#scenario-and-use-case)
3. [Learning objectives](#learning-objectives)
4. [Lab components](#lab-components)
5. [Prerequisites](#prerequisites)
6. [How to work through this lab](#how-to-work-through-this-lab)
7. [Part 1 — Google Cloud Console overview](#part-1--google-cloud-console-overview)
8. [Part 2 — Google Cloud Shell](#part-2--google-cloud-shell)
9. [Part 3 — Install and configure the Google Cloud CLI](#part-3--install-and-configure-the-google-cloud-cli)
10. [Part 4 — IAM concepts in the Console](#part-4--iam-concepts-in-the-console)
11. [Part 5 — Users, groups, roles, and permissions](#part-5--users-groups-roles-and-permissions)
12. [Part 6 — Service accounts and authentication](#part-6--service-accounts-and-authentication)
13. [Part 7 — IAM security best practices](#part-7--iam-security-best-practices)
14. [Part 8 — Clean up lab resources](#part-8--clean-up-lab-resources)
15. [Validation checklist](#validation-checklist)
16. [Evidence to submit](#evidence-to-submit)
17. [Knowledge check](#knowledge-check)
18. [Optional challenge](#optional-challenge)
19. [Official references](#official-references)

---

## Lab description

In this hands-on lab, you explore how administrators and developers interact with Google Cloud through the **Console**, **Cloud Shell**, and the **Google Cloud CLI (`gcloud`)**. You then apply **Identity and Access Management (IAM)** to control who can do what in the **CloudMart** training project.

You will:

- Navigate the Console and interpret project context, APIs, and activity
- Use Cloud Shell as a managed environment for administration
- Install and configure `gcloud` on your own computer (optional but recommended)
- Read and explain IAM policies, roles, and principals
- Grant **least-privilege** access to users and groups
- Create **service accounts** for applications and automation
- Practice **secure authentication** patterns (no long-lived keys where avoidable)
- Review IAM hygiene and cleanup lab identities

This lab aligns with **Module 2: GCP Console, CLI and Identity Management**.

**Standalone lab:** You do not need any other lab in this folder. All steps use only this project and the resources created here.

---

## Scenario and use case

### CloudMart platform access model

CloudMart’s security team is preparing for production. Different people and systems need different access:

| Identity type | Example | Typical need |
| --- | --- | --- |
| **Human developers** | `developer@cloudmart.example` | View resources, deploy to dev, read logs |
| **Human operators** | `ops@cloudmart.example` | Manage Compute Engine, networking, monitoring |
| **Application runtime** | VM or service workload | Write logs and metrics only |
| **CI/CD automation** | Build pipeline | Push images, deploy to specific resources |

**Your role:** As a cloud administrator, you must:

1. Document how the team accesses Google Cloud (Console vs CLI).
2. Define IAM bindings using **predefined roles** (and understand **custom roles**).
3. Create dedicated **service accounts** for the app and for deployment automation.
4. Avoid overly broad roles (such as **Project Owner** or **Editor**) on everyday identities.
5. Remove temporary lab grants and service accounts when finished.

---

## Learning objectives

When you finish this lab, you should be able to:

- Describe major Console areas: project selector, navigation, search, APIs, IAM, and Activity/Audit logs
- Start Cloud Shell, use its persistent home directory, and run `gcloud` safely
- Install Cloud SDK locally and configure project, region, and authentication
- Explain IAM terms: **principal**, **role**, **permission**, **policy**, **allow policy**
- Differentiate **basic**, **predefined**, and **custom** roles
- Add and remove IAM bindings for users and Google Groups
- Create service accounts and assign minimal roles
- Compare user credentials, service account keys, **attached service accounts**, and **impersonation**
- Apply IAM security best practices: least privilege, separation of duties, no key sprawl, auditability

---

## Lab components

| Artifact | Name / value | Purpose |
| --- | --- | --- |
| Training project | Your assigned project ID | IAM policy scope for this lab |
| Developer principal | Lab user or teammate Google account | Human access via predefined role |
| Group (optional) | `cloudmart-developers@yourdomain.com` | Group-based IAM binding |
| Application SA | `cloudmart-app-sa` | Runtime identity (logs/metrics) |
| CI/CD SA (optional) | `cloudmart-cicd-sa` | Narrow deploy automation identity |
| Custom role (optional) | `cloudmartLogViewer` | Demonstrate custom role (if permitted) |
| Predefined roles used | `roles/viewer`, `roles/logging.viewer`, `roles/monitoring.metricWriter`, `roles/logging.logWriter`, `roles/iam.serviceAccountUser` | Least-privilege examples |

**Important:** Exact role names and whether you can create custom roles depend on organization policy. Use the roles your instructor approves.

---

## Prerequisites

Before you start, confirm the following:

- [ ] Google Cloud account with access to a **training project**
- [ ] One of: **Project Owner**, **Project IAM Admin** (`roles/resourcemanager.projectIamAdmin`), or equivalent custom permissions to modify IAM and create service accounts
- [ ] Billing enabled on the project (required for some APIs; this lab is mostly IAM and has minimal cost)
- [ ] Web browser and internet access
- [ ] For Part 3: permission to install software on your laptop (or skip Part 3 and use Cloud Shell only)
- [ ] Optional: second Google account or a **Google Group** your instructor created for group-binding exercises

**Instructor / org notes:**

- Learners without IAM Admin cannot complete binding exercises; pair them with an admin or use a dedicated sandbox project.
- If Google Groups are not available, bind a second individual user account instead.

---

## How to work through this lab

| Path | Description |
| --- | --- |
| **Console-first** | Parts 1, 2, 4, 5, 6, 7 — primary UI workflows |
| **CLI parallel** | Each part includes `gcloud` commands; Part 3 focuses on local SDK |
| **Minimal cost** | No chargeable compute required; service accounts and IAM are free |

**Record your project ID** before you begin (you will use it in almost every command):

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
```

In Cloud Shell on Windows learners, use:

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
```

For PowerShell (local CLI):

```powershell
$env:PROJECT_ID = "YOUR_PROJECT_ID"
```

---

## Part 1 — Google Cloud Console overview

**Time estimate:** ~15 minutes

### Step 1 — Sign in and select the project

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Sign in with your training Google account.
3. Click the **project selector** at the top.
4. Select your training project.
5. Note the **Project name**, **Project ID**, and **Project number** (visible in **IAM & Admin → Settings**).

**Why this matters:** IAM policies attach to **resources** (project, folder, organization, or individual resources). Most lab work is at the **project** level.

---

### Step 2 — Tour the Console chrome

Explore these UI elements and note what each is for:

| UI element | Purpose |
| --- | --- |
| **Navigation menu** (☰) | Service categories (Compute, IAM, etc.) |
| **Search** (top bar) | Jump to services, docs, and resources by name |
| **Cloud Shell** icon | Browser-based terminal with `gcloud` preinstalled |
| **Notifications** | Long-running operations and alerts |
| **Help** (?) | Documentation and support links |
| **User account menu** | Account switcher, theme, Cloud Shell quota |

**Exercise:** Use search to open **IAM & Admin → IAM** without browsing the full menu.

---

### Step 3 — APIs and enabled services

1. Go to **APIs & Services → Enabled APIs & services**.
2. Review which APIs are already enabled on your project.
3. Open **APIs & Services → Library**.
4. Search for **Identity and Access Management (IAM) API** — it is enabled automatically on standard projects; confirm it appears under enabled services if listed.

Optional CLI:

```bash
gcloud services list --enabled --project="$PROJECT_ID" | head -20
```

---

### Step 4 — Activity and audit visibility

1. Go to **IAM & Admin → Audit logs** (or **Logging → Log Explorer** if your project uses Cloud Logging views).
2. In **Log Explorer**, run a simple query for admin activity (example):

```text
logName="projects/YOUR_PROJECT_ID/logs/cloudaudit.googleapis.com%2Factivity"
```

Replace `YOUR_PROJECT_ID` in the query.

3. Open **IAM & Admin → Activity** (if available) to see recent configuration changes.

**What you learned:** The Console is the control plane for configuration; **audit logs** provide accountability for IAM changes.

---

## Part 2 — Google Cloud Shell

**Time estimate:** ~15 minutes

### Step 5 — Start Cloud Shell

1. Click **Activate Cloud Shell** in the Console toolbar.
2. Wait for the session to provision (first launch may take a minute).
3. Confirm the prompt shows your user and project.

Cloud Shell provides:

- Temporary compute (session timeout)
- **Persistent 5 GB home directory** (`$HOME`)
- Pre-authenticated `gcloud` as **your user account**

---

### Step 6 — Basic Cloud Shell workflow

In Cloud Shell, run:

```bash
gcloud version
gcloud config list
whoami
pwd
ls -la ~
```

Create a lab notes file in your persistent home directory:

```bash
mkdir -p ~/cloudmart-labs
echo "CloudMart Module 2 — $(date -u +%Y-%m-%d)" > ~/cloudmart-labs/module2-notes.txt
cat ~/cloudmart-labs/module2-notes.txt
```

**Editor and upload:** Use the **Open editor** button to edit files, or **Upload file** from the Shell menu to transfer small scripts.

---

### Step 7 — Set default project and zone in Cloud Shell

```bash
gcloud config set project "$PROJECT_ID"
gcloud config set compute/region asia-south1
gcloud config set compute/zone asia-south1-a
gcloud config configurations list
```

**Configurations:** Advanced users can create named configurations (`gcloud config configurations create`) for multiple projects. For this lab, the `default` configuration is enough.

---

### Step 8 — Cloud Shell security habits

Follow these rules during training and production work:

- Do not store **service account keys** in `$HOME`.
- Do not commit secrets to files in Cloud Shell; the home directory persists.
- Log out of shared computers; Cloud Shell ties to your Google user session.
- Prefer **impersonation** and **attached service accounts** over downloadable keys (covered in Part 6).

---

## Part 3 — Install and configure the Google Cloud CLI

**Time estimate:** ~20 minutes (skip if you only use Cloud Shell)

Install the Cloud SDK on your **local** machine so you can administer GCP from your laptop CI/CD-style.

### Step 9 — Install the Cloud SDK

Choose your operating system:

**Windows (recommended installer)**

1. Download the installer from [Install the gcloud CLI](https://cloud.google.com/sdk/docs/install).
2. Run the installer and include `gcloud` on your PATH.
3. Open a **new** PowerShell or Command Prompt window.

**macOS (Homebrew example)**

```bash
brew install --cask google-cloud-sdk
```

**Linux (Debian/Ubuntu example)**

Follow Google’s apt/yum instructions in the official install guide.

Verify:

```bash
gcloud version
```

---

### Step 10 — Initialize and authenticate as your user

```bash
gcloud init
```

During the wizard:

1. Sign in with your **training Google account** when the browser opens.
2. Select the **training project** when prompted.
3. Optionally set a default Compute zone (e.g. `asia-south1-a`).

If you already initialized before, refresh login:

```bash
gcloud auth login
gcloud config set project "$PROJECT_ID"
gcloud auth list
```

**Expected result:** The active account shows your user with a `*` marker.

---

### Step 11 — Application Default Credentials (ADC)

ADC lets client libraries and tools pick up credentials from your environment.

```bash
gcloud auth application-default login
```

**When to use ADC:** Local development with client libraries, Terraform with user creds, or tools that call Google APIs. In production, workloads should use **attached service accounts** or **Workload Identity Federation**, not personal user ADC.

Verify ADC file location (do not share this file):

```bash
# Linux/macOS
ls -l ~/.config/gcloud/application_default_credentials.json

# Windows PowerShell
Get-Item "$env:APPDATA\gcloud\application_default_credentials.json"
```

---

### Step 12 — Useful CLI configuration commands

```bash
gcloud config get-value project
gcloud projects describe "$PROJECT_ID" --format="yaml(projectId,name,projectNumber)"
gcloud info --format="value(config.paths.global_config_dir)"
```

Create a second named configuration (optional):

```bash
gcloud config configurations create cloudmart-lab
gcloud config set project "$PROJECT_ID"
gcloud config configurations activate cloudmart-lab
```

---

## Part 4 — IAM concepts in the Console

**Time estimate:** ~15 minutes

### Step 13 — Open the IAM page

1. Go to **IAM & Admin → IAM**.
2. Ensure the scope selector at the top shows your **project** (not organization) unless your instructor directs otherwise.
3. Review the **Permissions** tab (principal list).

| Column | Meaning |
| --- | --- |
| **Principal** | Who: user, group, service account, domain |
| **Role** | Bundle of permissions granted on this resource |
| **Security insights** | Recommendations (if enabled) |

---

### Step 14 — Understand roles and permissions

Open **IAM & Admin → Roles**. Filter **Google-managed** roles.

| Role type | Description | Example |
| --- | --- | --- |
| **Basic roles** | Legacy broad roles | Owner, Editor, Viewer |
| **Predefined roles** | Curated least-privilege bundles | `roles/compute.viewer` |
| **Custom roles** | Org/project-specific permission sets | `cloudmartLogViewer` |

Select **roles/viewer** and review included permissions.

**Rule of thumb:** Avoid **Owner** and **Editor** for daily work. Prefer **predefined** roles scoped to job function.

---

### Step 15 — Read the project IAM policy (CLI)

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --format=json | head -c 2000
```

Identify:

- `bindings[]` — pairs of `role` and `members[]`
- Your user’s current roles

Optional formatted table:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --format="table(bindings.role,bindings.members)"
```

**Concept:** An **allow policy** on a resource lists role grants. Deny policies and organization constraints are advanced topics; know they exist at org/folder level.

---

## Part 5 — Users, groups, roles, and permissions

**Time estimate:** ~25 minutes

You will grant a **developer** principal read-oriented access without Project Editor.

### Step 16 — Choose the member to grant access

Pick **one** option:

| Option | Member format | Notes |
| --- | --- | --- |
| **A — Teammate** | `user:colleague@example.com` | Second learner in the same project |
| **B — Google Group** | `group:cloudmart-developers@example.com` | Preferred at scale |
| **C — Yourself (sandbox)** | `user:YOUR_EMAIL` | Only if no second principal; use a **narrower** test role and remove it in cleanup |

Set a variable for the member (Cloud Shell/bash):

```bash
export DEV_MEMBER="user:developer@example.com"
# OR: export DEV_MEMBER="group:cloudmart-developers@example.com"
```

---

### Step 17 — Grant a predefined role (Console)

1. **IAM & Admin → IAM → Grant access**.
2. **New principals:** enter the email (user or group).
3. **Role:** select **Viewer** (`roles/viewer`).
4. Click **Save**.

**Why Viewer?** Developers can inspect resources and troubleshoot without creating or deleting infrastructure.

---

### Step 18 — Grant additional least-privilege roles

Still on **Grant access** (or **Edit principal**), add roles as needed:

| Role | ID | Purpose |
| --- | --- | --- |
| Logs Viewer | `roles/logging.viewer` | Read Cloud Logging for debugging |
| Compute Viewer | `roles/compute.viewer` | List VMs and disks (read-only) |

Avoid stacking **Editor** or **Owner** on the developer principal.

**CLI equivalent (Viewer only):**

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="roles/viewer"
```

**Logging viewer:**

```bash
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="roles/logging.viewer"
```

Each `add-iam-policy-binding` call is **additive**; it does not remove other bindings.

---

### Step 19 — Verify effective access

**If you have a second user:** sign in as that user and confirm:

- They can open **Compute Engine → VM instances** (read).
- They **cannot** create a VM (no create permission).

**If you only have one user:** verify bindings appear on the IAM page and in:

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:$DEV_MEMBER" \
  --format="table(bindings.role)"
```

---

### Step 20 — Optional: create a custom role (project level)

Skip this step if your organization disallows custom roles.

1. **IAM & Admin → Roles → Create role**.
2. **Title:** `CloudMart Log Viewer (Custom)`.
3. **ID:** `cloudmartLogViewer`.
4. Add permissions, for example:
   - `logging.logEntries.list`
   - `logging.logs.list`
5. Create the role.
6. Grant it to `$DEV_MEMBER` on the project.

CLI sketch (permissions must match your selection):

```bash
gcloud iam roles create cloudmartLogViewer --project="$PROJECT_ID" \
  --title="CloudMart Log Viewer" \
  --permissions=logging.logEntries.list,logging.logs.list

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="projects/$PROJECT_ID/roles/cloudmartLogViewer"
```

**What you learned:** Custom roles fine-tune permissions when no single predefined role fits.

---

### Step 21 — Remove a role (practice revocation)

Remove **only** the Viewer binding you added (Console: IAM → principal → pencil icon → remove **Viewer** → Save).

CLI:

```bash
gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="roles/viewer"
```

Re-add Viewer if your instructor wants the binding to remain until Part 8 cleanup.

---

## Part 6 — Service accounts and authentication

**Time estimate:** ~25 minutes

### Step 22 — Create the application service account

1. Go to **IAM & Admin → Service Accounts**.
2. Click **Create service account**.
3. **Name:** `CloudMart Application`
4. **ID:** `cloudmart-app-sa`
5. **Description:** Runtime identity for CloudMart workloads
6. Click **Create and continue**.

Grant **project-level** roles (adjust per instructor policy):

| Role | ID |
| --- | --- |
| Logs Writer | `roles/logging.logWriter` |
| Monitoring Metric Writer | `roles/monitoring.metricWriter` |

Click **Done**.

CLI:

```bash
gcloud iam service-accounts create cloudmart-app-sa \
  --project="$PROJECT_ID" \
  --display-name="CloudMart Application"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/monitoring.metricWriter"
```

---

### Step 23 — Create a CI/CD service account (optional)

Repeat for automation with **narrower** scope:

| Field | Value |
| --- | --- |
| Name | `CloudMart CI/CD` |
| ID | `cloudmart-cicd-sa` |

Example roles (choose one pattern your instructor approves):

- **Artifact Registry only:** `roles/artifactregistry.writer` on a specific repo (resource-level IAM), **or**
- **Compute limited:** custom role with `compute.instances.list` only for read-only pipeline validation

Do **not** grant `roles/owner` to CI/CD identities.

---

### Step 24 — Service account keys (what not to do)

1. Open `cloudmart-app-sa` → **Keys** tab.
2. Read the warning about key security.

**Lab rule:** Do **not** create a JSON key unless your instructor explicitly requires it for a legacy integration exercise.

Preferred patterns:

| Pattern | Use when |
| --- | --- |
| **Attached service account** on Compute Engine, GKE, Cloud Run | Workload runs on Google Cloud |
| **Workload Identity Federation** | External CI (GitHub, GitLab) without keys |
| **Short-lived impersonation** | Admin automation from user or another SA |

---

### Step 25 — Impersonate a service account (user → SA)

Your user needs permission to impersonate:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  "cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --member="user:YOUR_EMAIL" \
  --role="roles/iam.serviceAccountTokenCreator"
```

Impersonate and print an access token (proves identity without a key file):

```bash
gcloud auth print-access-token \
  --impersonate-service-account="cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com"
```

Test a read-only API call as the service account:

```bash
gcloud projects describe "$PROJECT_ID" \
  --impersonate-service-account="cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com"
```

If this fails with permission denied, the SA may lack `resourcemanager.projects.get` — that is expected. Try a logging API list instead or accept that impersonation token generation alone is sufficient proof.

**Revoke impersonation grant** in Part 8 if you added `serviceAccountTokenCreator` only for this exercise.

---

### Step 26 — Grant developers permission to use a service account

Developers deploying workloads may need `iam.serviceAccountUser` on a specific SA:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  "cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --member="$DEV_MEMBER" \
  --role="roles/iam.serviceAccountUser"
```

**Meaning:** The member can attach or act as this SA where their other roles already allow deployment (principle of least privilege).

---

## Part 7 — IAM security best practices

**Time estimate:** ~15 minutes

### Step 27 — Audit overly broad bindings

On **IAM & Admin → IAM**, filter or scan for:

- Principals with **Owner** or **Editor** (`roles/owner`, `roles/editor`)
- Service accounts with **Owner/Editor**
- Unused service accounts (no keys, no recent activity)

Document one finding in `~/cloudmart-labs/module2-notes.txt` (Cloud Shell) or your local notes.

Optional: open **IAM & Admin → Policy Troubleshooter** or **Recommender** (if enabled) and review a suggestion.

---

### Step 28 — Apply the least-privilege checklist

For CloudMart, confirm or plan:

| Practice | Action in this lab |
| --- | --- |
| **Least privilege** | Developer gets Viewer + logging.viewer, not Editor |
| **Separation of duties** | Separate `cloudmart-app-sa` and human admins |
| **No key sprawl** | No JSON keys created |
| **Use groups** | Bind `group:...` when available |
| **Resource-level IAM** | Prefer repo/bucket-level roles for CI/CD when possible |
| **Auditability** | IAM changes appear in Admin Activity logs |
| **Regular review** | Remove Part 5–6 grants in Part 8 |

---

### Step 29 — Simulate excessive permission (thought exercise)

**Do not run in production.**

If your instructor allows a sandbox binding, temporarily add **Editor** to a test SA, list what it can do in the Console, then remove it immediately.

```bash
# Only if instructed — remove right after
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:cloudmart-cicd-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/editor"

gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:cloudmart-cicd-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/editor"
```

Discuss with your class: why **Editor** on automation identities is high risk.

---

### Step 30 — Troubleshooting IAM

| Symptom | Likely cause | What to try |
| --- | --- | --- |
| `Permission denied` on API call | Missing role on project or resource | IAM page → verify principal roles |
| User cannot see project | Not granted any role | Add at least Viewer |
| SA cannot write logs | Missing `logging.logWriter` | Add role to SA |
| Impersonation fails | Missing `serviceAccountTokenCreator` | Fix SA IAM policy on your user |
| `gcloud` uses wrong account | Multiple creds | `gcloud auth list` and `gcloud config set account` |
| Group binding has no effect | User not in group | Cloud Identity / Google Groups membership |

---

## Part 8 — Clean up lab resources

**Time estimate:** ~10 minutes

Remove lab-specific IAM bindings and service accounts. **Do not** remove your own Owner/IAM Admin access needed for other labs.

### Step 31 — Remove project IAM bindings for lab principals

```bash
gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="roles/logging.viewer" 2>/dev/null || true

gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="roles/viewer" 2>/dev/null || true
```

Remove custom role binding and delete custom role if created:

```bash
gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
  --member="$DEV_MEMBER" \
  --role="projects/$PROJECT_ID/roles/cloudmartLogViewer" 2>/dev/null || true

gcloud iam roles delete cloudmartLogViewer --project="$PROJECT_ID" 2>/dev/null || true
```

---

### Step 32 — Delete lab service accounts

```bash
gcloud iam service-accounts delete \
  "cloudmart-app-sa@$PROJECT_ID.iam.gserviceaccount.com" --quiet

gcloud iam service-accounts delete \
  "cloudmart-cicd-sa@$PROJECT_ID.iam.gserviceaccount.com" --quiet 2>/dev/null || true
```

Console: **IAM & Admin → Service Accounts** → select account → **Delete**.

---

### Step 33 — Verify cleanup

```bash
gcloud iam service-accounts list --project="$PROJECT_ID" \
  --filter="email:cloudmart"

gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:cloudmart" \
  --format="table(bindings.role,bindings.members)"
```

No `cloudmart-*` service accounts or unexpected bindings should remain.

---

## Validation checklist

- [ ] Selected the correct project and recorded Project ID
- [ ] Used Console search and opened IAM
- [ ] Ran commands successfully in Cloud Shell
- [ ] Installed or verified local `gcloud` (or documented Cloud Shell-only path)
- [ ] Explained principal, role, and allow policy
- [ ] Granted least-privilege roles to a user or group
- [ ] Created `cloudmart-app-sa` with log/metric writer roles
- [ ] Avoided creating service account JSON keys
- [ ] Tested service account impersonation (or documented why it was skipped)
- [ ] Reviewed IAM page for Owner/Editor on service accounts
- [ ] Removed lab bindings and deleted lab service accounts

---

## Evidence to submit

| # | Screenshot or output |
| --- | --- |
| 1 | IAM page showing project principals |
| 2 | Service account list with `cloudmart-app-sa` (before cleanup) |
| 3 | Cloud Shell output of `gcloud config list` |
| 4 | Local terminal output of `gcloud auth list` (redact if needed) |
| 5 | Output of `gcloud projects get-iam-policy` snippet showing your developer binding |
| 6 | VM metadata or impersonation command success (before cleanup) |
| 7 | IAM page or CLI proof after cleanup (no lab SAs) |

---

## Knowledge check

1. What is a **principal** in IAM?  
   - A) A GCP region  
   - B) An identity such as a user, group, or service account  
   - C) A VPC firewall rule  

2. Which role type should you prefer for daily operations?  
   - A) Basic Owner  
   - B) Predefined least-privilege roles  
   - C) Unrestricted custom role with all permissions  

3. Where is `gcloud` preinstalled and authenticated with your user?  
   - A) Cloud Shell  
   - B) Every GCE image by default  
   - C) Cloud Storage buckets  

4. What is the recommended way for a GCE workload to call Google APIs?  
   - A) Downloaded JSON service account key on disk  
   - B) Attached service account with appropriate IAM roles  
   - C) Shared Owner user password  

5. Which command adds a member to a role on a project?  
   - A) `gcloud projects add-iam-policy-binding`  
   - B) `gcloud compute instances create`  
   - C) `gcloud services enable`  

<details>
<summary><strong>Answer key</strong></summary>

1. **B** — Principals are identities.  
2. **B** — Predefined roles follow least privilege.  
3. **A** — Cloud Shell.  
4. **B** — Use attached service accounts, not keys.  
5. **A** — Adds an IAM binding on the project.

</details>

---

## Optional challenge

1. Create a **resource-level** IAM binding: grant `roles/storage.objectViewer` on a single bucket to `cloudmart-app-sa` (create a bucket `cloudmart-iam-lab-$PROJECT_ID` if needed).  
2. Enable **IAM Deny** or organization policy discussion with your instructor (read-only demo).  
3. Configure **Workload Identity Federation** documentation review: summarize how GitHub Actions could deploy without long-lived keys.

---

## Official references

- [Google Cloud Console overview](https://cloud.google.com/docs/cloud-console)
- [Google Cloud Shell documentation](https://cloud.google.com/shell/docs)
- [Install the gcloud CLI](https://cloud.google.com/sdk/docs/install)
- [Initialize the gcloud CLI](https://cloud.google.com/sdk/docs/initializing)
- [IAM overview](https://cloud.google.com/iam/docs/overview)
- [Understanding roles](https://cloud.google.com/iam/docs/understanding-roles)
- [Granting, changing, and revoking access](https://cloud.google.com/iam/docs/granting-changing-revoking-access)
- [Service accounts](https://cloud.google.com/iam/docs/service-accounts)
- [Best practices for using service accounts](https://cloud.google.com/iam/docs/best-practices-service-accounts)
- [Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)

---

## Lab outcome

You can navigate the Console, work efficiently in Cloud Shell, configure the Cloud SDK, interpret IAM policies, grant appropriate access to users and groups, create and use service accounts securely, and apply core IAM best practices for CloudMart.

You can take other training labs in any order; none are required before or after this one.