# PRism Architecture

PRism is a GitHub App. When a PR or issue is opened on a repo where it's installed, LangGraph agents summarize the change, pick labels and find duplicates. Every GitHub action goes through an MCP tool server, and nothing is written to GitHub until a person approves it on a Next.js dashboard.

Everything runs on **Vultr**: one Cloud Compute VM, a Managed PostgreSQL cluster, and Serverless Inference for the LLM.

## Tech stack

| Layer | Pick | Why |
|---|---|---|
| Frontend | Next.js 15 (App Router), TypeScript, Tailwind, shadcn/ui | Runs in Docker on the Vultr VM behind Caddy |
| Auth | Auth.js with GitHub OAuth | Users log in with the account the app is installed on |
| API | Python 3.12, FastAPI, Pydantic v2, uv | Async webhooks + SSE streaming |
| Queue | arq on Redis (container on the Vultr VM) | Webhooks must answer in < 10 s; agents take 20–60 s, so they run in a worker |
| Agents | LangGraph (supervisor + parallel subgraphs via `Send`) | Fan-out, `interrupt()` for human approval, checkpointed resume |
| LLM | Vultr Serverless Inference (OpenAI-compatible API, tool calling) for chat + embeddings | Stays on Vultr; the OpenAI client works with a Vultr `base_url` |
| Tools | MCP server (official `mcp` Python SDK, FastMCP, streamable HTTP) + `langchain-mcp-adapters` | Tools are a separate service with their own auth |
| GitHub | GitHub App (JWT → installation tokens), githubkit | Per-repo permissions, higher rate limits |
| Database | Vultr Managed PostgreSQL + pgvector, SQLAlchemy 2 + Alembic | One DB for runs, approvals, LangGraph checkpoints (`PostgresSaver`) and embeddings; managed backups |
| Observability | LangSmith tracing (free tier) + structured JSON logs | Every LLM and tool call traced |
| Evals / CI | pytest + LangSmith evaluators, GitHub Actions | Ruff, tests, eval gate on PRs |
| Hosting | One Vultr Cloud Compute VM, Docker Compose: caddy, web, api, worker, mcp, redis | Always on (no cold starts vs GitHub's 10 s timeout); deploy = `docker compose up -d` |
| Local dev | Docker Compose + smee.io webhook proxy | Real GitHub webhooks hitting localhost |

Not Vultr, and still used: GitHub (the product itself), smee.io (local dev only), LangSmith (tracing).

## Services

```
GitHub ──webhook──▶ Caddy ──▶ api (FastAPI) ──enqueue──▶ redis ──▶ worker (LangGraph)
                      │            │                                   │
                      └──▶ web     └──── SSE ◀── pub/sub ──────────────┤
                       (Next.js)                                       │ tool calls (MCP)
                                                                       ▼
                    Vultr Managed PostgreSQL ◀──────────────────── mcp (FastMCP) ──▶ GitHub API
                    (runs, approvals, checkpoints, embeddings)
                                         Vultr Serverless Inference ◀── worker (LLM calls)
```

The API and the MCP server also read Postgres: the API to list runs, the MCP server to check approval ids before any write.

## Agents

Routing is plain code. LLMs are only used where judgment is needed. Each specialist is a small ReAct loop with only the read tools it needs, at most 6 tool calls, returning a Pydantic object.

| Node | Type | MCP tools | Returns |
|---|---|---|---|
| Supervisor | Code | none | Which specialists to run (PR → all, issue → labeler + duplicates) |
| Summarizer | LLM agent | `get_pr`, `list_changed_files`, `get_pr_diff`, `get_file` | `PRSummary {what, why, risk, risky_files[], test_coverage_note}` |
| Labeler | LLM agent | `list_labels`, `list_changed_files` | `LabelSet {labels[], confidence, reason}`, labels restricted to the repo's labels via a runtime enum |
| Duplicate finder | Code + LLM check | `search_similar` (pgvector) | Up to 3 related issues/PRs with score + one-line reason |
| Reviewer recommender (stretch) | Code score + LLM tie-break | `get_codeowners`, `get_file_authors`, `get_review_load` | Top 3 reviewers with reasons |
| Aggregator | LLM | none | One markdown triage comment + proposed actions |
| Policy gate | Code | none | Auto-apply vs needs approval (`interrupt()`) |
| Executor | Code | write tools only: `add_labels`, `post_comment`, `request_reviewers` | Audit log rows |

No model ever holds a write tool.

## Request flow: PR opened → comment posted

1. GitHub sends `POST /webhooks/github`. The API verifies HMAC-SHA256 of the raw body (`hmac.compare_digest`), else 401.
2. Filter: `pull_request` opened / reopened / synchronize, `issues` opened; everything else 204.
3. Dedupe: `SET delivery:<uuid> 1 NX EX 86400`; a retry returns 200 and does nothing.
4. Insert `triage_runs` (queued), enqueue `run_triage(run_id)`, return 202.
5. Worker mints an installation token (JWT → token, cached 50 min), starts the graph with `thread_id = run_id` and `PostgresSaver`.
6. Supervisor fans out with `Send`; agents call MCP tools; every step goes to `agent_steps`.
7. Aggregator builds the comment + actions; the policy gate applies `.prism.yml` and calls `interrupt()` for anything needing approval → `awaiting_approval`, Redis pub/sub → SSE → dashboard card.
8. Approve on the dashboard → `POST /runs/{id}/approve` → `resume_triage` → `Command(resume=decision)`.
9. Executor calls write tools with each `approval_id`; the MCP server checks it (approved, unused, matches repo/PR/body hash) before calling GitHub.

A `synchronize` event cancels any run still awaiting approval for that PR and re-triages the new commits.

## Data model

Every row carries `installation_id`, so tenants stay separate from day one.

| Table | Key columns |
|---|---|
| installations | id (GitHub installation id), account_login, created_at |
| repos | id, installation_id, full_name, config (parsed `.prism.yml`, jsonb) |
| triage_runs | id, repo_id, kind (pr/issue), number, head_sha, status (queued / running / awaiting_approval / done / failed / cancelled), cost_usd, latency_ms |
| agent_steps | run_id, agent, step, tool_name, args (jsonb), tokens_in, tokens_out, latency_ms, error |
| proposed_actions | id, run_id, type (label / comment / reviewers), payload (jsonb), payload_hash, auto_applied |
| approvals | id, action_id, approved_by, status, used_at |
| embeddings | repo_id, kind, number, title, vector(N), HNSW index. N = the chosen Vultr embedding model's dimension |

LangGraph's `PostgresSaver` creates its own checkpoint tables in the same database.

## Guardrails

- Untrusted content (titles, bodies, diffs) always sits inside `<untrusted>` tags; the system prompt says it is data, never instructions.
- Agents hold read tools only. Writes need a valid `approval_id`, checked by the MCP server against Postgres.
- Outputs are Pydantic schemas; labels are an enum built from the repo's real labels.
- Diffs over ~40k tokens: skip lockfiles, `dist/`, generated code; summarize per file, then combine.
- Limits: 6 tool calls per agent, 90 s run timeout, per-repo daily cost cap in `.prism.yml`.

## Production security on Vultr

- Vultr firewall group on the VM: only ports 22, 80, 443.
- Managed PostgreSQL: trusted sources = the VM's IP only; SSL connections.
- Secrets live in the `.env` on the VM, never in git.
