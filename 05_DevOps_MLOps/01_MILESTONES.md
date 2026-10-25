# ShipIt ML: milestones

> Format: **Goal → Why now → Concepts → What to build (Core / Stretch) → Best practices → Definition of Done → Pitfalls**. Total: 7 milestones over about 5 days.

---

## 💸 Zero-cost rule: this whole project costs $0 and needs no card

| Need | Free choice ✅ | Avoid ❌ |
|------|---------------|---------|
| CI/CD | **GitHub Actions on a public repo**: free, unlimited minutes for public repositories, and **environments with required reviewers** work on the free plan for public repos (private repos on the free plan get limited minutes and no required reviewers, so I check the GitHub docs) | Paid CI plans |
| Deploy from CI to my PC | A **self-hosted GitHub Actions runner** on my laptop (free). GitHub's cloud runners can't reach my local kind cluster, so the deploy jobs run on my own machine. Staging smoke tests can also run inside a throwaway **kind cluster in the CI job** | — |
| Deploy target | Project 01 **Track A** (Azure Container Apps on student credit, no card) or **Track B** (kind + Knative locally) | Paid cloud services |
| Container registry | **GHCR** (free) | ACR / ECR (bill daily) |
| Data versioning remote | **A local folder** (e.g. a separate directory or drive), **Google Drive** (free 15 GB, DVC supports it), or the S3-compatible **Garage/SeaweedFS** from project 02 | Paid buckets |
| Tracking + registry | **MLflow** server in Docker Compose (Postgres + local artifact folder) | Paid hosted MLflow / W&B team plans |
| Monitoring | **Prometheus + Grafana**, **Evidently** (open-source library), **NannyML** (open source) | Evidently Cloud, WhyLabs, Arize paid tiers |
| Alerts | Grafana → a free **Discord webhook** or **ntfy.sh** push | Paid paging tools |
| Security scans | **Trivy**, **gitleaks**, **Dependabot** (all free) | Paid scanners |

**My hard rules:** never enter a card; keep the repo **public** (I never commit secrets or personal data, and gitleaks enforces that); **OIDC to the cloud only applies on Track A**. On Track B there are no cloud keys at all, and I write down why.

---

## M0: The MLOps mindset (Day 1, first ~2 h)

**Goal:** understand *why* MLOps exists and define what "production-ready" means for **my** model.

**Why now:** without a target, I'll over-engineer some parts and forget others. Real teams start with requirements and a maturity goal.

**Concepts to learn**
- DevOps principles: CI, CD (delivery vs deployment), trunk-based development, "you build it, you run it", **DORA metrics** (deployment frequency, lead time, change failure rate, MTTR)
- Why ML is different: data + code + model all change; **silent failures**; training–serving skew
- Sculley et al. (2015), "Hidden Technical Debt in Machine Learning Systems". Read it; it's short.
- **Google's MLOps maturity levels 0 / 1 / 2** and the **CI / CD / CT** pipeline idea
- Microsoft's / Azure's MLOps maturity model (for comparison)

**What to build**
- Core: `docs/requirements.md` containing the SLOs for the API (e.g. p95 < 200 ms, 99.5% availability), model quality requirements (minimum AUC, maximum fairness gap from project 03), retraining policy, and rollback policy.
- Core: `docs/architecture.md` with my version of the target diagram, plus a note on which maturity level I'm aiming for and why.

**Definition of Done**
- [ ] Requirements + architecture docs written
- [ ] I can explain maturity levels 0/1/2 and what CT means

---

## M1: Repository engineering (Day 1, ~4 h)

**Goal:** a clean, enforced, professional repository, the foundation everything else is automated on.

**Why now:** CI/CD can only automate what's already well structured. Messy notebooks can't be tested or deployed. Every serious ML team has this layer, and it's what reviewers notice first in my portfolio.

**Concepts to learn**
- The `src/` layout, packaging with `pyproject.toml`, `uv` lock files and reproducibility
- Linting/formatting (Ruff), type checking, pre-commit hooks
- **The ML testing pyramid**: unit tests (feature functions), data tests (schema, ranges), model tests (minimum performance, invariance, directional expectation, e.g. "more payment delays ⇒ higher risk"), API/contract tests
- Branch protection, PR reviews, conventional commits, semantic versioning
- Configuration management (pydantic-settings, `.env`, `params.yaml`)

**What to build**
- Core: refactor the project 03 notebook logic into a `src/shipit/` package: `data.py`, `features.py`, `train.py`, `evaluate.py`, `predict.py`, `config.py`.
- Core: Ruff + type checker + pre-commit (including **gitleaks** for secrets); a `justfile`/`Makefile` with `setup`, `lint`, `test`, `train`, `serve`.
- Core: tests: ≥ 5 unit tests, ≥ 3 data tests (pandera schema), ≥ 3 **behavioural model tests** (minimum AUC on a fixed sample, an invariance test, a directional test).
- Core: GitHub: protected `main`, PR template, issue templates, CODEOWNERS (myself).
- Stretch: generate docs with MkDocs.

**Definition of Done**
- [ ] `just lint` and `just test` pass locally; pre-commit blocks bad commits
- [ ] No notebook is needed to train the model
- [ ] Behavioural tests exist and fail when I break the model on purpose

---

## M2: Reproducibility: data versioning, pipelines, experiment tracking (Day 2, ~6 h)

**Goal:** anyone (including CI) can reproduce **exactly** the same model from a commit hash.

**Why now:** before automating training in CI (M4), training must be deterministic and traceable. Real-world question: "Which data, code, and parameters produced the model that rejected this customer last March?" I must be able to answer it.

**Concepts to learn**
- Data versioning: why Git can't store data, how **DVC** works (hashes, `.dvc` files, remotes, cache)
- Pipelines as DAGs: stages, dependencies, outputs, parameters, caching (only re-run what changed)
- Experiment tracking: params, metrics, artefacts, tags, **lineage** (git commit + data hash on every run)
- MLflow architecture: tracking server, **backend store** (Postgres) vs **artifact store** (bucket/local), model signatures & input examples, `LoggedModel` (MLflow 3)
- Randomness: seeds, deterministic splits

**What to build**
- Core: put the raw data under **DVC** with a free remote: a local folder acting as the remote, Google Drive, or the S3-compatible Garage/SeaweedFS from project 02.
- Core: a `dvc.yaml` pipeline: `validate_data → featurize → train → evaluate` with `params.yaml` (hyperparameters, thresholds). Change one parameter and confirm that only the downstream stages re-run.
- Core: an **MLflow tracking server** in Docker Compose (Postgres + artifact storage). Every training run logs params, metrics (AUC, PR-AUC, Brier, fairness gap), plots, the model with a **signature**, the git commit, and the DVC data hash.
- Core: **reproducibility test**: delete everything local, clone the repo, `dvc pull`, `dvc repro`, and get the same metrics.
- Stretch: replace or wrap the DVC pipeline with **Prefect 3** or **ZenML** and compare the developer experience.

**Definition of Done**
- [ ] `git clone` → `dvc pull` → `dvc repro` reproduces the metrics exactly
- [ ] Every MLflow run links to a commit + data version
- [ ] I can explain the backend store vs the artifact store

**Pitfalls:** committing data to Git; MLflow writing artefacts to a local path that the API container can't see.

---

## M3: Model registry, packaging, and serving (Day 3 morning, ~5 h)

**Goal:** a production-grade prediction API that loads the **approved model from the registry**, not from a file path.

**Why now:** I have reproducible, tracked models. Now I need a **controlled hand-off** from training to serving. The registry is the contract between data scientists and the platform.

**Concepts to learn**
- Model registry: versions, **aliases** (`@champion`, `@challenger`), tags, approval workflow
- Online vs batch vs streaming inference; latency budgets
- API design: input validation (Pydantic), versioned endpoints (`/v1/predict`), error handling, idempotency, request IDs
- **Training–serving skew** and how sharing the same `features.py` prevents it
- Health vs readiness endpoints; graceful shutdown; worker/concurrency settings (uvicorn/gunicorn)
- Exposing Prometheus metrics from an app

**What to build**
- Core: register the best model; assign `@champion`.
- Core: a FastAPI service that, **at startup**, loads `models:/fairlend@champion` (cached), validates input with Pydantic, returns `{probability, decision, model_version, request_id}` and (optionally) the top-3 reason codes from project 03. Endpoints: `/v1/predict`, `/v1/predict/batch`, `/health`, `/ready`, `/metrics`.
- Core: a **prediction log**: every request + response written (asynchronously) to a store (Postgres/Parquet) for monitoring in M5. **Don't log raw personal data**; look up "pseudonymisation".
- Core: Dockerise it (non-root, slim, health check), then load-test with Locust/k6 and record p50/p95/p99 latency and max RPS.
- Core: API contract tests with `TestClient`.
- Stretch: do the same with **BentoML** and compare.

**Definition of Done**
- [ ] The API serves the `@champion` model; switching the alias + a restart (or reload) changes the model with **no code change**
- [ ] Latency numbers meet (or are compared with) my M0 SLO
- [ ] The prediction log is being written

---

## M4: CI/CD pipelines with quality gates (Day 3 afternoon + Day 4 morning, ~6 h)

**Goal:** fully automated, **gated** pipelines: bad code, bad data, or a worse model **cannot** reach production.

**Why now:** every component exists and is tested. Now I chain them together so that humans only *approve*; they don't *do*. This is the "CI/CD" part of CI/CD/CT.

**Concepts to learn**
- GitHub Actions: workflows, jobs, steps, matrix, caching (uv cache), artefacts, **environments** (staging/prod) with **required reviewers**, secrets vs **OIDC federation** to the cloud (no long-lived keys)
- Quality gates: lint/test gates, **data validation gates**, **model evaluation gates** (challenger must beat champion by X and must not violate the fairness threshold)
- Container supply chain: image scanning, SBOM, pinned actions (by SHA)
- Deployment strategies: recreate, rolling, blue/green, canary
- Semantic versioning of images (`v1.4.2` + git SHA tag)

**What to build**
- Core: **`ci.yml`** (on every PR): Ruff, type check, pytest with coverage, pandera data checks, train on a **small sample** to catch pipeline breakage. It must be **fast (< 5 min)**.
- Core: **`train.yml`** (on merge to main or manually): full `dvc repro` → log to MLflow → register a new version as **`@challenger`** → an **evaluation gate** comparing it with `@champion` on the same held-out set, covering AUC, calibration, and the fairness gap. It produces a comparison report as a job summary.
- Core: **`cd.yml`**: build image → Trivy scan (fails on CRITICAL) → push to the registry → deploy to **staging** → smoke tests → **manual approval** → deploy to **prod**.
- Core: **Track A**: use **OIDC** for Azure auth (search "GitHub Actions OIDC Azure"). **Track B**: register a **self-hosted runner** on my PC for the deploy jobs (search "GitHub Actions self-hosted runner Windows/WSL"), limit it to this repo, and note the security risk of self-hosted runners on public repos (only trigger deploys from `main`, never from fork PRs).
- Core: break things on purpose (a failing test, a corrupted data file, a worse model) and screenshot each gate blocking the change.
- Stretch: Dependabot/Renovate for dependency updates; pin actions by commit SHA.

**Definition of Done**
- [ ] A PR with failing tests can't be merged
- [ ] A worse model is **not** promoted (shown)
- [ ] Staging → approval → prod works end-to-end
- [ ] No long-lived cloud keys in GitHub secrets (OIDC), or a documented reason why not

**Pitfalls:** full training in PR CI (too slow); secrets printed in logs; non-deterministic evaluation that makes the gate flaky.

---

## M5: Monitoring: system health and ML health (Day 4 afternoon, ~4 h)

**Goal:** detect when the service **or the model** is degrading, ideally **before** users or the business notice.

**Why now:** models decay silently. The input data drifts (economic changes, new customer segments) and the API still returns 200 OK while making worse decisions. Monitoring closes the loop that makes continuous training possible.

**Concepts to learn**
- System metrics: RED (Rate, Errors, Duration) / the four golden signals; Prometheus data model (counters, gauges, **histograms**), PromQL basics; Grafana dashboards; alerting rules
- ML monitoring: **data drift** (covariate shift), **prediction drift**, **concept drift**, label delay and proxy metrics
- Drift statistics: **PSI**, KS test, Wasserstein distance, Jensen–Shannon; why "statistically significant" ≠ "important" with big samples
- Data-quality monitoring in production (nulls, unseen categories, out-of-range values)

**What to build**
- Core: Prometheus + Grafana (Docker Compose or Helm). Dashboard panels: RPS, error rate, latency p50/p95/p99, model version serving, **prediction score distribution**, and the decision rate (% rejected).
- Core: an **alert** (latency SLO breach, or rejection rate jumping by > X%).
- Core: a **traffic simulator** that sends (1) normal requests drawn from the test set, then (2) **drifted** requests (e.g. shift the income/limit distribution, raise payment delays, introduce a new category).
- Core: a daily **Evidently** job (scheduled with GitHub Actions or Prefect) comparing the prediction log with the reference (training) data, producing an HTML drift report + a pass/fail test suite. When drift is detected, it emits an event or opens a GitHub issue automatically.
- Core: simulate **delayed ground truth** (labels arriving later) and compute the real performance over time.
- Stretch: compare Evidently with NannyML's performance estimation *without* labels (CBPE).

**Definition of Done**
- [ ] Grafana dashboard live during the load and drift simulations
- [ ] Drift is detected on the drifted traffic (and *not* on the normal traffic; check for false alarms)
- [ ] An alert fired and I have a screenshot

---

## M6: Continuous training, safe releases, and operations (Day 5, ~5 h)

**Goal:** close the loop: **drift → retrain → challenger → safe rollout → (rollback)**, plus the documentation an on-call engineer needs.

**Why now:** this is the final level of maturity: the system improves itself under human supervision. It comes last because it depends on everything before it (pipelines, registry, gates, monitoring).

**Concepts to learn**
- Retraining triggers: scheduled vs drift-based vs performance-based; the risks of feedback loops
- Champion/challenger; **shadow deployment** (the challenger gets a copy of the traffic but doesn't answer); **canary** (5% → 25% → 100%); automatic rollback criteria
- Runbooks, on-call, blameless **postmortems**
- Responsible ML in operations: re-running the fairness checks on every retrain, keeping the model card up to date, audit trails
- Awareness: **LLMOps** differences (prompt versioning, eval suites instead of accuracy, tracing, cost monitoring), which link to project 04

**What to build**
- Core: the drift event from M5 triggers `train.yml` on fresh (drifted + labelled) data, which produces a new `@challenger`.
- Core: **shadow mode**: the API scores with both champion and challenger and logs both; only the champion's answer is returned. Compare them after N requests.
- Core: **canary rollout** of the challenger (Track A: Container Apps revision traffic split; Track B: **Knative** revision traffic split, or ingress-nginx canary annotations in kind). Define automatic **rollback** criteria (error rate, latency, decision-rate shift) and **demonstrate one rollback**.
- Core: `docs/runbook.md` (for each alert: meaning, dashboard link, diagnosis steps, mitigation) and `docs/postmortem-001.md` about an incident I caused on purpose (timeline, root cause, impact, action items, blameless tone).
- Core: final README: architecture diagram, how to run locally, the CI/CD badges, screenshots, the maturity level reached, and what I'd add at company scale (feature store, Kubeflow/KServe, multi-region, etc.).
- Stretch: add an eval-suite CI gate for DocuMind (project 04): a prompt or model change must not lower faithfulness below a threshold. That's LLMOps in practice.

**Definition of Done**
- [ ] A full retrain → shadow → canary → promote (or rollback) cycle demonstrated and recorded
- [ ] Runbook + postmortem written
- [ ] I could hand this repo to a new teammate and they could run it from the README alone

---

## Self-check: interview questions

1. What's different about CI/CD for ML compared with regular software? What is CT?
2. Describe Google's MLOps maturity levels. Which level did I reach, and what's missing for the next one?
3. How do I guarantee that a model can be reproduced 6 months from now?
4. What's training–serving skew? How did my design prevent it?
5. What does a model registry give me that a model file in a bucket doesn't?
6. What gates did my pipeline have, and which failure does each one prevent?
7. Data drift vs concept drift vs prediction drift: how do I detect each one when labels are delayed?
8. PSI = 0.3 on one feature. What do I do next? (Hint: not "retrain immediately".)
9. Shadow vs canary vs blue/green: give a scenario for each.
10. Why use OIDC instead of storing cloud keys in CI secrets?
11. How do the DORA metrics apply to an ML team?

## References to search
- Google Cloud: "MLOps: Continuous delivery and automation pipelines in machine learning"
- Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)
- Chip Huyen, **"Designing Machine Learning Systems"** (O'Reilly)
- **Made With ML** (Goku Mohandas) and the **DataTalksClub MLOps Zoomcamp** (free, updated yearly)
- The MLflow 3 docs (Model Registry, aliases, tracking server); the DVC docs (pipelines, remotes)
- Evidently docs (data drift presets, test suites); Prometheus docs (histograms, alerting)
- Google SRE book (free): monitoring, postmortem culture
- "Accelerate" (Forsgren, Humble, Kim): the DORA research
