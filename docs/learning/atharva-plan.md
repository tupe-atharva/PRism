# Atharva's PRism learning plan

Follows the [team plan](../PLAN.md). I own the ingest path, the agents, and evals + deploy. Om owns the skeleton, MCP server, approval gate and dashboard, which I explain back each day.

**Approach: learn while building.** When a new concept comes up, Claude gives an elevator explanation first, then we go straight into implementing it. There are no separate study sessions or self-tests. The implementation is the practice.

**Elevator format (at most 5 lines):**
1. What it is, in one sentence
2. Why PRism needs it, right here
3. The one gotcha to watch for
4. Where it shows up in the code we're about to write

## At a glance

| Day | Goal | I own (go deep) | Om owns (I explain back) |
|---|---|---|---|
| 1: Foundations | Opening a PR creates a queued run; tool server fetches PR data | Ingest path: webhook, signature check, event filter, dedupe, enqueue, worker stub | Repo skeleton, Docker, shared contracts, DB migration, MCP read tools |
| 2: The brain | Draft comment, run pauses, approving posts to GitHub | Agents: summarizer, labeler, duplicate finder, supervisor | Remaining tools, approval gate, write tools, executor |
| 2: Join | Same | Aggregator + approve endpoint, with Om | Same, with me |
| 3: Product and proof | Approve from UI, deployed on Vultr, eval numbers | Evals and deploy | Dashboard |

Each step below lists the concepts it brings up. Each one gets its elevator explanation when we reach that step.

---

## Day 1: ingest path

Done when opening a PR on the test repo creates a queued run in the database.

**Accounts (with Om, about 1 hr)**
- [ ] Vultr account access; create a Serverless Inference API key
- [ ] Vultr Managed PostgreSQL connection string (pgvector enabled) into my `.env`
- [ ] Join the smee.io channel and run the smee client locally
- [ ] GitHub App id, private key and webhook secret into my `.env` (never committed)

**Build (me, about 3–4 hrs)**
- [ ] Wait for Om's skeleton + `packages/core` contracts, then `git checkout -b atharva/ingest`
- [ ] Webhook route: *concepts: FastAPI routes, status codes 202/204/401, async Python, webhooks + smee*
- [ ] HMAC check: *concepts: HMAC-SHA256 over the raw body, `hmac.compare_digest`*
- [ ] Event filter: *concepts: GitHub event types and actions*
- [ ] Redis dedupe: *concepts: idempotency, `SET NX EX`, GitHub retries*
- [ ] Insert `triage_runs` (queued) + enqueue `run_triage(run_id)`: *concepts: job queues, arq producer/consumer*
- [ ] Worker stub that logs the run id and marks it `running`
- [ ] Tests for the route: valid signature → 202, bad → 401, ignored event → 204, repeated delivery → 200 with no new job
- [ ] Open a real PR on the test repo and watch it reach the worker
- [ ] PR to `main`; Om reviews

**End of day (30 min)**
- [ ] Update my "what I learned / what confused me" list
- [ ] Teach-back: I explain Om's skeleton, migration and MCP server; Om explains my ingest path

---

## Day 2: agents

Done when a PR produces a draft comment, the run pauses, and approving it posts to GitHub.

**Build (me, about 4–5 hrs)**
- [ ] `git checkout -b atharva/agents`
- [ ] Summarizer alone first (`get_pr`, `list_changed_files`, `get_pr_diff`, `get_file` → `PRSummary`): *concepts: LLM tool calling, Vultr's OpenAI-compatible API, ReAct loop, LangGraph `StateGraph` + `ToolNode`, MCP client (`MultiServerMCPClient`, `bind_tools`), structured output*
- [ ] Check its LangSmith trace: *concepts: tracing*
- [ ] Labeler (`list_labels`, `list_changed_files` → `LabelSet`): *concepts: runtime Pydantic enum*
- [ ] Duplicate finder (`search_similar` + LLM check on top 3): *concepts: embeddings, pgvector, cosine similarity, HNSW*
- [ ] Supervisor with plain-code routing + `Send` fan-out, 6-tool-call cap, `truncated=true`: *concepts: parallel subgraphs, state merging*
- [ ] `<untrusted>` tags around PR content: *concepts: prompt injection*
- [ ] Every step written to `agent_steps`
- [ ] Tests: each agent on 2–3 saved PR fixtures, LLM mocked
- [ ] PR to `main`; Om reviews

**Join (with Om, about 1–2 hrs)**
- [ ] Aggregator: merge outputs into one markdown comment + proposed actions
- [ ] `POST /runs/{id}/approve` → enqueue `resume_triage`
- [ ] End to end: open PR → `awaiting_approval` → approve with curl → comment on GitHub

**End of day (30 min)**
- [ ] Teach-back: I explain Om's approval gate, write tools and executor; Om explains my agents

---

## Day 3: evals and deploy

Done when a real PR gets triaged in production and the README has a results table.

**Evals (me, about 2–3 hrs)**
- [ ] `git checkout -b atharva/evals`
- [ ] Script: pull 30 merged PRs (labels + reviewers) into `evals/prs.jsonl`: *concepts: eval sets and ground truth*
- [ ] 10 prompt-injection PRs in `evals/injection.jsonl`
- [ ] Eval runner: label F1, duplicate recall@3, summary faithfulness, injection writes (must be 0), cost per PR, p95 latency: *concepts: precision/recall/F1, recall@k, LLM-as-judge, LangSmith datasets*
- [ ] Record the numbers in a README results table

**Deploy to Vultr (me, about 1–2 hrs)**
- [ ] Vultr firewall group on the VM: allow 22, 80, 443 only: *concepts: SSH, firewalls*
- [ ] Managed PostgreSQL: add the VM's IP as a trusted source: *concepts: trusted sources, SSL connections*
- [ ] On the VM: clone the repo, production `.env` on the host, `docker compose up -d`: *concepts: images vs containers, secrets in production*
- [ ] Caddy serves HTTPS on our domain or an sslip.io hostname: *concepts: reverse proxy, automatic HTTPS*
- [ ] Point the GitHub App webhook at production; install PRism on its own repo and open a test PR

**End of day (30 min)**
- [ ] Teach-back: I explain Om's dashboard; Om explains my evals and deploy

---

## Teach-back prep: Om's areas

About 30 min per day reading Om's PR with Claude: "Give me the elevator explanation of each concept in this PR, then walk me through the code."

| Day | Om's area | Concepts I must explain | My teach-back question |
|---|---|---|---|
| 1 | Repo skeleton + Docker | Monorepo layout, Dockerfile vs Compose, service networking, volumes | How does the API container reach Postgres? |
| 1 | Shared contracts + migration | `packages/core`, SQLAlchemy 2 models, Alembic revisions, the 7 tables, `installation_id` on every row | What happens when we add a column? |
| 1 | MCP read tools | FastMCP, streamable HTTP, JSON-RPC `tools/list` + `tools/call`, installation tokens (JWT → token, 50-min cache), diff paging | Walk through one `get_pr_diff` call end to end |
| 2 | Approval gate | `interrupt()`, `Command(resume=...)`, `PostgresSaver` checkpoints, `.prism.yml` policy | Why can a run resume after a crash? |
| 2 | Write tools + executor | `approval_id` check: approved, unused, matches body hash; executor is code, not an LLM | Why can't an injected agent post a comment? |
| 3 | Dashboard | Next.js App Router, server vs client components, Auth.js GitHub OAuth, SSE + Redis pub/sub | How does a new run card appear without a refresh? |

## How I use Claude Code

1. **New concept:** Claude gives the elevator explanation (format above), then we implement it right away.
2. **Build time:** output style **Learning** (in `/config`), so Claude leaves `TODO(human)` gaps for core logic I write.
3. **New concept in the repo:** plan mode first, accept each edit by hand. Auto mode only for boilerplate I understand.
4. **Every task:** one branch per area, `/clear` between tasks, `/code-review` before opening a PR.
5. **After each day:** add any rule Claude broke twice to `CLAUDE.md`; log what confused me for the teach-back.
