# AgentLab: a tool-using AI agent with an MCP server, evaluations, and security

**Field:** Advanced AI Workshops (Ateliers IA Avancée), focused on **Agentic AI**, the most in-demand advanced topic for AI engineers in 2026 · **Duration:** 4 days (~25 h) · **Level at the end:** an AI engineer who can design, build, **evaluate**, and **secure** an LLM agent, and who knows when *not* to use one

> **Why agents?** The "advanced" topics on AI engineering job postings in 2026 are dominated by agentic systems: tool calling, the **Model Context Protocol (MCP)**, stateful workflows, multi-agent patterns, agent evaluation, and agent security. If my school's workshops cover something else (computer vision, diffusion models, reinforcement learning…), see the **alternatives** at the end of this file. The same engineering method applies.

---

## 1. The story (project idea)

My team now has three systems (from the earlier projects): a **regulation assistant** (DocuMind, 04), a **taxi lakehouse** (CityPulse, 02), and a **model service** (ShipIt ML, 05). People keep asking questions that span all of them:

- "How many airport trips were there last month, and what's the average tip?" → SQL over the lakehouse
- "Does our credit model count as high-risk under the AI Act? Cite the article." → RAG
- "Which model version is in production, and is drift detected?" → model service API
- "Write me a short report combining all three." → planning + several tools + synthesis

I build **AgentLab**, an agent that **plans, calls tools, asks a human before risky actions, remembers context, and is measured and secured like a product**.

> **Skipped earlier projects?** Use stand-in tools: a **SQLite/DuckDB** database (e.g. the Chinook sample DB), a **Wikipedia/web search** tool, a **calculator**, and a **mock "ops API"** that returns JSON. The learning is identical.

---

## 2. Why this project covers the field

| Advanced concept | Milestone |
|------------------|-----------|
| Agents vs workflows; when not to use agents; agent design patterns (ReAct, plan-and-execute, reflection, routing) | M0 |
| LLM tool/function calling, JSON schemas, structured outputs, the agent loop **from scratch** | M1 |
| **Model Context Protocol (MCP)**: servers, clients, tools/resources/prompts, transports, auth | M2 |
| Stateful agent frameworks, graphs, checkpointing, **human-in-the-loop**, text-to-SQL safety | M3 |
| Memory (short/long-term), **context engineering**, multi-agent architectures and their real costs | M4 |
| **Agent evaluation** (task success, trajectory, cost, latency), observability, OpenTelemetry for GenAI | M5 |
| **Agent security** (indirect prompt injection, excessive agency, tool misuse), sandboxing, guardrails, deployment | M6 |

---

## 3. Stack (current as of October 2026)

| Purpose | Tool (preferred) | Alternatives | Note |
|---------|------------------|--------------|------|
| LLM | A local open model through **Ollama** on my RTX 4080 (a recent 7–14B Qwen/Llama/Mistral/Gemma with tool-calling support) **and** a **free-tier API model** with no card (GitHub Models or Google AI Studio) | — | Compare reliability and estimated cost; **$0 spent** |
| Agent loop (M1) | **Plain Python + the provider SDK**. No framework. | — | I must understand the loop before using a framework |
| MCP | Official **MCP Python SDK** (includes FastMCP-style decorators) | FastMCP standalone | The **2026-07-28 spec revision** made the protocol **stateless** (sessions removed) and added an extensions framework; check the current spec at modelcontextprotocol.io |
| MCP clients for testing | **MCP Inspector**, **VS Code**, my own agent (all free); Claude Desktop if its free plan allows local MCP servers | — | |
| Framework (M3+) | **LangGraph** (graphs, checkpointing, HITL, the most production-proven) | **PydanticAI** (typed, lightweight), OpenAI Agents SDK, Claude Agent SDK, Google ADK, Microsoft Agent Framework | Pick ONE. AutoGen is in maintenance mode; avoid it for new work |
| Memory / vectors | Qdrant or pgvector (from project 04) | — | |
| Structured data tool | **DuckDB** (reads my Iceberg/Parquet tables from project 02) | SQLite | Read-only connection! |
| Observability | **Self-hosted Langfuse** (Docker) or **MLflow 3 tracing** (OpenTelemetry-based); the **OpenTelemetry GenAI semantic conventions** | Arize Phoenix (open source, local) | Avoid paid SaaS tiers |
| Evaluation | My own task suite + **DeepEval** / **Inspect** (UK AISI) / framework eval tools | — | |
| Guardrails | Input/output validation (Pydantic), allow-lists, LLM-based classifiers; read about Llama Guard-style safety models | NeMo Guardrails | |
| Sandbox (if code execution) | A Docker container with no network, or a hosted sandbox | — | |
| Serving | **FastAPI** + Docker (from project 01/05) + a **Cloudflare quick tunnel** for a free public demo URL | — | |

> 💸 **The whole project costs $0 and needs no card.** See the zero-cost rule at the top of `01_MILESTONES.md`.

---

## 4. Target architecture

```
               user ─▶ FastAPI /chat (streaming)
                          │
                ┌─────────▼──────────┐   checkpoints (state, history)
                │  Agent graph       │◀──────────────▶ Postgres/SQLite
                │  (LangGraph)       │
                │  router ─▶ planner │──▶ guardrails (input) ─▶ LLM (API / local)
                │      │             │◀── guardrails (output, tool-arg validation)
                │  executor ─▶ tools │
                │      │      ▲      │──▶ HUMAN APPROVAL node (risky actions)
                └──────┼──────┼──────┘
                       │ MCP client
       ┌───────────────┼──────────────────────────────┐
       ▼               ▼                              ▼
 MCP server:      MCP server:                    MCP server:
 "lakehouse"      "regulations"                  "ml-ops"
 - run_sql (RO)   - search_regulations (RAG)     - get_prod_model_version
 - list_tables    - get_article                  - get_drift_report
 - resource:      - prompt: compliance_check     - request_promotion  ⚠ needs approval
   schema docs
       │                                              │
   DuckDB / Iceberg (02)      DocuMind RAG (04)       ShipIt API + MLflow (05)

 All steps traced (OpenTelemetry GenAI) ─▶ Langfuse/MLflow ─▶ eval suite + dashboards
```

---

## 5. Final deliverables
1. Repo `agentlab/` with `agent_scratch/` (M1), `mcp_servers/` (M2), `agent/` (graph), `evals/`, `security/`, `api/`, `docs/`.
2. **Three MCP servers** that work in at least two different MCP clients.
3. An **evaluation report**: task success rate, tool-selection accuracy, average steps, tokens, estimated cost and latency per task, scratch vs framework, single vs multi-agent, free API vs local model.
4. A **red-team report**: attacks tried, results, mitigations, residual risk.
5. A Dockerised agent API + a 3–5 minute demo recording, including a human-approval moment.

---

## 6. Alternatives (if my "Ateliers IA Avancée" cover another topic)

Use the **same 7-milestone structure** (framing → from scratch → standard tool → advanced features → evaluation → security/deployment):
- **Computer vision:** "VisionGuard": fine-tune YOLO-family detection on a custom dataset, then add segmentation (SAM-family) and a vision-language model for captions. Evaluate with mAP, deploy at the edge (ONNX/TensorRT-lite).
- **Generative / diffusion:** "BrandGen": a LoRA-fine-tuned image diffusion model for a consistent style, with evaluation (FID/CLIP score + human preference) and safety filters.
- **Reinforcement learning:** "GridPilot": DQN → PPO on Gymnasium environments, then RL fine-tuning of a small LLM (GRPO-style) on a verifiable task (math/format).
