# SNine_Project: a 30-day solo journey through 6 AI engineering fields

> My notes to myself, written in **October 2026**, one week after starting these 6 courses at school. I have **one month** and I'm working **solo**, so each project is sized for about 4 to 5 days. I'm building each one the way a real company would, so that I finish with skills (and portfolio pieces) that look like real work, not like school exercises.
>
> I'm leaving out code **on purpose**. For every milestone I wrote down **what** to build, **why** it comes at that point, **which concepts to look up**, and **how to know I'm done**. The searching, reading, and building are up to me. That struggle is where the learning happens.
>
> 💸 **All 6 projects cost $0 and need no bank card.** Everything runs on my PC (i9-14900HX, 32 GB RAM, **RTX 4080 Laptop GPU with 12 GB VRAM**) with free, open-source tools, plus student offers and free tiers that don't ask for a card. Each `01_MILESTONES.md` starts with a **zero-cost rule** table (free choice ✅ / avoid ❌).

---

## 1. Folder map

| # | Folder | Project | Field |
|---|--------|---------|-------|
| 01 | `01_Cloud_Computing_et_Virtualisation` | **SkyForge**: from a bare VM to an auto-scaling cloud service built with Infrastructure-as-Code | Cloud Computing & Virtualisation |
| 02 | `02_Ingenierie_Big_Data` | **CityPulse Lakehouse**: batch + streaming lakehouse on NYC taxi data | Big Data Engineering |
| 03 | `03_Explainable_AI` | **FairLend**: an explainable, audited credit-risk model | Explainable AI |
| 04 | `04_Modeles_de_Langage_et_TAL` | **DocuMind FR**: French/English document assistant (classic NLP → Transformers → RAG → LoRA) | Language Models & NLP |
| 05 | `05_DevOps_MLOps` | **ShipIt ML**: putting a model into production with CI/CD, a registry, and drift monitoring | DevOps & MLOps |
| 06 | `06_Ateliers_IA_Avancee` | **AgentLab**: a tool-using AI agent with an MCP server, evaluations, and security | Advanced AI Workshops (Agentic AI) |

Each folder contains:
- `00_PROJECT_IDEA.md`: the story, why this project covers the field, the stack, the architecture, and the deliverables.
- `01_MILESTONES.md`: the step-by-step journey, with self-check interview questions at the end.

---

## 2. Recommended order (and why)

The order below is **not** the order of my course list. It is the order in which **each project reuses skills from the one before it**:

```
Week 1  ── 01 Cloud & Virtualisation  (Linux, networking, Docker, K8s, Terraform)
             │  every later project runs in containers
Week 2  ── 02 Big Data  ──────────────▶ 03 Explainable AI
             │  I learn data at scale     │  I train and explain a real model
Week 3  ── 04 NLP & LLMs  ────────────▶ 05 DevOps / MLOps
             │  I build an LLM system     │  I ship model 03 (or 04) to production
Week 4  ── 06 Advanced AI (Agents)    (capstone: agents that use tools built in 02/04)
Days 29–30 ── buffer, polish, portfolio
```

| Days | Project | Main reason it sits here |
|------|---------|---------------------------|
| 1–5 | 01 Cloud & Virtualisation | Linux, networking, Docker, and IaC are the **ground floor** for every other project. |
| 6–10 | 02 Big Data | Uses Docker Compose from 01. Teaches me to handle data before I model it. |
| 11–14 | 03 Explainable AI | I need a trained model before I can explain it. Tabular data is the most common case in industry. |
| 15–19 | 04 NLP & LLMs | The most "hyped" field, so I come to it with solid engineering habits already in place. |
| 20–24 | 05 DevOps / MLOps | It needs **a model to ship**. I reuse FairLend (03) or DocuMind (04). |
| 25–28 | 06 Advanced AI | The capstone: an agent that calls tools, sits on top of the earlier projects, and is evaluated like a product. |
| 29–30 | Buffer | Finish stretch goals, write blog posts or READMEs, record demos. |

> **School alignment:** if my course covers a field earlier than this plan does, move that project up. Each `00_PROJECT_IDEA.md` has an "If I skipped a previous project" note so every project can also stand alone.

---

## 3. Daily rhythm (solo, sustainable)

Plan for about **5–6 focused hours a day**:

1. **Read and learn (1–1.5 h):** look up the "Concepts" listed in the milestone. Read the **official docs first**, then blog posts or videos.
2. **Build (3–4 h):** do the "What to build" tasks. Get it working ugly first, then clean it up.
3. **Write (30 min):** keep a `JOURNAL.md` in each project: what I did, what broke, what I learned. In real companies this is called an *ADR* (Architecture Decision Record). Look it up.
4. **Self-check (15 min):** tick the "Definition of Done" boxes. Don't move on until they are ticked or **consciously** marked as skipped.

**Rule of scope:** each milestone has a **Core** part (required) and a **Stretch** part (optional). If I fall behind, skip the stretch parts. Never skip a Core part.

---

## 4. Engineering habits that apply to EVERY project

These are the "real-life practices" that separate an engineer from a student. Apply them from Day 1.

| Habit | What it means | Search keywords |
|-------|---------------|-----------------|
| **Git from minute one** | One Git repo per project, small commits, meaningful messages, a feature branch for each milestone | "conventional commits", "trunk-based development" |
| **uv for Python** | One `pyproject.toml` per project, locked dependencies (`uv.lock`), pinned Python version | "uv project workflow", "uv workspaces" |
| **Lint + format** | Ruff for linting and formatting, run automatically by pre-commit | "ruff", "pre-commit hooks" |
| **Type hints** | Typed functions, checked by a type checker | "mypy", "pyright", "ty type checker" |
| **Tests** | pytest from the first milestone, even if it's 3 tests | "pytest fixtures", "testing data pipelines" |
| **Config ≠ code** | No hard-coded paths, keys, or passwords. Use `.env` files plus settings classes | "12-factor app", "pydantic-settings" |
| **Secrets hygiene** | Never commit keys. `.gitignore` my `.env` | "gitleaks", "secret scanning" |
| **README per project** | Problem, architecture diagram, how to run, results, limitations | "README driven development" |
| **Diagrams** | Draw the architecture before I build | "C4 model", "excalidraw", "mermaid diagrams" |
| **Cost awareness** | I spend $0, but I still **estimate** what my setup would cost a company (Infracost, token × price) and tear everything down when I'm finished | "FinOps basics" |

---

## 5. About the Python environment in this folder

- **Each project folder is its own independent uv project**, with its own `pyproject.toml`, `uv.lock`, `.python-version` (3.12), and `.venv`. Project names: `skyforge`, `citypulse`, `fairlend`, `documind`, `shipit-ml`, `agentlab`.
- **Why separate and not one shared venv:** the heavy stacks (PySpark/Airflow, torch/transformers/trl, mlflow/dvc, langgraph/MCP) pin conflicting versions. Separate projects can never conflict with each other. uv's global cache links packages, so 6 venvs cost little disk space.
- **Why not a uv workspace:** a workspace shares **one** lock file, so every member would have to agree on versions again. These projects are independent.
- Python 3.12 everywhere: every tool in these projects supports it, including PySpark, SHAP, and PyTorch.
- The root `.venv` + `pyproject.toml` are only for quick scratch tests.
- Global CLI tools (outside all projects): `ruff` and `pre-commit`, installed with `uv tool install`.
- Heavy services (Airflow, Kafka, the MLflow server, Qdrant, Prometheus/Grafana) run in **Docker**, not in these venvs.

**My daily loop:**
1. `cd` into the project folder, then `code .` (VS Code finds that folder's `.venv` by itself)
2. `uv add <package>` to install (never `pip install`)
3. `uv run python file.py` / `uv run pytest` to run things (no activation needed)
4. `git add .` → `git commit -m "..."` → `git push` (one commit per milestone step)

---

## 6. Accounts I will need (all free, **no card**)

| Account | Used in | Note (Oct 2026) |
|---------|---------|-----------------|
| GitHub | All (repos, Actions CI, GHCR) | Free. Apply for the **GitHub Student Developer Pack**. |
| Hugging Face | 04, 06 | Free account for models, datasets, Spaces |
| Azure for Students (**optional**, Track A) | 01, 05 | **$100 credit, no card**, verified with my school email. I can never be charged. If it isn't approved, I use **Track B (local cloud)**. Ask my teacher about **AWS Academy** (no card). ❌ Never the normal AWS/Azure/GCP trials: they need a card. |
| Docker Desktop (or Rancher Desktop / Podman) with WSL2 | All | Needed on Windows |
| Kaggle | 03 (datasets, backup GPU) | Free (phone verification, no card) |
| Free LLM API (optional) | 04, 06 | **GitHub Models** (uses my GitHub account) or **Google AI Studio** free tier. No card. Main path: **Ollama** locally on my RTX 4080. Only send public data to free APIs. |
| Databricks Free Edition (optional) | 02 (stretch) | Free sign-up, no card |

---

## 7. What "good at this field" means at the end

At the end of each project I should be able to:
1. **Explain** every design decision in an interview (each milestone file ends with self-check questions).
2. **Rebuild** the core of it from scratch without a tutorial.
3. **Show** a public GitHub repo with a clean README, a diagram, tests, and results.

Next step for me: open `01_Cloud_Computing_et_Virtualisation/00_PROJECT_IDEA.md` and start Day 1.
