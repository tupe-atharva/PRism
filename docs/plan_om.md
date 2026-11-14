# PRism plan: Om

Your parts: **foundation, MCP tool server, approval gate + write tools, dashboard**, plus the GitHub App. Atharva's parts are in [plan_atharva.md](plan_atharva.md). Team rules are in [PLAN.md](PLAN.md); stack and flows in [ARCHITECTURE.md](ARCHITECTURE.md).

**How each step works:** when a step names *concepts*, Claude gives a ≤5-line elevator explanation first, then we implement. Work top to bottom; a ⏸ line means wait for Atharva before going on.

## Shared milestone map

| # | Milestone | Owner | Needs | Done when |
|---|---|---|---|---|
| M0 | Accounts and keys | Both | — | A test webhook shows in the local smee log; both laptops connect to Vultr Postgres |
| M1 | Foundation | **Om** | M0 | `docker compose up` + `alembic upgrade head` work on both laptops; CI green |
| M2 | Ingest path | Atharva | M1 | Opening a PR on the test repo creates a `queued` run and the worker logs its id |
| M3 | MCP read tools I | **Om** | M1 | MCP Inspector calls `get_pr`, `list_changed_files`, `get_pr_diff`, `get_file` on the test repo |
| — | Day 1 teach-back | Both | M2, M3 | Each explains the other's milestones with no notes |
| M4 | MCP read tools II | **Om** | M3 | `list_labels` and `search_similar` work in MCP Inspector |
| M5 | Agents | Atharva | M2, M3, M4 | A PR produces a draft comment + proposed actions in the DB and a full LangSmith trace |
| M6 | Approval + writes | **Om** | M4 | Approving with curl posts a comment; a write with no valid approval is refused |
| M7 | Join | Both | M5, M6 | PR → `awaiting_approval` → curl approve → comment on GitHub |
| — | Day 2 teach-back | Both | M7 | |
| M8 | Dashboard | **Om** | M7 | Approve from the UI; new runs appear live |
| M9 | Evals | Atharva | M7 | README has the results table |
| M10 | Deploy | Atharva | M8, M9 | A real PR gets triaged in production |
| — | Day 3 teach-back | Both | M10 | |

M1 blocks Atharva's M2, so do it first and keep its PR small.

---

## M0: Accounts and keys (Day 1, about 1 hr, in parallel with Atharva)

Atharva does Vultr and smee. You do GitHub.

1. [ ] Create a test repo (e.g. `prism-sandbox`) with a few labels (`bug`, `enhancement`, `docs`, `tests`) and 3–4 sample issues.
2. [ ] Register a GitHub App (Settings → Developer settings → GitHub Apps). Permissions: Pull requests read/write, Issues read/write, Contents read, Metadata read. Events: Pull request, Issues. *Concepts: GitHub Apps vs OAuth apps, permissions*
3. [ ] Set the webhook URL to Atharva's smee URL and generate a webhook secret.
4. [ ] Download the private key (`.pem`), note the App ID, and install the app on the test repo; note the installation id.
5. [ ] In the same app, turn on "Request user authorization (OAuth)" and note the client id + secret (for dashboard login in M8). *Concepts: GitHub App user auth*
6. [ ] Send Atharva the App ID and installation id in `#nov14_building`; send the webhook secret, private key and client secret in a DM only.

## M1: Foundation (Day 1, about 2 hrs)

7. [ ] `git checkout main && git pull && git checkout -b om/foundation`
8. [ ] Create a uv workspace: root `pyproject.toml` with members `apps/api`, `apps/worker`, `apps/mcp`, `packages/core`; Python 3.12. *Concepts: monorepo, uv workspaces*
9. [ ] Create `infra/docker-compose.dev.yml` with `pgvector/pgvector:pg16` (port 5432, named volume) and `redis:7` (port 6379). *Concepts: Docker Compose, volumes, ports*
10. [ ] Create `packages/core/settings.py` with pydantic-settings: `DATABASE_URL`, `REDIS_URL`, `GITHUB_APP_ID`, `GITHUB_PRIVATE_KEY_PATH`, `GITHUB_WEBHOOK_SECRET`, `VULTR_INFERENCE_BASE_URL`, `VULTR_INFERENCE_API_KEY`, `CHAT_MODEL`, `EMBEDDING_MODEL`, `EMBEDDING_DIM`, `MCP_URL`, `MCP_SERVICE_TOKEN`, `LANGSMITH_API_KEY`. Add `.env.example` with the same keys and no values. *Concepts: config from env vars*
11. [ ] Create `packages/core/db.py`: async SQLAlchemy engine + `get_session()`. *Concepts: async SQLAlchemy, sessions*
12. [ ] Create `packages/core/models.py` with the 7 tables from ARCHITECTURE.md (`installations`, `repos`, `triage_runs`, `agent_steps`, `proposed_actions`, `approvals`, `embeddings` with `Vector(EMBEDDING_DIM)`); `installation_id` on every tenant row. *Concepts: ORM models, foreign keys, jsonb*
13. [ ] Create `packages/core/schemas.py` (shared contracts): `RunStatus` enum, `PRSummary`, `LabelSet`, `DuplicateMatch`, `ProposedAction`. **Agree on them with Atharva before merging.** *Concepts: Pydantic models as contracts*
14. [ ] Create `packages/core/llm.py`: `chat_model()` (OpenAI-compatible client on the Vultr base URL) and `embed(texts) -> list[list[float]]`.
15. [ ] Run `alembic init migrations`; set it up for async + `models.py`; first migration runs `CREATE EXTENSION IF NOT EXISTS vector`, creates all 7 tables and an HNSW index on `embeddings.vector`. *Concepts: migrations, Alembic revisions, HNSW*
16. [ ] Add ruff + pytest config and `.github/workflows/ci.yml` running `ruff check` and `pytest` on PRs. *Concepts: CI*
17. [ ] Add a "Local setup" section to `README.md`: `uv sync`, `docker compose -f infra/docker-compose.dev.yml up -d`, `alembic upgrade head`.
18. [ ] Open a PR titled `M1: foundation`; Atharva reviews and runs the setup on his laptop. Merge as soon as it's green (it unblocks him).

## M3: MCP read tools I (Day 1, about 2–3 hrs)

19. [ ] `git checkout main && git pull && git checkout -b om/mcp-read`
20. [ ] Create `packages/core/github_auth.py`: sign a JWT with the app private key (RS256, `iat` = now − 60 s, `exp` = now + 10 min), exchange it at `POST /app/installations/{id}/access_tokens`, cache the token in Redis for 50 min under `ghtoken:<installation_id>`. *Concepts: JWT, installation tokens, caching*
21. [ ] Create `apps/mcp/server.py` with FastMCP over streamable HTTP; reject requests without `Authorization: Bearer <MCP_SERVICE_TOKEN>`; read the installation id from a request header. *Concepts: MCP, JSON-RPC, streamable HTTP*
22. [ ] Tool `get_pr(repo, number)`: title, body, author, base/head, state, via githubkit. *Concepts: githubkit*
23. [ ] Tool `list_changed_files(repo, number)`: filename, status, additions, deletions.
24. [ ] Tool `get_pr_diff(repo, number, page=1)`: request with `Accept: application/vnd.github.diff`; drop lockfiles, `dist/` and generated files; redact strings that look like secrets; split into ~12k-token pages and return `page X/Y`. *Concepts: content negotiation, pagination*
25. [ ] Tool `get_file(repo, path, ref)`: file contents at the PR's head sha, capped in size.
26. [ ] Tests in `apps/mcp/tests/` using recorded GitHub responses: auth rejected without a token, diff filtering drops `package-lock.json`, paging returns the right page count.
27. [ ] Run the server, open it in MCP Inspector (`npx @modelcontextprotocol/inspector`), and call every tool on a test-repo PR.
28. [ ] Open a PR titled `M3: MCP read tools`; Atharva reviews.

⏸ **Day 1 teach-back** (30 min): you explain Atharva's M2 ingest path with no notes; he explains your M1 + M3.

## M4: MCP read tools II (Day 2, about 1 hr; do this first, Atharva's labeler waits on it)

29. [ ] `git checkout main && git pull && git checkout -b om/mcp-read-2`
30. [ ] Tool `list_labels(repo)`: name + description for every repo label.
31. [ ] Tool `search_similar(repo, text, k=5)`: `embed(text)` with `packages/core/llm.py`, then a pgvector cosine search on `embeddings` filtered by `repo_id`; return number, title, kind, score. *Concepts: vector search, cosine distance operator*
32. [ ] Tests for both, then open a PR titled `M4: MCP read tools II` and tell Atharva it's merged.

## M6: Approval gate + write tools (Day 2, about 3 hrs)

33. [ ] `git checkout main && git pull && git checkout -b om/approval`
34. [ ] Create `apps/worker/graph/checkpoint.py`: `get_checkpointer()` returning LangGraph's async `PostgresSaver` on the same DB; call `.setup()` once. Tell Atharva so `run_triage` uses it. *Concepts: checkpoints, durable execution*
35. [ ] Create `apps/worker/graph/policy_gate.py`: read `repos.config` (parsed `.prism.yml`); labels with confidence ≥ 0.8 auto-apply; comments and reviewer requests need approval. For those, set the run to `awaiting_approval`, publish `{"run_id": ...}` on Redis channel `runs:<installation_id>`, and call `interrupt()`. *Concepts: `interrupt()`, human-in-the-loop, pub/sub*
36. [ ] In the MCP server, add write tools `add_labels(repo, number, labels, approval_id)` and `post_comment(repo, number, body, approval_id)`. Each one checks in Postgres that the approval is `approved`, `used_at` is null, and repo + PR + `payload_hash` match. Only then call GitHub and set `used_at`. Reject everything else. *Concepts: capability tokens, hashing*
37. [ ] Make write tools need a separate scope on the service token, so agents (read scope) can't call them.
38. [ ] Create `apps/worker/graph/executor.py`: plain code (no LLM) that calls the write tools for each approved or auto-applied action and sets the run to `done`.
39. [ ] Add arq job `resume_triage(ctx, run_id)`: resume the graph with `Command(resume=decision)` on `thread_id=run_id`. *Concepts: `Command`, resuming a graph*
40. [ ] Handle `synchronize`: a new push cancels any run still `awaiting_approval` for that PR (status `cancelled`).
41. [ ] Tests: `post_comment` with no approval → refused; reused approval → refused; body changed after approval (hash mismatch) → refused.
42. [ ] Open a PR titled `M6: approval + writes`; Atharva reviews.

## M7: Join (Day 2, about 1–2 hrs, with Atharva)

43. [ ] Add `POST /runs/{id}/approve` in `apps/api/routes/runs.py`: body `{approved_action_ids, edited_comment}`; write `approvals` rows (recompute `payload_hash` if the comment was edited); enqueue `resume_triage(run_id)`.
44. [ ] End to end with Atharva: open a PR → `awaiting_approval` → `curl -X POST /runs/<id>/approve` → the comment and labels appear on GitHub.

⏸ **Day 2 teach-back** (30 min): you explain Atharva's M5 agents; he explains your M4 + M6.

## M8: Dashboard (Day 3, about 3–4 hrs)

45. [ ] `git checkout main && git pull && git checkout -b om/dashboard`
46. [ ] API read endpoints in `apps/api/routes/runs.py`: `GET /runs` (filtered by the user's installations), `GET /runs/{id}` (run + `agent_steps` + `proposed_actions`).
47. [ ] `GET /runs/stream`: an SSE endpoint that subscribes to `runs:<installation_id>` on Redis and streams events. *Concepts: SSE, Redis pub/sub*
48. [ ] `npx create-next-app@latest apps/web` (TypeScript, App Router, Tailwind); add shadcn/ui. *Concepts: App Router, server vs client components*
49. [ ] Auth.js with the GitHub provider using the app's OAuth client id/secret; protect every route except `/login`. *Concepts: OAuth, sessions*
50. [ ] Web → API auth: the Next.js server sends a short-lived signed JWT (GitHub user id + installation ids); FastAPI verifies it on every `/runs` endpoint.
51. [ ] Page `/runs`: table of runs (repo, PR #, status, cost, latency), newest first; a client component listens to `/runs/stream` and adds new rows live.
52. [ ] Page `/runs/[id]`: PR summary, an agent-step timeline from `agent_steps`, and a list of proposed actions.
53. [ ] Approve/edit panel on `/runs/[id]`: checkboxes per action, an editable comment textarea, an Approve button that calls `POST /runs/{id}/approve`.
54. [ ] `apps/web/Dockerfile` (Next.js standalone output); tell Atharva it's ready for deploy.
55. [ ] Open a PR titled `M8: dashboard`; Atharva reviews.

⏸ **Day 3 teach-back** (30 min, after Atharva's M10): you explain Atharva's M9 evals + M10 deploy; he explains your M8 dashboard.
