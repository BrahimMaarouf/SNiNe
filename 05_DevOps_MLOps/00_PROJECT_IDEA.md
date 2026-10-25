# ShipIt ML: putting a model into production with CI/CD, a registry, monitoring, and retraining

**Field:** DevOps & MLOps · **Duration:** 5 days (~30 h) · **Level at the end:** an ML engineer who can take a notebook model to a **monitored, automatically tested and deployed** service, reaching **MLOps maturity level 1–2** (Google's scale)

---

## 1. The story (project idea)

The bank from project 03 approved **FairLend**, my credit-risk model. Now the platform team says:

> "We don't deploy notebooks. Give us a service that is **reproducible**, **tested**, **versioned**, **deployed automatically**, **monitored for drift**, and that can be **retrained and rolled back** without a hero working at 2 a.m."

I build the **complete MLOps loop** around the model:

```
code + data + params ─▶ CI (lint, test, validate) ─▶ train & evaluate ─▶ registry (champion/challenger)
        ▲                                                                     │
        │                                                              CD: build, scan, deploy
        │                                                                     ▼
   retrain trigger ◀── drift / performance alerts ◀── monitoring ◀── serving API (prod)
```

**The model is not the point.** It's deliberately one I already have, so I can spend 100% of my time on **engineering the system around it**. That's exactly what MLOps is (see "Hidden Technical Debt in Machine Learning Systems": the model is the small box in the middle).

> **No FairLend?** Use any tabular classification dataset (e.g. the UCI credit default or Bank Marketing dataset). Or ship **DocuMind's classifier** from project 04.

---

## 2. Why this project covers the field

| Course concept | Milestone |
|----------------|-----------|
| DevOps culture, CI/CD, trunk-based development, IaC, DORA metrics | M0, M1, M4 |
| The ML lifecycle, MLOps maturity levels, CI/CD/**CT** (continuous training) | M0 |
| Code quality: linting, typing, testing pyramid for ML | M1 |
| Data versioning, experiment tracking, reproducibility, pipelines | M2 |
| Model registry, packaging, model serving (online/batch), API design | M3 |
| CI/CD pipelines, quality gates, container security, environments, approvals | M4 |
| Observability: system metrics + **ML metrics** (data drift, concept drift, prediction drift) | M5 |
| Continuous training, champion/challenger, **shadow & canary deployments**, rollback, runbooks | M6 |

---

## 3. Stack (current as of October 2026)

| Concern | Tool (preferred) | Alternatives | Note |
|---------|------------------|--------------|------|
| Package & env | **uv** (pyproject + uv.lock) | Poetry | The current standard for fast, reproducible Python environments |
| Quality | **Ruff** (lint + format), **mypy/pyright**, **pre-commit**, **pytest** (+ pytest-cov) | — | |
| Task runner | **just** or **Make** | nox / tox | One command for each workflow |
| Data versioning | **DVC** (free remote: local folder, Google Drive, or Garage/SeaweedFS from project 02) | lakeFS, Delta/Iceberg time travel | |
| Pipelines | **DVC pipelines** (simple) or **Prefect 3** / **ZenML** | Airflow (project 02), Kubeflow, Metaflow | Pick ONE |
| Tracking + registry | **MLflow 3.x** (tracking server backed by Postgres + artifact store) | Weights & Biases | Use **model aliases** (`@champion`, `@challenger`); the old "stages" API is deprecated |
| Data validation | **pandera** or **Great Expectations** | — | |
| Serving | **FastAPI** + Pydantic (or **BentoML** / MLflow serving) | Ray Serve, KServe | |
| Containers | **Docker** (multi-stage, from project 01) | — | |
| CI/CD | **GitHub Actions** on a public repo (free; environments + required reviewers; OIDC on Track A; a **self-hosted runner** on my PC for local deploys) | GitLab CI | |
| Registry | **GHCR** (free) | — | Avoid paid cloud registries |
| Security | **Trivy** (image + deps), **gitleaks**, Dependabot/Renovate | Snyk | |
| Deploy target | Project 01 **Track A** (Azure Container Apps on student credit, no card) **or** **Track B** (kind + Knative on my PC) | — | $0 either way |
| Monitoring | **Prometheus + Grafana** (system), **Evidently** (open-source drift reports and tests) | NannyML (open source) | |

> 💸 **The whole project costs $0 and needs no card.** See the zero-cost rule at the top of `01_MILESTONES.md`.
| Load test | **Locust** or **k6** | — | |
| (Stretch) Feature store | **Feast** | — | |

---

## 4. Target architecture

```
 GitHub repo (main protected, PRs required)
   ├── src/ (features, train, evaluate, serve)    ├── tests/ (unit, data, model, API)
   ├── dvc.yaml + params.yaml                     ├── .github/workflows/ (ci.yml, train.yml, cd.yml)
   └── docker/, infra/ (Terraform from project 01), monitoring/

 PR opened ─▶ CI: ruff · types · pytest · data validation · train on a sample · eval gate
 merge     ─▶ train.yml: full DVC pipeline ─▶ MLflow run ─▶ register a new version as @challenger
                                                 └─▶ compare with @champion (metrics + fairness)
           ─▶ cd.yml: build image ─▶ Trivy scan ─▶ push ─▶ deploy STAGING ─▶ smoke tests
                                                       ─▶ manual approval ─▶ deploy PROD (canary)
 PROD API ─▶ /predict · /health · /metrics ─▶ Prometheus ─▶ Grafana + alerts
          └─▶ prediction log ─▶ daily Evidently drift job ─▶ drift? ─▶ trigger the retraining workflow
```

---

## 5. Final deliverables
1. Repo `shipit-ml/`: a production-grade repo template (I can reuse it for every future project).
2. Working **CI** (fails on bad code, bad data, or a worse model) and **CD** (staging → approval → prod).
3. An MLflow server with runs, a registered model, and aliases; DVC-tracked data and pipeline.
4. A live API with a Grafana dashboard and an **Evidently drift report** generated from simulated production traffic.
5. A demonstrated **retrain → challenger → promote → canary → rollback** cycle.
6. `docs/`: architecture, ADRs, a **runbook** ("what to do when alert X fires"), and a **postmortem** of one incident I caused on purpose.

## 6. If I skipped earlier projects
- No project 01? Deploy to a **local kind cluster** or just Docker Compose; skip the cloud.
- No project 03? Train a quick LightGBM classifier on a public tabular dataset on Day 1 morning (≤ 2 h). Don't spend more time than that on the model.
