# SkyForge: milestones

> Format of every milestone: **Goal → Why now → Concepts to learn → What to build (Core / Stretch) → Best practices → Definition of Done → Pitfalls**.
> Total: 6 milestones over about 5 days.

---

## 💸 Zero-cost rule: this whole project costs $0 and needs no card

I have no bank account and no money, so **every step below is free and needs no credit card**. M0–M3 run 100% on my PC. For the cloud milestones (M4–M5) I pick **one of two tracks** at the end of M0:

| Track | What it is | Needs | When I choose it |
|-------|-----------|-------|------------------|
| **A: Real cloud, no card** | **Azure for Students**: $100 credit, verified with my **school email**, **no card**. When the credit runs out, resources just stop and I can never be charged. | A school email (or student ID) that passes verification | If verification works (it can take hours to a few days, so I apply on Day 1) |
| **B: 100% local cloud** | I rebuild the cloud on my PC: kind + Knative (≈ Cloud Run), Terraform, S3-compatible storage, Postgres for Terraform state, Prometheus/Grafana | Nothing, just my PC (i9, 32 GB RAM) | If student verification fails, or if I want to stay fully offline |

**Track B teaches about 90% of the same skills** (Terraform, state, modules, networking, least privilege, scale-to-zero, observability). Track A adds the experience of a real cloud provider. Either track completes the project.

**My hard rules**
- ❌ Never enter a card anywhere. If a site asks for one, it isn't my path.
- ❌ On Azure, **never click "Upgrade" / switch to Pay-As-You-Go** when the credit ends.
- ❌ Avoid paid-by-the-hour resources even on credit: NAT Gateway, Azure Firewall, Application Gateway, Front Door, AKS, Azure Container Registry (it bills daily, so I use the free **GHCR** instead).
- ✅ Free tools only: open source, free tiers that need no card, or student offers.
- ✅ Always `terraform destroy` when I'm done for the day on Track A.

**Other free student resources to apply for on Day 1:**
- **GitHub Student Developer Pack** (education.github.com/pack): free with my school email, and it includes the Azure for Students offer.
- **AWS Academy Learner Lab**: an AWS sandbox with no card **if my school is a member**. I'll ask my *Cloud Computing et Virtualisation* teacher. If I get it, I can use it like Track A with AWS services.

---

## M0: Foundations and setup (Day 1 morning, ~3 h)

**Goal:** a working local toolbox, a mental model of "what the cloud actually is", and my **track choice (A or B)** for M4–M5.

**Why now:** every later milestone fails if my tools are broken or I don't understand the vocabulary. Engineers always start by setting up the environment and the glossary. Student verification takes time, so I start it now.

**Concepts to learn**
- NIST definition of cloud computing (5 characteristics, 3 service models, 4 deployment models)
- IaaS vs PaaS vs CaaS vs FaaS vs SaaS, with one real product as an example for each
- The shared responsibility model (who patches the OS in each model?)
- Regions vs Availability Zones vs edge locations
- How cloud billing works (pay-as-you-go, free tiers, credits, budgets), so I understand what I'm avoiding

**What to build**
- Core: install the free toolbox: **WSL2 + Ubuntu, Docker Desktop, VirtualBox 7.2, kubectl, kind, Helm, Terraform (or OpenTofu)**. All of them are free. Create the Git structure from the project idea inside this folder.
- Core: write `docs/glossary.md` with the concepts above **in my own words**.
- Core: apply for **Azure for Students** and the **GitHub Student Developer Pack** with my school email, and ask my teacher about **AWS Academy**.
- Core: write `docs/track.md` saying which track I chose (A or B) and why. If verification is still pending, start with B and switch later. The M4 design is the same for both.
- Core (Track A only): once the Azure account is active, install the **Azure CLI**, then set a **budget alert at $10** of my $100 credit in *Cost Management → Budgets*, before I create anything. With no card I can't be charged, but alerts-before-resources is a professional habit. If budgets aren't available on my offer type, I'll note where to check my remaining credit in the portal (the Education hub) and check it daily.
- Stretch (Track A): enable **MFA** on my account, and practise least privilege by creating a resource group and giving a second identity only the *Contributor* role on it.

**Best practices:** no card on any account; MFA on the cloud account; budget alerts before resources; never use the admin identity for daily work.

**Definition of Done**
- [ ] `docker run hello-world` works, `kind create cluster` works, `terraform version` works
- [ ] Applications sent (Azure for Students, GitHub Student Pack) and my teacher asked about AWS Academy
- [ ] `docs/track.md` written (A or B)
- [ ] Track A only: budget alert configured (or a daily credit check noted)
- [ ] Glossary written

**Pitfalls:** Windows 11 **Home** has no Hyper-V Manager, so use VirtualBox for M1. VirtualBox runs on top of the Windows hypervisor (the green turtle 🐢 icon) when WSL2 is on, so it's slower but works. Don't disable WSL2, because Docker needs it. Don't sign up for the *normal* Azure, AWS, or GCP free trials: they all ask for a card.

---

## M1: Virtualisation by hand (Day 1 afternoon + Day 2 morning, ~6 h) · **$0**

**Goal:** create, network, snapshot, and measure virtual machines, so that I *feel* what a hypervisor does.

**Why now:** containers and cloud VMs are abstractions **on top of** virtualisation. If I skip this, "EC2 instance" or "container" stays a magic word. The cloud is just someone else's hypervisor with an API in front of it.

**Concepts to learn**
- Type 1 (bare-metal: ESXi, Hyper-V, KVM, Xen) vs Type 2 (hosted: VirtualBox, VMware Workstation)
- Full virtualisation vs para-virtualisation vs hardware-assisted virtualisation (Intel VT-x / AMD-V), and what `virtio` drivers are
- Virtual NICs: NAT vs NAT Network vs bridged vs host-only vs internal network
- Virtual disks: thin (dynamic) vs thick (fixed) provisioning, snapshots, linked clones
- What KVM is and why most clouds (AWS Nitro, GCP) are built on KVM-derived hypervisors

**What to build**
- Core: create **2 Ubuntu Server 24.04 LTS VMs** (free ISO from ubuntu.com), each with 2 vCPU, 2 GB RAM, and a 25 GB dynamic disk. VM-1 is the "web" VM, VM-2 the "db" VM.
- Core: put them on a **private (host-only/internal) network**, give them static IPs, and make them ping each other. Only VM-1 should have internet access through NAT. This is a mini VPC with a public and a private subnet.
- Core: SSH from my laptop into VM-1 using **key-based auth** (disable password login).
- Core: take a snapshot, break something on purpose (delete a system package), then restore the snapshot.
- Core: **measure**: boot time, RAM used at idle, disk size. Write the numbers down; I will compare them with containers in M2.
- Stretch: automate VM creation with **Vagrant** (free) or configure the VM at first boot with a **cloud-init** file (the same mechanism the cloud uses for "user data"). I reuse this in M4.
- Stretch: install a basic firewall (`ufw`) and allow only SSH from my host.

**Best practices:** key-based SSH only, least network exposure, document IPs and ports in `docs/network.md`.

**Definition of Done**
- [ ] Two VMs talk over a private network; only one reaches the internet
- [ ] SSH with keys works and password login is disabled
- [ ] Snapshot restore tested
- [ ] Boot time / RAM / disk measurements recorded
- [ ] I can draw and explain the network diagram from memory

**Pitfalls:** mixing up the NAT and NAT-network modes in VirtualBox; forgetting that two VMs on plain "NAT" mode **cannot** see each other.

---

## M2: Containers, the production way (Day 2 afternoon + Day 3 morning, ~6 h) · **$0**

**Goal:** a small, secure, reproducible container image of a minimal inference API.

**Why now:** I have just felt how heavy VMs are. Containers solve this by **sharing the host kernel**, and I'll measure the difference. Every later milestone (K8s, cloud, the other 5 projects) depends on a good image.

**Concepts to learn**
- Linux **namespaces** (pid, net, mnt, uts, ipc, user) and **cgroups** (resource limits). These two kernel features are what a container actually is.
- Image layers, union filesystems (OverlayFS), layer caching
- OCI image spec, registries, tags vs digests
- Multi-stage builds, distroless/slim base images, running as non-root
- Health checks, graceful shutdown (SIGTERM), the 12-factor app principles (config via env vars, logs to stdout)

**What to build**
- Core: a tiny Python API (FastAPI) with `/health` and `/predict` endpoints. The "model" can be a trivial rule or a tiny scikit-learn model. Keep it simple; infrastructure is the topic here.
- Core: a **multi-stage Dockerfile** using `uv` to install dependencies, a slim base image, a **non-root user**, and a `HEALTHCHECK`. Aim for an image **under 200 MB**.
- Core: a `docker-compose.yml` with the API + a Redis cache (or Postgres) on a user-defined network, the container equivalent of my 2 VMs from M1.
- Core: **measure** container start time, RAM, and image size, and compare them with the M1 VM numbers in a table.
- Core: scan the image with **Trivy** (free, open source) and fix the HIGH/CRITICAL vulnerabilities I can.
- Core: push the image to **GitHub Container Registry (GHCR)**, which is free and needs no card, with a version tag **and** note its digest. I log in with a GitHub personal access token that has only the `write:packages` scope (least privilege) and never commit it.
- Stretch: generate an SBOM (Software Bill of Materials, with Syft or `docker sbom`) and sign the image with **cosign** (free). Look up "supply chain security SLSA".

**Best practices:** pin base image versions, `.dockerignore`, no secrets in the image, one process per container, logs to stdout, immutable tags.

**Definition of Done**
- [ ] Image < 200 MB, runs as non-root, has a health check
- [ ] Compose stack starts with one command
- [ ] VM vs container comparison table with real numbers
- [ ] Trivy report saved; no unfixed CRITICAL vulnerabilities, or the remaining ones are justified
- [ ] Image is in GHCR

**Pitfalls:** using `latest` tags; copying the whole repo (including `.venv`) into the image; running as root; baking `.env` files into images; committing my GitHub token.

---

## M3: Orchestration with Kubernetes (Day 3 afternoon + Day 4 morning, ~6 h) · **$0**

**Goal:** run my container on a local K8s cluster with self-healing, config management, and **auto-scaling under load**.

**Why now:** one container on one machine is not production. Real systems need **desired-state** management: "always keep 3 healthy replicas, scale up when busy". K8s is the industry standard, and its concepts (declarative config, controllers, reconciliation) appear everywhere in cloud work.

**Concepts to learn**
- Control plane vs worker nodes; the API server, etcd, scheduler, controller manager, kubelet
- Pod, ReplicaSet, Deployment, Service (ClusterIP / NodePort / LoadBalancer), Ingress (or the newer **Gateway API**)
- ConfigMap, Secret (and why base64 is **not** encryption)
- Liveness vs readiness vs startup probes
- Resource requests/limits, QoS classes; HorizontalPodAutoscaler and metrics-server
- Rolling updates and rollbacks
- Namespaces, RBAC basics, ServiceAccounts

**What to build**
- Core: a kind cluster with **1 control-plane node + 2 worker nodes**.
- Core: Deployment (3 replicas) with probes and resource requests/limits, plus a Service and an Ingress (ingress-nginx or a Gateway API implementation).
- Core: externalise config into a ConfigMap and a fake API key into a Secret.
- Core: **chaos test**: delete a pod and watch it come back; drain a node and watch the pods reschedule.
- Core: install metrics-server, create an **HPA** (scale 2→6 at 50% CPU), load-test with **k6** or **hey**/**Locust** (all free), and capture a screenshot of the scaling.
- Core: do a rolling update to v2 of my image, then roll back.
- Stretch: package everything as a **Helm chart** with `values-dev.yaml` / `values-prod.yaml`.
- Stretch: install **kube-prometheus-stack** and view pod CPU and memory in Grafana. (Track B needs this in M5, so doing it now saves time.)

**Best practices:** declarative YAML in Git (never `kubectl edit` in production); always set requests and limits; readiness probes are what make zero-downtime deploys possible; one namespace per app or environment.

**Definition of Done**
- [ ] App reachable through the Ingress
- [ ] Self-healing demonstrated (pod delete + node drain)
- [ ] HPA scaling graph or screenshot under load
- [ ] Rolling update + rollback done
- [ ] I can explain what happens, component by component, when I run `kubectl apply`

**Pitfalls:** HPA never scales because metrics-server is missing or no CPU **requests** are set; the image can't be pulled into kind (look up "kind load docker-image"); giving WSL2 too little RAM for a 3-node cluster (I set `memory=8GB` or more in `.wslconfig`).

---

## M4: Cloud with Infrastructure-as-Code (Day 4 afternoon + Day 5 morning, ~7 h) · **$0**

**Goal:** provision a complete, secure environment **only through Terraform/OpenTofu**, and deploy my container to a **serverless, scale-to-zero** container service, on Track A (Azure for Students) or Track B (local cloud).

**Why now:** I have now done networking (M1), packaging (M2), and orchestration (M3) by hand or locally. The cloud lets me rent all of that through APIs. **Clicking in the console is not engineering**: IaC makes infrastructure reviewable, repeatable, and destroyable, and that's how every serious company works.

**Concepts to learn (same for both tracks)**
- Terraform: providers, resources, data sources, variables, outputs, **state** (and why remote state needs locking), modules, `plan` vs `apply`, drift
- Cloud networking: VPC/VNet, subnets (public/private), route tables, internet gateway / NAT gateway (and why it costs money!), security groups / NSGs
- IAM: users vs roles vs managed identities, **least privilege**, why apps use identities and not access keys
- Serverless containers (Cloud Run / Azure Container Apps / ECS Fargate): scale-to-zero, concurrency, cold starts. **Cloud Run is built on the Knative API**, which is why Knative is my local equivalent.
- Object storage (S3 / Blob / GCS), access keys vs identities, bucket policies
- Tagging/labelling strategy (owner, env, project)

**Track mapping: the same architecture with two implementations**

| Concept | Track A: Azure for Students | Track B: local cloud (on my PC) |
|---------|-----------------------------|---------------------------------|
| Terraform state with locking | Azure Storage account + blob container backend (locks with blob leases) | **Postgres in Docker** + Terraform's `pg` backend (real locking) |
| Network + "security groups" | VNet, public/private subnets, **NSGs** (free) | Docker networks (one `internal` = private subnet) + **Kubernetes NetworkPolicies** in kind |
| Container registry | **GHCR** (free; avoid ACR, which bills daily) | **GHCR** or a local `registry:2` container |
| Serverless container service | **Azure Container Apps** (consumption plan, with a monthly free grant), scale 0→3 | **Knative Serving** on kind (installed with the Terraform Helm/Kubernetes providers), scale 0→3 |
| Object storage | Storage account + blob container | **Garage** or **SeaweedFS** (S3-compatible, open source) in Docker; or LocalStack S3 (check its current free terms first) |
| App identity / least privilege | **Managed identity** with the *Storage Blob Data Reader* role (no keys!) | A K8s **ServiceAccount + RBAC**, and a **read-only** S3 key scoped to one bucket (Garage supports per-key bucket permissions) |
| IaaS comparison VM | One small **B1s** Linux VM (in the student free services) with cloud-init, deleted the same day | The **VirtualBox VM from M1**, provisioned by Vagrant + cloud-init |
| Public URL | The Container Apps HTTPS URL | **Cloudflare quick tunnel** (`cloudflared`, free, no account) for a temporary public HTTPS URL |

**What to build**
- Core: a Terraform project with **remote, locked state** (per my track) and a `modules/` folder (`network`, `app`, `storage`, `identity`) plus `envs/dev`. Pin provider versions.
- Core: provision the network (public + private, restrictive rules). Mirror the M1 design and write down the mapping (VirtualBox network ↔ VNet or Docker networks/NetworkPolicies).
- Core: deploy my GHCR image to the **serverless container service** (Container Apps or Knative), scaling from **0 to 3**, and show it scale to zero when idle.
- Core: an object storage bucket for "model artefacts"; the app gets **read-only** access (managed identity on A; ServiceAccount + a read-only scoped key in a K8s Secret on B). Write down why B is weaker than a real managed identity. That gap is the lesson.
- Core: **IaaS comparison**: run the same container on a VM (B1s on A, the M1 VirtualBox VM on B) with cloud-init. Compare effort, startup time, scaling, and (estimated) cost against the serverless service.
- Core: tag/label every resource; run `terraform plan` and save the output in a PR description, the way real teams review infrastructure.
- Core: **cost estimate**: run **Infracost** (free CLI; the free API key needs no card) on the Track A code, or write a pricing-page estimate for Track B, to see what this would cost a company per month. That's FinOps without spending anything.
- Stretch: scan the Terraform code with **Checkov** or **Trivy config** (free); add a GitHub Actions workflow (free for public repos) that runs `fmt`, `validate`, and `plan` on each PR.

**Best practices:** never commit `.tfstate`, `.terraform/`, or credentials; pin provider versions; small reusable modules; `plan` before `apply`, always; everything tagged; prefer identities over static keys.

**Definition of Done**
- [ ] `terraform apply` from zero builds the whole environment (on my track)
- [ ] My API answers over HTTPS (Container Apps URL or Cloudflare tunnel) and scales to zero when idle
- [ ] The app reads from the bucket with read-only, least-privilege access
- [ ] IaaS VM vs serverless comparison written, including the Infracost/pricing estimate
- [ ] My state is remote and locked
- [ ] **$0 spent, no card used**

**Pitfalls:** Track A: creating a NAT Gateway, a load balancer, ACR, or AKS (they eat credit fast), forgetting `terraform destroy`, or forgetting the region. Track B: a 3-node kind cluster + Knative + storage using too much RAM (raise WSL2 memory or use a 1-node cluster for M4); NetworkPolicies not enforced because the CNI doesn't support them (check this: look up "kind network policy").

---

## M5: Operate: observability, security, cost, teardown (Day 5 afternoon, ~4 h) · **$0**

**Goal:** prove I can **run** what I built: see it, secure it, control its (estimated) cost, and clean it up.

**Why now:** "Day 2 operations" is where most real cloud work happens. A system I can't observe or afford isn't production-ready.

**Concepts to learn**
- The three pillars of observability: **metrics, logs, traces**; the four golden signals (latency, traffic, errors, saturation); SLI / SLO / error budget
- Cloud-native monitoring (Azure Monitor / CloudWatch / Cloud Monitoring) vs open source (Prometheus, Grafana, Loki)
- The Well-Architected Framework pillars (operational excellence, security, reliability, performance, cost, sustainability)
- FinOps basics: cost allocation by tags, right-sizing, unit cost (cost per 1,000 requests)

**What to build**
| Task | Track A: Azure for Students | Track B: local |
|------|-----------------------------|----------------|
| Dashboard: request count, latency p50/p95, error rate, replica count | Azure Monitor / Log Analytics workbook (free 5 GB/month ingestion; keep logs small) | **Grafana** from kube-prometheus-stack (+ Knative metrics) |
| Alert (e.g. error rate > 5% or p95 > 1 s) | An Azure Monitor alert → an action group that **emails me** (free) | A Grafana alert → a free **Discord webhook** or **ntfy.sh** push to my phone |
| Cost view | Cost Management **by tag** + how much credit I used | **OpenCost** (open source) on kind to see cost per namespace/pod + my Infracost estimate |

- Core: define **one SLO** for my API (e.g. "99% of requests < 500 ms over 7 days") and explain how I'd measure it with my dashboard.
- Core: a 1-page **Well-Architected self-review**: for each pillar, one thing I did and one gap.
- Core: **teardown**: `terraform destroy`, then verify nothing is left (Track A: the portal shows an empty resource group; Track B: `kind delete cluster`, `docker ps -a`, `docker system df`). Screenshot it.
- Stretch: add OpenTelemetry tracing to the API and view traces in **Jaeger** (free, runs locally).

**Definition of Done**
- [ ] Dashboard + 1 working alert that reached me
- [ ] SLO defined
- [ ] Well-Architected review written
- [ ] Everything destroyed; **$0 spent** documented (Track A: credit used + "no card on account"; Track B: nothing to pay)
- [ ] README with the architecture diagram, which track I used, how to run it, results, and lessons learned

---

## Self-check: interview questions I should be able to answer

1. What's the difference between a Type 1 and a Type 2 hypervisor? Which type does AWS use, and why?
2. A container and a VM both "isolate" an app. What exactly is shared in each case, and what are the security implications?
3. What are namespaces and cgroups? Which one stops a container from using all the RAM?
4. Explain IaaS vs PaaS vs serverless containers with one trade-off each.
5. What happens step by step when I run `kubectl apply -f deployment.yaml`?
6. Why did my HPA not scale (give 3 possible reasons)?
7. Why is Terraform state sensitive? What happens if two people run `apply` at the same time without locking?
8. Why should an app use an IAM role / managed identity instead of access keys? What was weaker about my Track B solution?
9. What is the difference between a public and a private subnet? Why do NAT gateways cost money?
10. Define SLI, SLO, SLA and error budget, using my own API as the example.
11. How does Knative relate to Google Cloud Run and Azure Container Apps? What does "scale to zero" cost in latency?
12. Using my Infracost estimate: what would SkyForge cost a company per month, and which resource is the most expensive?

## Real-world references to search (all free)
- "AWS Well-Architected Framework" / "Azure Well-Architected Framework" / "Google Cloud Architecture Framework"
- **Microsoft Learn** free paths: "AZ-900 Azure Fundamentals" (good for M0 concepts, free to study)
- "Kubernetes the hard way" (stretch reading, Kelsey Hightower)
- "Terraform best practices" (terraform-best-practices.com) and the official HashiCorp tutorials
- Knative docs: "Getting started with Knative on kind"
- "Docker best practices" in the official Docker docs
- "Google SRE book: Service Level Objectives" chapter (free online)
