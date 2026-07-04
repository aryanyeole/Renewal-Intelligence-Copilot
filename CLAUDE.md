# CLAUDE.md

This file is the working contract between the human maintainer and any AI coding
agent (Claude Code, Cursor, etc.) contributing to this repository. Read it fully
before making changes. If a rule here conflicts with something you'd otherwise
do, the rule wins — ask the maintainer before deviating.

---

## 1. What this project is

**Harper Renewal Intelligence Copilot** — a workbench for a commercial insurance
brokerage's account managers to handle renewals at scale.

Every commercial policy expires every 12 months. Renewals are the hardest
workflow in a brokerage to scale because each one requires judgment: has the
business changed, does the current carrier still want the risk, is the
customer likely to leave, and what's the right play (renew in place, targeted
remarket, or full re-shop). This copilot answers those four questions per
policy via multi-agent LLM orchestration, produces a **renewal brief**
(an underwriting-memo-style document), and drafts the customer conversation.
The account manager reviews and approves.

This is a portfolio / proof-of-work project targeting a **Forward Deployed
Engineer / Member of Technical Staff** role at Harper Insure. Design and code
decisions should reflect the concerns of a real production LLM application at
an AI-native brokerage, not a hackathon demo.

**Why renewals specifically, and not intake:** Harper's public materials say
their AI already handles intake, submission routing, and underwriter
follow-ups. Renewals are the workflow they'll need to scale next as their book
grows from 6,000 to 15,000+ policies. Building for the next constraint, not
the current one, is the point.

## 2. Non-negotiables (do not violate without explicit approval)

1. **Type safety end to end.** Backend uses Pydantic v2 models. Frontend types
   are generated from the FastAPI OpenAPI schema via `openapi-typescript`, not
   hand-written. Never let the two drift.
2. **Structured LLM output only.** Every LLM call that produces data for the
   UI or another agent uses Anthropic tool use for schema enforcement. Never
   `json.loads` a raw text completion in production code paths.
3. **Agents are typed and composable.** Each of the four renewal agents
   (change analysis, appetite scoring, churn prediction, synthesis) takes a
   typed input and returns a typed output. They compose into the brief via a
   thin orchestrator, not via prompt chaining inside one giant prompt.
4. **Evals gate prompt changes.** Any change to an agent prompt, tool schema,
   or the synthesis logic must be accompanied by an eval run showing pass rate
   did not regress. See `evals/README.md`.
5. **LLM calls are always traced.** Every model call goes through the
   `apps/api/app/llm/client.py` wrapper, which emits an OTel span and a
   Langfuse trace. Do not call the Anthropic SDK directly from route handlers
   or agents.
6. **No secrets in the repo.** All keys via `.env` (gitignored) and
   `.env.example` (committed, no real values). CI uses GitHub secrets.
7. **Migrations, not `create_all`.** Every schema change is an Alembic
   migration. `Base.metadata.create_all` is banned outside of test fixtures.
8. **No `any` in TypeScript.** Use `unknown` and narrow. `// eslint-disable`
   requires a comment explaining why.
9. **Async all the way down.** FastAPI routes are `async def`, DB sessions use
   `AsyncSession`, LLM calls use the async Anthropic client, agent
   orchestration uses `asyncio.gather` where agents are independent.
10. **Synthetic data only.** This project ships with a seed script that
    generates a plausible book of business. No real customer data ever enters
    the repo, no scraping real Harper systems.

## 3. Architecture

Monorepo managed by **pnpm workspaces** (JS/TS) + **uv** (Python).

```
harper-renewal-copilot/
├── apps/
│   ├── web/     → Next.js 14 App Router, TypeScript strict, Tailwind,
│   │             shadcn/ui, TanStack Query, Zod. Portfolio dashboard +
│   │             per-policy brief view. Talks to api via typed client.
│   └── api/     → FastAPI, Python 3.12, async SQLAlchemy 2.0, Pydantic v2,
│                 Alembic. Owns the data model, agent orchestration, and
│                 brief generation.
├── packages/
│   ├── shared-types/  → auto-generated TS types from api's OpenAPI schema
│   └── eslint-config/ → shared lint config
├── evals/       → LLM eval harness. Labeled examples of renewal briefs +
│                 graders that measure agent-level correctness (did we
│                 correctly flag material change, correctly recommend
│                 remarket, correctly predict churn risk band).
├── scripts/
│   └── seed.py        → generates ~40 synthetic businesses, ~80 policies,
│                        ~25 claims, ~15 change events, ~10 carrier appetite
│                        signals. Deterministic (seeded RNG) so demos are
│                        reproducible.
├── infra/
│   ├── docker/          → Dockerfiles for api and web
│   └── docker-compose.yml → local Postgres + Redis + api + web + langfuse
├── .github/workflows/   → CI pipelines
├── Makefile             → dev, seed, test, eval, lint, typecheck, format, migrate
├── CLAUDE.md            → this file
└── README.md
```

### Data model (Postgres)

Rich enough to model a real book of business, small enough to seed and demo:

- **businesses** — id, name, industry, naics_code, state, employee_count,
  annual_revenue, founded_at
- **policies** — id, business_id, line_of_business (GL, WC, Auto, Cyber,
  Property, E&O, Umbrella), carrier_id, effective_date, expiration_date,
  premium, limits (JSONB), retentions (JSONB), status (active, non_renewed,
  cancelled)
- **claims** — id, policy_id, occurred_at, reported_at, type, amount_paid,
  amount_reserved, status (open, closed, denied)
- **carriers** — id, name, e_and_s (bool), lines_written (array), appetite
  (JSONB describing preferred classes, states, size bands)
- **carrier_appetite_signals** — id, carrier_id, signal_type (hardening,
  softening, non_renewal_wave, new_program), affected_classes, affected_states,
  effective_date, source (in the demo: seeded; in prod: bulletin scrape)
- **change_events** — id, business_id, event_type (added_vehicle,
  new_location, revenue_growth, employee_growth, business_pivot,
  ownership_change), detail (JSONB), occurred_at
- **renewal_briefs** — id, policy_id, status (pending, generating, ready,
  approved), material_changes (JSONB from change-analysis agent),
  carrier_appetite_assessment (JSONB from appetite agent), churn_risk (JSONB
  from churn agent), recommendation (enum: renew_in_place,
  targeted_remarket, full_remarket, exit), talking_points (text), created_at,
  approved_at
- **brief_generation_jobs** — id, brief_id, status, agent_states (JSONB),
  error, started_at, completed_at

### Request flow for generating a renewal brief

1. `POST /policies/{id}/briefs` → creates a `renewal_briefs` row with status
   `pending`, enqueues an Arq job, returns `202` + brief id.
2. Arq worker runs the orchestrator (see LLM section) which:
   a. Fires the three analysis agents (change, appetite, churn) in parallel
      via `asyncio.gather`. Each emits structured output via tool use.
   b. Feeds all three outputs to the synthesis agent which emits the
      recommendation + talking points.
   c. Persists agent outputs to the `renewal_briefs` row, updates status to
      `ready`.
3. Client subscribes to `GET /briefs/{id}/events` (SSE) or polls
   `GET /briefs/{id}` until status is `ready`.
4. Account manager reviews the brief in the UI, edits talking points if
   needed, hits Approve → status becomes `approved`.

Rationale for background jobs: four LLM calls (three parallel + one final)
takes 5–20 seconds. Blocking HTTP for that long is a production smell. Async
also lets us retry individual agents, cache them by input hash, and observe
each cleanly.

## 4. Tech stack — pinned choices

Do not swap any of these without a note in the PR description explaining why.

- **Runtime:** Node 20 LTS, Python 3.12
- **Package managers:** pnpm 9 (JS), uv 0.4+ (Python)
- **Backend:** FastAPI 0.115+, Pydantic 2.9+, SQLAlchemy 2.0 async,
  Alembic 1.13+, Arq 0.26+ (background jobs), Redis 7, structlog
- **Database:** Postgres 16. Local via Docker Compose. Prod via Neon or Supabase.
- **Frontend:** Next.js 14 (App Router), React 18, TypeScript 5.5 strict,
  Tailwind 3.4, shadcn/ui, TanStack Query 5, Zod 3, TanStack Table 8
- **LLM:** `anthropic` Python SDK (async), `claude-sonnet-4-5` as the default
  model (configurable via `ANTHROPIC_MODEL` env). Tool use for structured
  output on every agent.
- **Observability:** OpenTelemetry SDK, Langfuse (self-hosted in dev via
  compose, cloud in prod)
- **Auth:** Clerk (frontend + backend JWT verification)
- **Data generation:** Faker + custom domain rules in `scripts/seed.py`
- **Testing:** pytest + pytest-asyncio + httpx (backend), Vitest (frontend
  unit), Playwright (one E2E happy path)
- **Lint / format:** ruff + ruff format (Python), eslint + prettier (JS/TS),
  pre-commit hooks
- **CI:** GitHub Actions

## 5. Code conventions

### Python (apps/api, evals, scripts)

- Ruff config in `pyproject.toml` is the source of truth. Run `make lint`.
- Line length 100. Double quotes. Trailing commas in multi-line.
- Type hints on every function signature. `mypy --strict` on `app/` passes.
- Module docstrings at the top of every non-trivial module explaining *why*
  the module exists, not what it does.
- No `print()`. Use the configured `structlog` logger.
- Error handling: raise domain exceptions from `app/errors.py`; the FastAPI
  exception handler maps them to HTTP responses. Never leak SQLAlchemy or
  Anthropic exceptions to the client.

### TypeScript (apps/web, packages/*)

- `strict: true`. No `any`. Use `unknown` and narrow.
- Server components by default; `"use client"` only when you need state,
  refs, or effects.
- Data fetching: **TanStack Query** in client components, direct `fetch` in
  server components. Never call the api with unwrapped `fetch` in a client
  component.
- Validate all API responses at the client boundary with the Zod schemas in
  `apps/web/lib/schemas.ts`, which are re-exported from `packages/shared-types`.
- Components under `apps/web/components/` are presentational. Business logic
  lives in `apps/web/lib/` or in server actions.

### Naming

- Files: `kebab-case.ts` for TS, `snake_case.py` for Python.
- React components: `PascalCase.tsx`, one component per file.
- Python classes `PascalCase`, functions and variables `snake_case`.
- API routes: plural nouns, kebab-case (`/policies`, `/renewal-briefs`).

### Commits

- Conventional Commits: `feat:`, `fix:`, `chore:`, `refactor:`, `test:`,
  `docs:`, `eval:`, `seed:`.
- Present tense, imperative mood.

## 6. LLM code — house rules

The `apps/api/app/llm/` module is the only place that imports the Anthropic
SDK. Everything else goes through its wrappers.

### The four agents

Each lives in its own file under `apps/api/app/llm/agents/`, with a colocated
prompt file and tool schema. All four inherit from a shared `BaseAgent`.

1. **ChangeAnalysisAgent** — input: policy + business snapshot at bind +
   business snapshot today + change events + claims. Output: list of material
   changes, each with severity (low/medium/high) and a one-line rationale.

2. **AppetiteAgent** — input: policy + current carrier + carrier appetite
   signals for the relevant class/state + peer carriers writing similar risks.
   Output: current carrier retention outlook (stable/uncertain/at_risk),
   rate movement estimate (with band), list of viable alternative carriers.

3. **ChurnRiskAgent** — input: policy history (premium trend), claims,
   business changes, tenure. Output: churn risk band (low/medium/high),
   top three drivers, retention lever suggestions.

4. **SynthesisAgent** — input: all three agent outputs + policy summary.
   Output: recommendation enum (renew_in_place, targeted_remarket,
   full_remarket, exit) + short rationale + talking points for the
   account manager to use with the customer.

### Rules

- **Prompts live in `apps/api/app/llm/agents/<name>/prompt.py`** as Python
  constants. Not YAML, not markdown — Python, so they can be typed and
  imported.
- **Tool schemas live in `apps/api/app/llm/agents/<name>/tools.py`** and are
  the single source of truth for the agent's output shape. Pydantic models
  in `app/schemas/` are derived from these.
- **Every agent has an eval dataset** in `evals/datasets/`. If you add or
  edit an agent without a corresponding eval, CI fails.
- **Retries are the wrapper's job.** The client wrapper does exponential
  backoff on `RateLimitError` and `APIConnectionError`, up to 3 attempts.
  Individual agents do not retry.
- **Cost & latency are logged per agent** via Langfuse trace.
- **Model version is pinned** in `settings.ANTHROPIC_MODEL`, never
  hardcoded in call sites.
- **Orchestrator lives in `apps/api/app/llm/orchestrator.py`** and is a
  small piece of code: run three agents in parallel, then feed their outputs
  to the fourth. It is not itself an LLM.

## 7. Evals

Evals for a judgment task are harder than evals for extraction. We treat
them as tests, not research.

- `evals/datasets/` — separate JSONL file per agent, plus one combined
  file for the synthesis end-to-end. Each line has `{input, expected}`.
- `evals/graders/` — a grader is `(prediction, expected) -> Score`.
  Structure of graders:
    - **Deterministic where possible:** did the change agent flag the
      material change we expected? Did the synthesis agent output a
      recommendation from the allowed enum? Are all required fields present?
    - **LLM-as-judge for the fuzzy parts:** are the talking points relevant
      and non-generic? Is the rationale grounded in the inputs?
    - Deterministic checks always run before LLM-as-judge on the same example.
- `make eval` runs the full suite. `make eval-smoke` runs a small subset
  used in CI.
- Every eval run writes a JSON report to `evals/runs/` (gitignored) with
  pass rate, per-example scores, cost, and latency percentiles.
- **Regression rule:** a PR that drops the smoke pass rate below the baseline
  in `evals/BASELINE.json` fails CI.
- Baseline is updated only in a PR whose sole purpose is updating the
  baseline, with a written justification.

## 8. Testing

- Unit tests colocated: `foo.py` → `test_foo.py` in the same directory
  (Python) or `Foo.tsx` → `Foo.test.tsx` (React).
- Backend tests use a fresh Postgres schema per test module via a fixture,
  not SQLite.
- LLM calls are **always mocked in unit tests**. Real LLM calls only happen
  in the eval suite.
- Agents have unit tests that mock the Anthropic client and verify the
  agent constructs the right tool-use request from a given input.
- The orchestrator has a test that verifies parallel dispatch and correct
  synthesis with mocked agents.
- Minimum bar for a PR: new/changed code has tests; `make test` passes
  locally and in CI.

## 9. How to run things

All commands are in the `Makefile`. If you find yourself typing a long command
twice, add a make target for it.

```
make dev         # docker compose up (postgres, redis, langfuse) + api + web
make seed        # wipe + reseed the database from scripts/seed.py
make test        # backend + frontend unit tests
make eval        # full eval suite
make eval-smoke  # small subset used in CI
make lint        # ruff + eslint
make typecheck   # mypy --strict + tsc --noEmit
make migrate     # alembic upgrade head
make migration name="add_change_events"  # generate new migration
make format      # ruff format + prettier
```

Local URLs (once `make dev` is running):
- Web:      http://localhost:3000
- API:      http://localhost:8000  (docs at /docs)
- Langfuse: http://localhost:3001
- Postgres: localhost:5432 (user/pass in `.env.example`)

## 10. Working with Claude Code — behavior rules

- **Ask before scope-creeping.** If a task can be done in the requested scope
  or in a broader refactor, do the requested scope and *mention* the broader
  refactor as a follow-up. Do not silently expand.
- **Run tests after non-trivial changes.** `make test` is cheap.
- **Prefer editing over creating.** If a file exists that plausibly belongs to
  the change, edit it rather than creating a parallel one.
- **When adding a dependency, justify it in the PR description.** Bundle size
  (frontend) and install time (backend) both matter.
- **Update this file when conventions change.** If you introduce a new pattern
  you expect future work to follow, add it here in the same PR.
- **Ask for clarification when the request is ambiguous.** Better to ask one
  question than to build the wrong thing.
- **Do not fabricate domain knowledge.** If unsure whether a coverage line, a
  carrier, or a workflow works a certain way, ask the maintainer rather than
  guess. Bad domain assumptions leak into prompts and are hard to catch.

## 11. Definition of done for a feature

A feature is not done until:

1. Code is written and passes `make lint typecheck test`.
2. If it touches an LLM agent or synthesis: eval suite runs and does not
   regress.
3. If it touches the API surface: OpenAPI schema is regenerated and
   `packages/shared-types` is updated.
4. If it changes user-facing behavior: README screenshots or Loom link are
   updated.
5. PR description explains the *why*, not just the *what*.
