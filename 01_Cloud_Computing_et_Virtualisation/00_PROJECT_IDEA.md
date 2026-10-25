# SkyForge: from a bare VM to an auto-scaling cloud service, all as code

**Field:** Cloud Computing & Virtualisation · **Duration:** 5 days (~25–30 h) · **Level at the end:** junior cloud / platform engineer

---

## 1. The story (project idea)

A small startup wants to serve a simple **AI inference API** (for example a sentiment or image classifier, or even a dummy model, since the model itself is not the point here). They hire me to do **the whole infrastructure journey**:

1. Understand virtualisation by building and networking **virtual machines** myself.
2. Package the app as a **container** following production best practices.
3. Run it on **Kubernetes** locally with self-healing and auto-scaling.
4. Provision the cloud environment with **Infrastructure-as-Code (Terraform / OpenTofu)**: network, identity, storage, and a serverless container service. That's either **Azure for Students** (no card) or a **local cloud on my PC** (see the zero-cost rule in `01_MILESTONES.md`).
5. Add **observability, security, and cost control**, then **destroy everything** with one command.

I go from the "metal" (hypervisor) up to managed cloud services, which is the exact stack modern AI products run on.

---

## 2. Why this project covers the field

| Course concept | Where I practise it |
|----------------|------------------------|
| Hypervisors (Type 1 vs Type 2), full vs para-virtualisation, hardware-assisted virtualisation (VT-x/AMD-V) | M1: build VMs with Hyper-V or VirtualBox |
| Virtual networking (NAT, bridged, host-only), virtual disks, snapshots | M1 |
| OS-level virtualisation: containers, namespaces, cgroups, union filesystems | M2 |
| Orchestration, desired state, self-healing, scaling | M3: Kubernetes |
| Service models IaaS / PaaS / CaaS / FaaS / SaaS, deployment models (public, private, hybrid) | M4: I use IaaS (a VM) **and** CaaS/PaaS (a managed container service) and compare them |
| Regions, availability zones, VPC/VNet, subnets, security groups, IAM | M4 |
| Elasticity, high availability, the shared responsibility model | M3–M5 |
| Monitoring, cost (FinOps), the Well-Architected Framework | M5 |

---

## 3. Stack (current as of October 2026)

> **The whole project costs $0 and needs no card.** For M4–M5 I use **Track A: Azure for Students** ($100 credit, verified with my school email, no card, so I can never be charged) or **Track B: a local cloud** on my PC (kind + Knative + S3-compatible storage + Terraform). The concepts transfer one-to-one to AWS and GCP. Every tool below is free.

| Layer | Tool (preferred) | Alternatives | Why this one |
|-------|------------------|--------------|--------------|
| Local hypervisor | **Hyper-V** (built into Windows Pro/Enterprise) or **VirtualBox** (works on Home editions) | VMware Workstation (free for personal use) | Windows 11 **Home** has no Hyper-V Manager, so use VirtualBox |
| Guest OS | **Ubuntu Server 24.04 LTS** | Debian 12/13 | Most cloud VM images are Ubuntu LTS |
| VM automation (stretch) | **Vagrant** or **cloud-init** | Packer | cloud-init is how *real* cloud VMs are configured at boot |
| Containers | **Docker** (Desktop + WSL2) | Podman, Rancher Desktop | Industry default |
| Local Kubernetes | **kind** or **k3d** | minikube | Light and fast, also used in CI pipelines |
| K8s packaging | **Helm** (stretch: Kustomize) | — | How companies ship K8s apps |
| IaC | **Terraform** or **OpenTofu** (the open-source fork, same language) | Pulumi, Bicep, CloudFormation | Multi-cloud, number one on job postings |
| Serverless container service | Track A: Azure **Container Apps** (consumption plan, monthly free grant) · Track B: **Knative Serving** on kind | Cloud Run / ECS Fargate (need a card) | Scale-to-zero containers; Cloud Run is built on the Knative API |
| Object storage | Track A: Azure Blob Storage · Track B: **Garage** / **SeaweedFS** (S3-compatible) | LocalStack S3 | — |
| Terraform state backend | Track A: Azure Storage · Track B: **Postgres** (`pg` backend) | — | Remote state with locking, for free |
| Registry | **GitHub Container Registry (GHCR)**, free | Docker Hub | Avoid ACR/ECR: they bill daily |
| Public URL (Track B) | **Cloudflare quick tunnel** (`cloudflared`), free, no account | — | — |
| Observability | **Prometheus + Grafana** (local), **Azure Monitor** (Track A, free tier) | OpenTelemetry + Jaeger | — |
| Cost (FinOps) | **Infracost** (free CLI, cost estimates) + **OpenCost** (Track B) | Azure Cost Management (Track A) | Learn cost control without spending anything |
| Security scanning | **Trivy** (images + IaC), **Checkov** or **tfsec/Trivy** for Terraform | Docker Scout | — |

---

## 4. Target architecture (draw this myself first!)

```
 [ My laptop ]
   ├── VM-1 (Ubuntu) ── VM-2 (Ubuntu)      ← M1: private network between VMs
   ├── Docker image: inference-api         ← M2
   └── kind cluster                        ← M3
         ├── Deployment (3 replicas) + HPA
         ├── Service + Ingress
         └── ConfigMap / Secret

 [ Cloud — provisioned ONLY by Terraform ]  ← M4   (Track A: Azure for Students | Track B: local)
   ├── Resource group / namespace + tags/labels
   ├── VNet + NSGs        | Docker networks + K8s NetworkPolicies
   ├── Registry: GHCR (free, both tracks)
   ├── Container Apps     | Knative Serving   (scale 0 → N)  ←  inference-api
   ├── Blob Storage       | Garage/SeaweedFS (S3)  (model artefacts)
   ├── Managed identity   | ServiceAccount + RBAC + read-only key (least privilege)
   └── Azure Monitor + budget alert | Grafana + alerts + OpenCost   ← M5
```

---

## 5. Final deliverables

1. A Git repo `skyforge/` containing: `docs/` (diagrams + ADRs), `vm/` (VM notes, cloud-init file), `app/` (minimal API + Dockerfile), `k8s/` (manifests or Helm chart), `infra/` (Terraform modules + environments), `README.md`.
2. A **comparison report** (1–2 pages): VM vs container vs serverless container, covering startup time, density, isolation, cost, and operational effort, **with numbers I measured myself**.
3. A **load test** result showing auto-scaling in action (a screenshot or graph).
4. Proof that `terraform destroy` removes everything, **$0 spent, no card used**, plus an **Infracost estimate** of what SkyForge would cost a company per month.

---

## 6. If I skipped a previous project

This is project 01, so there are no prerequisites. I only need basic Python (to write a tiny API) and the willingness to live in a Linux terminal.

## 7. Prerequisite skills to refresh (Day 0, 2–3 h)
- Linux CLI: files, permissions, users, `systemd`, package managers, `ssh`
- Networking basics: IP, CIDR, subnet masks, DNS, ports, TCP vs UDP, HTTP, NAT
- Search: "Linux command line for beginners", "CIDR notation explained", "how DNS works"
