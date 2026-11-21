# GCP Training — CloudMart Labs and Presentations

Hands-on **labs** and **slide decks** for Google Cloud Platform training. Scenarios use a fictional e-commerce company, **CloudMart**, for realistic context.

**Important:** Every lab in `Labs/` is **standalone**. You can complete them in any order. Each lab creates its own resources and includes a **cleanup** section.

---

## Repository layout

| Path | Contents |
| --- | --- |
| [`Labs/`](Labs/) | Learner-facing lab guides (Markdown) |
| [`Presentation/`](Presentation/) | Instructor PDF slide decks |

Default GCP region used in many labs: **`asia-south1` (Mumbai)** unless noted otherwise.

---

## Presentations

| # | File | Topic |
| --- | --- | --- |
| 09 | [09 - Docker and Containerization Fundamentals…](Presentation/09%20-%20Docker-and-Containerization-Fundamentals-Architecture-Dockerfile-and-Image-Management.pdf) | Docker architecture, Dockerfile, image management |
| 10 | [10 - Introduction to Kubernetes and Container Orchestration](Presentation/10%20-%20Introduction-to-Kubernetes-and-Container-Orchestration.pdf) | Kubernetes concepts |
| 11 | [11 - Google Kubernetes Engine (GKE)…](Presentation/11%20-%20Google-Kubernetes-Engine-GKE-Architecture-Deployment-Operations-and-Troubleshooting.pdf) | GKE deployment and operations |

> Additional presentation PDFs can be added under `Presentation/` with the same numbering pattern.

**Suggested flow for containers / orchestration:** Presentation **09** → **Lab 09** → Presentation **10** → **Lab 10** → Presentation **11** → **Lab 11**.

---

## Lab catalog

| Lab | Title | Module focus | Time | Guide |
| --- | --- | --- | --- | --- |
| 01 | Deploy and Manage a Virtual Machine Using Google Compute Engine | Compute Engine | 90–120 min | [Open lab](Labs/Lab%2001%20-%20Deploy%20and%20Manage%20a%20Virtual%20Machine%20Using%20Google%20Compute%20Engine.md) |
| 02 | Google Cloud Console, CLI, and Identity and Access Management | Console, `gcloud`, IAM | 90–120 min | [Open lab](Labs/Lab%2002%20-%20Google%20Cloud%20Console,%20CLI,%20and%20Identity%20and%20Access%20Management.md) |
| 03 | Google Cloud VPC Networking, Firewall Rules, NAT, and Load Balancing | VPC, NAT, load balancing | 120–150 min | [Open lab](Labs/Lab%2003%20-%20Google%20Cloud%20VPC%20Networking,%20Firewall%20Rules,%20NAT,%20and%20Load%20Balancing.md) |
| 04 | Google Cloud Storage Object Storage and Data Lifecycle | Cloud Storage | 90–120 min | [Open lab](Labs/Lab%2004%20-%20Google%20Cloud%20Storage%20Object%20Storage%20and%20Data%20Lifecycle.md) |
| 05 | Cloud SQL Managed MySQL for CloudMart | Cloud SQL | 60–90 min | [Open lab](Labs/Lab%2005%20-%20Cloud%20SQL%20Managed%20MySQL%20for%20CloudMart.md) |
| 06 | Cloud Run Serverless API for CloudMart | Cloud Run | 60–90 min | [Open lab](Labs/Lab%2006%20-%20Cloud%20Run%20Serverless%20API%20for%20CloudMart.md) |
| 07 | BigQuery Analytics for CloudMart | BigQuery | 60–90 min | [Open lab](Labs/Lab%2007%20-%20BigQuery%20Analytics%20for%20CloudMart.md) |
| 08 | Cloud Bigtable Real-Time Events for CloudMart | Cloud Bigtable | 60–90 min | [Open lab](Labs/Lab%2008%20-%20Cloud%20Bigtable%20Real-Time%20Events%20for%20CloudMart.md) |
| 09 | Docker Containerization and Image Management for CloudMart | Docker, Artifact Registry | 75–90 min | [Open lab](Labs/Lab%2009%20-%20Docker%20Containerization%20and%20Image%20Management%20for%20CloudMart.md) |
| 10 | Kubernetes Orchestration for CloudMart | `kubectl`, GKE Autopilot | 90–120 min | [Open lab](Labs/Lab%2010%20-%20Kubernetes%20Orchestration%20for%20CloudMart.md) |
| 11 | GKE Operations and Troubleshooting for CloudMart | GKE Standard, HPA, rollouts | 90–120 min | [Open lab](Labs/Lab%2011%20-%20GKE%20Operations%20and%20Troubleshooting%20for%20CloudMart.md) |

---

## Standard lab structure

Each lab guide follows the same learner-facing pattern:

1. Metadata table (module, level, time, services)  
2. Table of contents  
3. Lab description and **standalone** note  
4. Scenario and use case (CloudMart)  
5. Learning objectives  
6. What you will build / architecture  
7. Prerequisites checklist  
8. Numbered **Parts** with detailed steps  
9. Validation checklist and evidence to submit  
10. Knowledge check with answer key  
11. Optional challenge  
12. Official Google references  
13. Lab outcome and cleanup  

---

## Learner prerequisites (all labs)

- Google account and a **training project** with **billing enabled** (unless your instructor provides a shared sandbox)  
- Permission to create resources for that lab’s services  
- **Google Cloud Console** and **Cloud Shell** access  
- For local `gcloud` / Docker (where noted): installed tools and network access  

Replace `YOUR_PROJECT_ID` in lab commands with your training project ID.

---

## Cost and cleanup

Many labs create billable resources (VMs, SQL instances, Bigtable, load balancers, registries). **Always complete the cleanup part** at the end of each lab unless your instructor asks you to keep resources.

| Lab | Critical cleanup action |
| --- | --- |
| 01, 03 | Delete VMs, firewalls, VPC (Lab 03), load balancer components |
| 04 | Delete buckets and remove public IAM if used |
| 05 | **Delete Cloud SQL instance** |
| 06 | **Delete Cloud Run services** |
| 07 | **Delete BigQuery dataset** |
| 08 | **Delete Bigtable instance** |
| 09 | Stop containers; delete images; optional delete Artifact Registry repo |
| 10 | Delete namespace; **delete GKE cluster** |
| 11 | Delete namespace; **delete GKE Standard cluster** |

---

## For instructors

- Labs are **independent** — reorder by module or skill path without prerequisite chains.  
- Pair **Presentation 09** with **Lab 09** (Docker); **10** with **Lab 10** (Kubernetes intro); **11** with **Lab 11** (GKE ops).  
- **Lab 06** (Cloud Run) vs **Lab 09** (Docker CLI) vs **Lab 10** (orchestration)—complementary, not sequential.  
- Confirm learners have required IAM roles before IAM, networking, or data labs.  
- Enforce cleanup in shared projects to control cost.  

---

## Contributing

When adding a new lab:

- Name file: `Lab NN - Descriptive Title.md`  
- Keep the **standalone** note and a full **cleanup** section  
- Avoid dependencies on other lab resource names  

---

## License and materials

Training content for classroom use. Google Cloud product names and documentation links refer to [Google Cloud Documentation](https://cloud.google.com/docs).