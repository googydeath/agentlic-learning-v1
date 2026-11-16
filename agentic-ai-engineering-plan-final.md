# Agentic AI Engineering: 18-Week Execution Plan (Final)

**Goal:** Reach production-grade competence in Python agentic systems, proven by a portfolio capstone covering both the harness layer and the distributed-systems layer.
**Budget:** ~170 hours at 8–10 hrs/week (17 weeks + 1 buffer week). API spend ~$100–250 (my estimate; verify current model prices).
**Approach:** Primary sources first. Build a thin capstone skeleton early, instrument it from day one, and upgrade it every block instead of studying everything and building at the end.

> Estimates are mine, not measured. Re-plan at each checkpoint. Links are ones I verified during research, unless marked "search".

**Final version: durable engine locked to Temporal. Changes since v1:** observability moved into Block B; LangGraph compressed to ~9h (saved hours moved to the capstone); prompt-cache prefix stability added to Blocks A–B; TPM/cached-token accounting check added to the gateway milestone; all Week 1 decisions consolidated in one table. Weeks after Block B shifted one week earlier; the capstone gets a seventh week.

---

## Week 1 decisions (finalize before starting)

| Decision | Choice | Notes |
|---|---|---|
| Agent framework | **Pydantic AI** (locked) | Decoding AI course already uses it; integrates with Temporal. LangGraph is a one-week comparative study only (Block C). |
| Durable engine | **[x] Temporal** (locked) | Self-hosted locally with Docker Compose: Temporal Server + UI backed by Postgres, so workflow state survives full container restarts (unlike the in-memory `temporal server start-dev`). Temporal Python SDK for workflows and activities; Pydantic AI plugs in through its Temporal durability support. |
| Models in dev | Cheap/small model for iteration; frontier model only for final benchmark runs | |
| Hard budget caps | Per-run token and dollar caps enforced in code by Week 2 | |
| Tracing stack | OpenTelemetry + single-container Arize Phoenix locally | Set up in Block B. |

---

## Timeline at a glance

| Block | Weeks | Focus | Hours |
|---|---|---|---|
| A | 1–2 | Mental models + harness part 1 (+ cache-prefix awareness) | ~19 |
| B | 3–4 | Harness part 2 + capstone skeleton + observability | ~22 |
| C | 5 | LangGraph, compressed | ~9 |
| D | 6–8 | Durable execution | ~27 |
| E | 9–10 | Security + evals | ~18 |
| F | 11–17 | Capstone build + chaos tests + write-up (includes ~8h extra buffer) | ~65 |
| Buffer | 18 | Slip, polish, publish | ~9 |

Block B is the heaviest at ~11 hrs/week. If you slip, Checkpoint 1 tells you what to drop.

---

## Block A (Weeks 1–2): Mental models + harness part 1

**Read (≈6h), in this order**
1. Building effective agents (search the title on anthropic.com/engineering)
2. Effective context engineering for AI agents (search the title on anthropic.com/engineering)
3. [Effective harnesses for long-running agents](https://anthropic.com/engineering/effective-harnesses-for-long-running-agents)
4. [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system): note the token multiples (agents ~4× chat, multi-agent ~15×) and when multi-agent is a poor fit.
5. [Demystifying evals for AI agents](https://anthropic.com/engineering/demystifying-evals-for-ai-agents)

**Build (≈12h):** [Building a Coding Agent From Scratch](https://github.com/decodingai-magazine/building-a-coding-agent-from-scratch-course), lessons 1–4 (architecture, agent loop, sandboxing, context engineering).
- Get it running with the free-tier provider first.
- Don't just read the code. Modify it.
- The course ships with Opik tracing. Keep it for the course exercises; the capstone uses OpenTelemetry (Block B).

**Cache-prefix awareness (≈1h, folded into lesson 4)**
- Provider prompt caching generally depends on a stable prompt prefix. Read each provider's caching docs for your chosen model (search "<provider> prompt caching"); I did not verify provider-specific rules or savings figures.
- While reading the compaction code, identify what it rewrites. Goal: static system prompt and tool definitions stay byte-identical at the front, and only the dynamic history is compacted.

**Deliverables**
- Fork of the repo with one custom tool and one custom permission rule (allow/ask/deny).
- A test asserting the system prompt and tool-definition prefix is unchanged after a compaction event.
- A one-page note: "what the loop, sandbox, and compaction each protect against".

**Done when:** you can explain, without looking, why compaction fires before the window limit, what the permission layer blocks, and what compaction must not touch.

---

## Block B (Weeks 3–4): Harness part 2 + capstone skeleton + observability

**Build (≈12h):** Decoding AI lessons 5–8 (subagents, remote headless mode, evals, replay).
- Run subagent fan-out on one task and read the traces.
- Record a run, then replay it with a swapped model.
- Keep each subagent's static prefix (system prompt + tools) stable and separate from its task-specific content.

**Capstone skeleton (≈6h):** in a new repo, create:
- FastAPI endpoint that accepts a "job".
- Postgres `jobs` table with a transactional outbox pattern.
- One worker that runs a **stub agent** (a fixed script) and writes results.
- Docker Compose for the whole thing: app Postgres, **Temporal Server + Temporal UI backed by Postgres** (use a separate database or schema from your app tables, with a named volume so data persists), and Phoenix (below). Follow Temporal's current docs for the Postgres-backed Docker Compose setup (search "Temporal docker-compose Postgres"); I did not verify the exact compose files or images here. Allow ~1–2h. If Checkpoint 1 shows you're behind, make this the first task of Block D instead, but the stack must be running before you write any workflow code.

**Observability (≈3–4h)**
- Add the OpenTelemetry SDK and OTLP exporter to the API and worker. Emit GenAI-convention spans: one parent span per job, one per agent invocation, one per tool call, one per model call, with token counts. See [OTel GenAI conventions](https://dev.to/gabrielanhaia/opentelemetry-genai-semantic-conventions-your-llm-traces-should-look-like-this-in-2026-3ff6). Pin library versions (the `gen_ai.*` namespace is still experimental).
- Run Arize Phoenix as a single container in Docker Compose. Check Phoenix's current self-hosting docs for the image and ports (search "Arize Phoenix self-hosting"); an OTLP endpoint is all the app needs.
- Record cached-token counts on model-call spans where the provider reports them, so cache hit rates are visible from week 4 onward.

**Deliverables:** forked harness with replay working; a skeleton that runs end to end with a fake agent; a Phoenix trace of one full job (API → outbox → worker → stub tool calls).

**Checkpoint 1 (end of Week 4):** If behind by more than 4 hours, skip Decoding AI lesson 8's extras and keep only record/replay. Never skip the tracing setup.

---

## Block C (Week 5): LangGraph, compressed (~9h)

**Study (≈3–4h)**
- [LangGraph v1 release notes](https://docs.langchain.com/oss/python/releases/langgraph-v1): `create_react_agent` is deprecated in favor of `create_agent` with middleware; graph primitives are unchanged.
- [Intro to LangGraph (LangChain Academy)](https://academy.langchain.com/courses/intro-to-langgraph), modules 1–4: skim state, reducers, subgraphs. Skip prebuilt-agent lessons.
- Use the current docs for Postgres checkpointing. I could not confirm the Academy repo covers it, and could not verify the repo's last update.

**Build (≈4–5h):** one small workflow (plan → act → human approval → finish) in LangGraph with a `PostgresSaver` checkpointer and an interrupt/resume. Kill the process at each node boundary and observe what is and isn't restored.

**Deliverable (≈1h):** a durability comparison note: what a LangGraph checkpoint saves vs. what a durable engine records, and what breaks on a mid-node kill. This note is your interview-ready evidence.

**Done when:** you can state exactly what state survives at a node boundary and what does not.

---

## Block D (Weeks 6–8): Durable execution

**Engine: Temporal, self-hosted in Docker Compose (Temporal Server + UI backed by Postgres).**

- [Pydantic AI durable execution docs](https://pydantic.dev/docs/ai/harness/durable-execution/): use the Temporal sections.
- [Durable AI agent with Gemini and Temporal](https://ai.google.dev/gemini-api/docs/temporal-example): the tutorial uses `temporal server start-dev`, which keeps state in memory by default. Run it once to learn the model, then point your worker at the Compose-hosted, Postgres-backed server for all capstone work.
- [Temporal for AI agents](https://activewizards.com/blog/indestructible-ai-agents-a-guide-to-using-temporal/): activities are at-least-once, so design every side-effecting activity for idempotency.
- Temporal's own docs for the Python SDK: determinism rules, activity retries and timeouts, and workflow versioning (search "Temporal Python SDK versioning").

**Study (≈15–20h) and build (≈8h)**
1. Wrap the skeleton's stub agent in a durable workflow.
2. Add **idempotency keys** to every side-effecting tool (PR, comment, push).
3. Implement a human-approval wait that survives a restart.
4. Learn **workflow versioning**: deploy changed code while an old run is mid-flight.
5. Kill the worker at randomized step boundaries and assert no duplicate side effects.
6. Confirm traces show workflow steps and retries (they should already be flowing to Phoenix).
7. **Restart-survival test:** start a workflow, then stop the entire Compose stack (Temporal Server, UI, Postgres) without deleting volumes, bring it back, and assert the workflow resumes and finishes with no duplicated side effects. Then repeat after killing only the worker. Both must pass.

**Deliverables:** skeleton with durable workflows, idempotent tools, and a kill-test script.

**Checkpoint 2 (end of Week 8):** The kill-at-every-step test must pass before moving on. If it doesn't, extend this block by a week and cut from Block E's reading list, not from this.

---

## Block E (Weeks 9–10): Security + evals

**Security (≈8–10h)**
- [The lethal trifecta for AI agents](https://simonw.substack.com/p/the-lethal-trifecta-for-ai-agents)
- [Design Patterns for Securing LLM Agents against Prompt Injections](https://simonwillison.net/2025/Jun/13/prompt-injection-design-patterns/)
- "Prompt Injection Isn't Fixed by a Filter. It's Fixed by Architecture" (dev.to, by Eugen Ullrich; secondary source, search the title to find the rendered post)
- [OWASP Top 10 for Agentic Applications, practical guide](https://koi.ai/blog/owasp-agentic-ai-top-10-a-practical-security-guide) (secondary; read OWASP's own publication too)
- [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) (beta, v0.1: use as a checklist, not a standard)

**Build:** a written threat model for the capstone (which trifecta legs exist, where untrusted text enters, how it's isolated) and a small injection test set (poisoned issue text, README, tool output).

**Evals (≈8–10h)**
- [Hamel Husain and Shreya Shankar's evals FAQ](https://hamel.dev/blog/posts/evals-faq/evals-faq.pdf)
- [Free 17-part evals email course](https://ai.hamel.dev/eval-course)
- [How to do agentic evals](https://theworkingprototype.substack.com/p/how-to-do-agentic-evals) (pass@k vs pass^k)

**Build:** 25–30 task golden benchmark with deterministic graders (tests fail before, pass after). Read 30+ real transcripts in Phoenix and write your own failure taxonomy before writing any LLM judge.

**Deliverables:** `THREAT_MODEL.md`, benchmark with a baseline pass@k and pass^k number.

---

## Block F (Weeks 11–17): Capstone build (~65h)

**System:** multi-tenant issue-to-PR service. FastAPI → Postgres outbox → durable workflow → orchestrator + sandboxed worker subagents → deterministic verification gate → human approval → PR.

**Sub-milestones**

| Weeks | Milestone | Hours |
|---|---|---|
| 11–12 | Replace the stub with the real harness in sandboxes (network denied, no credentials inside). Verification gate: tests must fail before and pass after. Keep static prompt/tool prefixes stable across compaction and subagent spawns. | ~20 |
| 13 | **Provider gateway:** dual RPM/TPM token bucket with reserve-then-refund, shared state (Redis or Postgres), per-tenant fair share, bounded concurrency, honor `retry-after`, jitter, capped retries. See [Rate limiting for LLMs](https://gravitee.io/corpus/gen-770/doorway-to-fame/rate-limiting-for-llms.html) and [production patterns](https://dev.to/aakash_gour/rate-limiting-llm-apis-in-production-patterns-that-dont-lose-your-data-2g18). **Before writing the accounting logic, verify how each provider you use counts cached tokens against its TPM and RPM limits** (I have not verified this; it varies by provider). Build the reservation and refund code so the cached-token rule is a per-provider setting, and test it against a fake provider for each variant. | ~12 |
| 14 | Cost-per-job reporting and dashboards from the existing traces (tokens, cached tokens, dollars, retries per job and per tenant). Memory: hybrid retrieval over the repo (BM25 + embeddings in pgvector); episodic store of past runs. | ~11 |
| 15–17 | Chaos suite (below), CI gating on pass^k, README and write-up. Includes ~8h of extra buffer for overruns. | ~22 |

### Chaos tests, in priority order

**Tier 1 (must have)**

| Failure | Injection | Must hold |
|---|---|---|
| Worker killed mid-tool-call | SIGKILL at randomized step boundaries | Resumes; zero duplicate side effects |
| Crash after effect, before result recorded | Kill between the two | Idempotency key prevents the second effect |
| 429 storm | Fake provider returns 429 with `retry-after` | No retry storm; bounded queue; other tenants unaffected |
| One tenant saturates TPM | Flood of large-context jobs | Fair share holds |
| Indirect prompt injection | Poisoned issue, README, tool output | Blocked by architecture (egress denied, no secrets), not a filter |
| Runaway loop | Tool that never converges | Hard token/step budget kills run, returns partial result |

**Tier 2 (if time allows)**
- Provider timeout or partial stream (no double billing)
- Postgres failover or lock contention (no lost jobs, no duplicate leases)
- Approval arrives after a deploy (old runs resume on their original workflow version)
- MCP tool descriptor changes after approval (hash mismatch forces re-approval)
- Compaction drops a constraint (an eval catches it), or changes the cached prefix (a test catches it)
- Model version swap (regression suite gates it)

**Deliverables:** public repo, architecture diagram, `THREAT_MODEL.md`, benchmark table (pass@k, pass^k, cost per task, cache hit rate if the provider reports it), chaos-test report, and a short "what I'd change" section.

---

## Buffer week (Week 18)
Fix flaky tests, finish the write-up, record a short demo, publish. Unused buffer goes into Tier 2 tests.

---

## If you fall behind (cut order)
1. Tier 2 chaos tests
2. Memory/hybrid retrieval depth (keep a minimal version)
3. LangGraph build exercise (keep the comparison note)
4. Benchmark size (25 → 15 tasks)

**Never cut:** tracing from Block B, the kill-at-every-step test, idempotency keys, the threat model, and pass^k measurement. These are what make the project credible.

---

## Cost control
- Dev on cheap models; per-run dollar cap enforced in code.
- Benchmark: 3–5 trials per task during development; larger sweeps only for final numbers.
- Multi-agent runs use far more tokens than chat (~15× in Anthropic's account), so default to a single agent plus subagents only where work is truly parallel.
- Prefix-stable prompts may reduce input cost where the provider supports caching. Measure your own hit rate from traces; don't assume a savings figure.
- Local Docker Compose for Postgres, Redis, Phoenix, and Temporal Server + UI (Postgres-backed). The Decoding AI course lists $30 of Modal credits.

---

## Optional extras (only after the capstone)
- The [ai-engineering-curriculum agentic track](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) as an exercise-idea checklist. Treat its content as unvetted (small repo, AI-assisted, vendor-maintained).
- Paid Maven evals course by Husain and Shreya Shankar (price not verified).

---

## Caveats
- I could not verify the LangChain Academy repo's last commit.
- Star counts for the Decoding AI repo differed between snapshots, so treat popularity as approximate.
- Several security links are secondary write-ups; confirm against the primary papers and OWASP's own pages.
- Provider prompt-caching rules and how cached tokens count toward rate limits were not researched; verify in each provider's docs.
- The Temporal Docker Compose and Postgres-persistence setup was not verified in this research; follow Temporal's current documentation for images, versions and configuration.
- Hours, weeks and dollars are planning estimates.
