# PRism: Team Plan (Om and Atharva)

PRism is a GitHub App. It triages new PRs and issues with a team of AI agents (summarize, label, find duplicates). Nothing is written to GitHub until a person approves it on a dashboard. 2 people, one phase per "day". A day can stretch over several sessions: learning comes first, so time is not a hard limit.

All hosting, data and LLM services run on **Vultr**. See [ARCHITECTURE.md](ARCHITECTURE.md) for the stack.

Personal plans (milestones + sequential action items):
- Atharva: [plan_atharva.md](plan_atharva.md)
- Om: [plan_om.md](plan_om.md)

## How we work

- Split by area, not by frontend/backend, so both of us understand the whole system.
- Each area has an **owner**, who goes deep on it and has to be able to explain it without notes.
- Learn while building: when a new concept comes up, get a short elevator explanation (what it is, why PRism needs it, one gotcha), then implement it right away. No separate study sessions or tests.
- At the end of each day, each of us explains the *other's* area on a whiteboard or call. If we can't, we go over it again before moving on.
- One branch per area (`<name>/<area>`, e.g. `atharva/ingest`), small PRs to `main`, the other person reviews. Merge often.
- Shared contracts (`packages/core`: DB models, schemas, settings) only change when we both agree.
- Each person keeps their own `.env`, never committed.
- Keep a short "what I learned / what confused me" list for the teach-back.

## Day 1: Foundations

**Goal:** opening a PR creates a queued run, and the tool server can fetch PR data.

| Part | Work | Owner |
|---|---|---|
| Accounts and keys | GitHub App, smee.io, test repo, Vultr account: Cloud Compute VM (Ubuntu + Docker), Managed PostgreSQL with pgvector enabled, Serverless Inference API key. Own `.env` each. | Both |
| Foundation | Repo skeleton, Docker Compose (Postgres + Redis for local dev), shared contracts, DB migration | Om |
| Ingest path | Webhook route, signature check, event filter, dedupe, enqueue, worker stub | Atharva |
| GitHub tool server | MCP server with read tools (get PR, changed files, diff) | Om |

## Day 2: The brain

**Goal:** a PR produces a draft comment, the run pauses, and approving it posts to GitHub.

| Part | Work | Owner |
|---|---|---|
| Tools and approval | Remaining read tools, approval gate (pause/resume), gated write tools, executor | Om |
| Agents | Summarizer, labeler, duplicate finder, supervisor (LLM via Vultr Serverless Inference) | Atharva |
| Join | Aggregator and approve endpoint | Both |

## Day 3: Product and proof

**Goal:** approve from the UI, deployed on Vultr, with eval numbers.

| Part | Work | Owner |
|---|---|---|
| Dashboard | GitHub login, runs list, run detail, approve/edit panel, live updates | Om |
| Evals and deploy | Eval sets (real PRs and prompt-injection cases), run and record numbers; deploy everything to the Vultr VM with `docker compose up` behind Caddy | Atharva |

Owners for Day 3 can swap if either of us wants to learn the other side.

## Open questions

- [ ] Auto-apply labels by default, or require approval for everything at first?
- [ ] Which public repo for the eval set (httpx, fastapi, other)?
- [ ] Domain for the VM: one we own, or a free sslip.io hostname?
- [ ] Which Vultr Serverless Inference models do we use for chat (tool calling) and embeddings? Check the current model list; the embedding size sets the `vector(N)` column.
- [ ] Redis in production: a container on the VM (default) or Vultr Managed Valkey?

## Stretch (only if time is left)

Reviewer-recommender agent, CI eval gate, chat with your repo, voice briefing.
