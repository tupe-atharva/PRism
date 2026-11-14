# PRism

GitHub App that triages PRs and issues with LangGraph agents; writes only after human approval. All hosting, DB and LLM on Vultr.

- Plan: [docs/PLAN.md](docs/PLAN.md)
- Architecture: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- Learning plans: [docs/learning/](docs/learning/)

## How to work with us

We are learning backend, LangGraph and Next.js while building.

- When a task uses a concept we haven't met yet in this repo, start with an **elevator explanation** (at most 5 lines): what it is, why PRism needs it here, one gotcha, where it shows up in the code. Then implement right away. No quizzes or separate study steps.
- Leave `TODO(human)` for the core logic of a new concept. Write the boilerplate yourself.
- Stay inside the current task's area. Don't touch other areas' files.
- Shared contracts in `packages/core` change only when both of us agree. Ask before editing them.
- Never commit `.env` or secrets.
