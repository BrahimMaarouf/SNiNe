# AgentLab: milestones

> Format: **Goal → Why now → Concepts → What to build (Core / Stretch) → Best practices → Definition of Done → Pitfalls**. Total: 7 milestones over about 4 days.

---

## 💸 Zero-cost rule: this whole project costs $0 and needs no card

My **RTX 4080 Laptop GPU (12 GB VRAM)** runs tool-calling LLMs locally, so the agent never needs a paid API.

| Need | Free choice ✅ | Avoid ❌ |
|------|---------------|---------|
| Main LLM | **Ollama** locally: a recent **7–14B instruct model with tool-calling support**, in 4-bit (fits in 12 GB) | Paid APIs |
| "API model" for comparison | A free tier with **no card**: **GitHub Models** (free, rate-limited, works with my GitHub account) or **Google AI Studio** (Gemini API free tier). I check the current limits first and design around rate limits (retries, backoff) | Any API that asks for a card or credits |
| Frameworks / MCP | LangGraph or PydanticAI, the official MCP Python SDK, MCP Inspector (all open source) | — |
| MCP clients for testing | **MCP Inspector**, **VS Code** (free), **my own agent**; Claude Desktop if its free plan supports local MCP servers | Paid plans |
| Tracing / evaluation | **Self-hosted Langfuse** (Docker) or local **MLflow tracing**; DeepEval / Inspect (open source) | LangSmith / Arize paid tiers |
| Deployment + public demo | **Docker Compose** on my PC + a **Cloudflare quick tunnel** for a temporary public URL (free, no account) | Paid hosting |

**My hard rules**
- Never enter a card. If a provider asks for billing details, I don't use it.
- **Free API tiers may log or train on my prompts**, so I only send public or synthetic data and never secrets or personal data. That's also part of the M6 security lesson.
- Every "cost" number is an **estimate**: tokens × public API price per 1M tokens. Budget caps are set in **tokens per task**, so I learn cost control while spending $0.

---

## M0: Framing: agents, workflows, and when not to use them (Day 1, first ~2 h)

**Goal:** a clear design that defines what the agent must do, which tools it gets, what it must **never** do, and how success is measured.

**Why now:** most failed agent projects fail because of scope, not code. They give an LLM too much freedom on a problem that a simple workflow would solve better. The industry lesson of 2025–2026: **use the simplest pattern that works**, and add autonomy only where it pays off.

**Concepts to learn**
- What an agent is: LLM + tools + loop + (memory) + environment feedback
- **Workflows vs agents** (read Anthropic's "Building effective agents"): prompt chaining, routing, parallelisation, orchestrator–workers, evaluator–optimizer, autonomous agents
- Patterns: **ReAct**, plan-and-execute, reflection/self-critique
- The cost of autonomy: latency, cost, unpredictability, a larger security surface
- Read: OpenAI's "A practical guide to building agents", or an equivalent recent guide

**What to build**
- Core: `docs/agent_spec.md` containing:
  - **10–15 target user tasks** (easy → hard, single-tool → multi-tool), each with its expected tools and expected answer. This becomes my evaluation suite in M5, so build it now.
  - A tool list with an **input schema, output, side effects, and risk level** (read-only / write / irreversible)
  - **Hard rules** (e.g. SQL is read-only; promotion requires human approval; never reveal secrets; cite sources)
  - Success metrics (task success ≥ 80%, average tokens per task < N (≈ estimated $ at public API prices), p95 latency < Y s)
- Core: for each task, decide: **workflow or agent?** Justify it.

**Definition of Done**
- [ ] Agent spec with the tasks, tools, risks, rules, and metrics
- [ ] I can explain 5 agentic patterns and give a use case for each

---

## M1: The agent loop from scratch (Day 1, ~4 h)

**Goal:** build a working tool-calling agent with **no framework**, so that I know exactly what every framework does under the hood.

**Why now:** frameworks change every few months; the loop doesn't. When something breaks in production (an infinite loop, malformed tool arguments, context overflow), I can only debug it if I've written the loop myself.

**Concepts to learn**
- Chat message roles (system, user, assistant, tool); tool/function definitions as **JSON Schema**
- The loop: send messages + tools → the model returns text **or** tool calls → execute → append the results → repeat → stop
- Parallel tool calls; tool errors returned to the model as observations
- Stop conditions: final answer, **max iterations**, budget limits, timeouts
- **Structured outputs** (forcing a JSON schema for the final answer)
- Token accounting and the context window

**What to build**
- Core: a minimal agent (≈150–250 lines once I write it) with 3 tools: `run_sql` (read-only DuckDB), `search_regulations` (calls my DocuMind retriever, or a stand-in), and `calculator`.
- Core: robustness: validate the tool arguments (Pydantic) and return validation errors to the model; cap the iterations; record the token usage and cost per run; log the full trajectory as JSON.
- Core: run 5 tasks from my spec and record the success/failure for each with the **free-tier API model vs the local Ollama model**. Write down the differences in tool-calling reliability.
- Stretch: add a "reflection" step (the model critiques its answer before finishing) and measure whether it helps.

**Definition of Done**
- [ ] The agent solves ≥ 3 of the 5 tasks with my best model (local or free API)
- [ ] Trajectories are logged, and cost/tokens are counted
- [ ] I can draw the loop and explain every message in it

**Pitfalls:** letting the LLM write arbitrary SQL with write permissions; no iteration cap (runaway costs); swallowing tool errors silently.

---

## M2: Standardising tools with MCP (Day 2 morning, ~4 h)

**Goal:** expose my tools through **Model Context Protocol servers** so that *any* MCP-compatible client (my agent, IDEs, desktop assistants) can use them.

**Why now:** in M1 my tools were hard-wired into one agent. In real companies, tools are **shared infrastructure** maintained by different teams. MCP is the open standard that makes them reusable, the "USB-C of AI tools", and it's now supported by every major framework and client.

**Concepts to learn**
- MCP architecture: **host → client → server**; primitives: **tools**, **resources**, **prompts** (plus client features such as sampling and elicitation)
- Transports: **stdio** (local) vs **Streamable HTTP** (remote)
- **The 2026-07-28 spec revision**: removal of protocol-level sessions (stateless servers), header-based routing, the extensions framework. Read the official changelog and understand what it means for server design (horizontal scaling, no sticky sessions).
- Authorization for remote servers (OAuth 2.1-based); the current roadmap items (agent identity, progressive tool discovery)
- Tool design principles: clear names/descriptions (the LLM reads them!), narrow scope, typed inputs, helpful error messages, pagination of large outputs

**What to build**
- Core: **three MCP servers** with the official Python SDK:
  - `lakehouse`: tools `list_tables`, `describe_table`, `run_sql` (read-only, with row limits and a timeout); a **resource** containing the schema documentation
  - `regulations`: tools `search_regulations`, `get_article`; a **prompt** template `compliance_check`
  - `ml-ops`: tools `get_prod_model_version`, `get_drift_report`, `request_promotion` (**marked as dangerous**; it will need approval in M3)
- Core: test each server with **MCP Inspector**, then connect them to **at least one real MCP client** (e.g. VS Code, or Claude Desktop if its free plan allows local MCP servers) and to **my own M1 agent**, which I refactor into an MCP client.
- Core: run one server over **Streamable HTTP** in Docker.
- Stretch: add authentication to the HTTP server, following the MCP authorization spec.

**Definition of Done**
- [ ] 3 servers working in MCP Inspector + 1 external client + my agent
- [ ] One server is remote over HTTP in a container
- [ ] I can explain tools vs resources vs prompts, and what changed in the 2026 spec

---

## M3: Stateful agents with a framework + human-in-the-loop (Day 2 afternoon, ~4 h)

**Goal:** rebuild the agent as a **stateful graph** with persistence, streaming, and **human approval for risky actions**.

**Why now:** my from-scratch loop works, but production needs things that are tedious to build by hand: **checkpointing** (resume after a crash), **interrupts** for human approval, branching logic, and streaming. Now I can judge a framework instead of depending on it blindly.

**Concepts to learn**
- LangGraph concepts: **state**, nodes, edges, conditional edges, reducers, **checkpointers**, threads, **interrupts** (human-in-the-loop), streaming modes
- Routing: classify the request → choose a sub-flow (workflow for simple tasks, agent for complex ones)
- **Text-to-SQL safely**: schema grounding, a read-only DB role, query allow-lists/linting, row limits, timeouts
- Durable execution and idempotent tools (what happens if the agent retries a "write" tool?)

**What to build**
- Core: a LangGraph (or PydanticAI) agent with a **router** (simple question → direct workflow; complex → planner/executor), MCP tools loaded from my servers, a **checkpointer** (SQLite/Postgres), and streaming output.
- Core: **human-in-the-loop**: any call to `request_promotion` triggers an **interrupt**; a human approves or rejects it through the API/CLI; the graph resumes from the checkpoint.
- Core: **crash test**: kill the process mid-task, restart it, and resume the same thread.
- Core: compare it with my M1 agent on the same 5 tasks (success, code size, debuggability). Write an honest verdict.
- Stretch: implement the same agent in a second framework (PydanticAI or OpenAI/Claude Agent SDK) and compare.

**Definition of Done**
- [ ] Router + planner/executor graph working with MCP tools
- [ ] Human approval interrupt demonstrated
- [ ] Resume-after-crash demonstrated
- [ ] Scratch vs framework comparison written

---

## M4: Memory, context engineering, and multi-agent (Day 3 morning, ~4 h)

**Goal:** give the agent useful memory, keep its context **lean**, and test whether multiple agents actually beat one.

**Why now:** once tasks get longer, agents fail because of **context**: too much irrelevant tool output, lost instructions, forgotten user preferences. "Context engineering" became a core skill in 2025–2026. Multi-agent comes after it because it's mostly a context-management strategy (each sub-agent gets a focused context), and it has a real cost.

**Concepts to learn**
- Short-term memory (the conversation state) vs long-term memory (facts and preferences across sessions: semantic/episodic memory)
- **Context engineering**: summarising/compacting history, truncating and paginating tool outputs, selecting relevant memories, structured scratchpads, the risk of "context rot"
- Multi-agent patterns: **supervisor/orchestrator–workers**, handoffs, hierarchical teams; the costs (tokens ×N, coordination errors)
- Prompt caching (an economic factor for long system prompts)

**What to build**
- Core: **long-term memory**: store user preferences and facts (e.g. "the user prefers answers in French", "the user's team owns zone 132") in a vector store, retrieve them when relevant, and let the user view or delete them (privacy!).
- Core: **context management**: summarise old turns past N tokens; truncate large SQL results to a summary + the first rows; measure the token reduction on a long task.
- Core: a **multi-agent variant**: a supervisor + 2 specialists ("data analyst" with the lakehouse tools, "compliance analyst" with the regulation tools). Run the **multi-tool report task** with single-agent vs multi-agent and compare quality, tokens, cost, and latency.
- Stretch: keep a stable system-prompt prefix and measure the latency gain from Ollama reusing its warm model and KV cache (`keep_alive`). Read how paid APIs turn the same idea into **prompt caching** discounts.

**Definition of Done**
- [ ] Long-term memory works, and the user can inspect or delete it
- [ ] Token reduction from context management is measured
- [ ] Single vs multi-agent comparison table + verdict

---

## M5: Evaluation and observability (Day 3 afternoon, ~4 h)

**Goal:** measure the agent **like a product**: success rate, trajectory quality, cost, latency, with full traces I can inspect.

**Why now:** I've built several variants (scratch, framework, single, multi, API, local). Without systematic evaluation, picking one is guesswork. In 2026, the **hard problem with agents is reliability, not capability**, and evaluation is how teams achieve reliability.

**Concepts to learn**
- Levels of agent evaluation: **final outcome** (task success), **trajectory** (right tools, right order, no unnecessary steps), **single-step** (tool-argument correctness)
- Deterministic checks (exact SQL results, required citations) vs LLM-as-judge (rubrics), and calibrating the judge against my own labels
- Non-determinism: run each task **k times** and report the success rate and variance (pass@k vs pass^k)
- Observability: traces and spans per LLM call / tool call; the **OpenTelemetry GenAI semantic conventions**; cost and latency dashboards
- Regression testing: a fixed eval set run on every change (linking to CI in project 05)

**What to build**
- Core: expand the M0 task list to **25–30 tasks** (including ambiguous ones, impossible ones the agent should refuse, and ones that need approval), each with a **grader** (deterministic where possible, rubric-based LLM-judge otherwise).
- Core: an **eval runner** that runs each task **3×**, reporting success rate, tool-selection accuracy, average steps, tokens, cost, and p50/p95 latency per variant.
- Core: instrument the agent with **Langfuse or MLflow tracing** (OpenTelemetry). Find **3 failed trajectories** in the traces and classify the root cause (wrong tool, bad arguments, hallucinated answer, gave up, loop).
- Core: fix the top failure cause (tool descriptions, prompt, schema) and **re-run** to show the improvement.
- Stretch: wire the eval suite into a GitHub Actions workflow (from project 05) as a quality gate.

**Definition of Done**
- [ ] Eval suite of 25–30 tasks with graders, 3 runs each
- [ ] A comparison table across my agent variants
- [ ] Failure analysis from traces + one measured improvement

---

## M6: Security, red-teaming, and deployment (Day 4, ~5 h)

**Goal:** make the agent **safe to expose**: resistant to prompt injection, least-privileged, budget-capped, auditable, and deployed.

**Why now:** an agent with tools is an **attack surface**. A malicious document or tool output can hijack it into leaking data or calling dangerous tools. Security comes last *in the order of building*, but it's a **release blocker**: no deployment without it.

**Concepts to learn**
- The **OWASP Top 10 for LLM Applications** (prompt injection, sensitive information disclosure, excessive agency, improper output handling, unbounded consumption…) and OWASP's guidance on **agentic AI threats**
- **Indirect prompt injection** through tool outputs and retrieved documents; the "lethal trifecta" idea (private data + untrusted content + an exfiltration channel)
- Defences: least-privilege tools, allow-lists, human approval, **separating trusted and untrusted content**, output validation, sandboxing, rate and budget limits, audit logging. Also why "just tell the model not to" is not enough.
- MCP-specific risks: tool poisoning (malicious tool descriptions), over-broad scopes, untrusted servers
- Deployment: secrets management, per-user auth, timeouts, cost caps

**What to build**
- Core: a **red-team suite** of ≥ 10 attacks: direct jailbreaks, **injected instructions inside a regulation chunk and inside a DB text field**, attempts to make `run_sql` write or drop data, attempts to trigger `request_promotion` without approval, secret-exfiltration attempts, a cost-explosion prompt (infinite loop), and a **poisoned MCP tool description**.
- Core: mitigations: a read-only DB user, a SQL allow-list/linter, approval gates (already built), output validation, a per-request **token/cost budget**, tool-output sanitisation and delimiting, and an audit log of every tool call.
- Core: re-run the red-team suite **before vs after** the mitigations, and write the **red-team report** (attack, result, mitigation, residual risk).
- Core: deploy: a Docker Compose stack on my PC (or project 01 Track A/B), with a **Cloudflare quick tunnel** for a temporary public demo URL. The stack has the agent API + MCP servers + tracing; secrets in env vars or a secret manager; a health endpoint; rate limiting.
- Core: final README + architecture diagram + eval and red-team summaries + the demo video.
- Stretch: add an input/output safety classifier model and measure its false-positive rate on my benign eval tasks.

**Definition of Done**
- [ ] Red-team report with before/after results
- [ ] No attack can perform a write/irreversible action without human approval
- [ ] The deployed stack runs from one command; the demo is recorded

---

## Self-check: interview questions

1. Agent vs workflow: give a task where an agent is the wrong choice, and say why.
2. Walk through the message flow of one tool-calling loop iteration.
3. What are MCP's three server primitives? When would I use a resource instead of a tool?
4. What changed in the 2026-07-28 MCP spec, and how does it affect how I deploy servers?
5. How did I implement human-in-the-loop, and how does checkpointing make it possible?
6. What is context engineering? Name 4 techniques I used and their measured effect.
7. When did multi-agent help or hurt in my experiments? What did it cost?
8. How do I evaluate a non-deterministic agent? What's the difference between pass@k and pass^k?
9. Explain indirect prompt injection with an example from my project, and 4 defences.
10. What is "excessive agency", and how did my tool design limit it?
11. My agent's success rate dropped from 85% to 70% after a model upgrade. How do I investigate?

## References to search
- Anthropic, **"Building effective agents"** (patterns, workflows vs agents); Anthropic engineering posts on **context engineering** and **writing tools for agents**
- OpenAI, "A practical guide to building agents"
- **modelcontextprotocol.io**: the specification, its changelog (2026-07-28 revision), and the roadmap
- The LangGraph docs (persistence, human-in-the-loop) / the PydanticAI docs
- **OWASP Top 10 for LLM Applications** + the OWASP Agentic AI threats & mitigations guide
- Simon Willison's writing on prompt injection and the "lethal trifecta"
- OpenTelemetry **GenAI semantic conventions**; the Langfuse docs
- Chip Huyen, "AI Engineering" (the agents chapter)
- Yao et al. (2022), **ReAct**; Shinn et al. (2023), **Reflexion**
