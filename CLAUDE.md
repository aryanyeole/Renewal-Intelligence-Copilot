# CLAUDE.md

Working contract for any AI coding agent contributing to this repo. Read it
in full before writing code. If a rule here conflicts with something you'd
otherwise do, the rule wins — ask the maintainer.

---

## 1. What this project is

**Harper Renewal Intelligence Copilot** — a proof-of-work web app that
demonstrates AI-assisted renewal decisioning for a commercial insurance
brokerage. Given an expiring commercial policy scenario (business,
coverage, claims history, carrier appetite context), the app produces a
**renewal brief** styled like an underwriting memo, covering: what
changed, how the current carrier's appetite looks, churn risk, and a
recommendation with talking points.

**Scope note:** this is a portfolio artifact designed to be shown in a
short Loom demo and live URL as part of outreach to Harper Insure, not a
production application. It should feel thoughtful and domain-aware, not
enterprise-shaped. No database, no auth, no background jobs, no
multi-agent orchestration. One Next.js app. One well-designed LLM call.
Four preloaded scenarios. Ship in a day.

## 2. Non-negotiables

1. **One LLM call, structured output only.** Extraction and reasoning
   happen in a single Anthropic call using **tool use** for
   schema-enforced JSON output. Never `json.loads` a raw text completion.
2. **The four brief sections are the schema.** Material changes, carrier
   appetite assessment, churn risk, recommendation + talking points.
   Every scenario produces all four. This is the domain contract, not a
   nice-to-have.
3. **Type safety end to end.** TypeScript strict. The tool schema is
   defined once and Zod-validated at the API boundary.
4. **No secrets in the repo.** `ANTHROPIC_API_KEY` via `.env.local`
   (gitignored); `.env.example` is committed with a placeholder.
5. **The four demo scenarios are hardcoded** in
   `lib/scenarios.ts`. They are the demo. They must feel real — see
   section 6.

## 3. Architecture

Single Next.js 14 App Router project. That's it.

```
harper-renewal-copilot/
├── app/
│   ├── page.tsx          → main UI (scenario picker + brief display)
│   ├── layout.tsx
│   ├── globals.css
│   └── api/
│       └── brief/route.ts → POST endpoint, calls Anthropic, returns brief
├── components/
│   ├── ScenarioPicker.tsx
│   └── RenewalBrief.tsx   → the underwriting-memo document
├── lib/
│   ├── anthropic.ts       → thin wrapper around the SDK
│   ├── prompt.ts          → the system prompt (Python-string style, exported const)
│   ├── tools.ts           → the tool schema for structured output
│   ├── scenarios.ts       → the 4 hardcoded demo scenarios
│   └── schemas.ts         → Zod schemas mirroring the tool schema
├── public/
├── .env.example
├── README.md
└── CLAUDE.md
```

Request flow: user selects a scenario → client POSTs its payload to
`/api/brief` → route handler calls Anthropic with tool use → returns
typed brief → client renders it in the memo layout.

## 4. Tech stack

- **Next.js 14** (App Router), React 18, **TypeScript strict**
- **Tailwind 3.4** for styling. **shadcn/ui** for base components
  (Button, Card, Badge, Skeleton, Alert). Nothing more.
- **Anthropic TypeScript SDK** (`@anthropic-ai/sdk`), `claude-sonnet-4-5`
  as the model (`ANTHROPIC_MODEL` env var, defaulted in code)
- **Zod** for runtime validation at the API boundary
- **Deploy target: Vercel.** Zero config beyond setting `ANTHROPIC_API_KEY`.

No database. No Docker. No auth. No background jobs. No test framework
(one manual pass through each scenario is the acceptance test).

## 5. Code conventions

- Files: `PascalCase.tsx` for components, `kebab-case.ts` or
  `camelCase.ts` for lib.
- No `any`. Use `unknown` and narrow. `// eslint-disable` requires a
  reason comment.
- Server components by default. `"use client"` only for the scenario
  picker and brief display (they need state).
- The system prompt lives as an exported const in `lib/prompt.ts`, not
  inlined. The tool schema lives in `lib/tools.ts`. This keeps the
  domain contract visible and reviewable.
- Commit messages: Conventional Commits (`feat:`, `fix:`, `chore:`,
  `docs:`, `style:`).

## 6. The four demo scenarios

The scenarios in `lib/scenarios.ts` are the product. They must span the
outcome space so the demo shows the copilot exercising judgment, not
just producing memos.

Each scenario has: business (name, industry, state, revenue, employee
count), expiring policy (line, carrier, premium, expiration date, prior
premium for rate trend), recent claims, change events, and relevant
carrier appetite signals. These fields feed the prompt.

The four required scenarios and what each is meant to demonstrate:

1. **Obvious remarket** — hospitality account (bar+restaurant) in
   Florida, currently with Kinsale, where the appetite signal shows
   Kinsale hardening on FL hospitality. Correct call: targeted remarket.
2. **Obvious renew-in-place** — SaaS company with Hiscox E&O, stable
   revenue, no claims, no material change, appetite stable. Correct
   call: renew in place.
3. **Obvious non-renewal risk** — trucking/freight operator with
   Progressive Commercial, two auto claims this term, rising loss ratio,
   Progressive signal shows tightening on high-LR transportation.
   Correct call: full remarket with expectation of rate shock.
4. **Ambiguous** — a childcare/daycare that grew from 25 to 45 staff and
   added a second location, no claims, with Atlantic Casualty. Growth
   is material but clean. Correct call: judgment — could go either way,
   copilot should surface the tension explicitly.

All carrier names above are real Harper partners (per their homepage) —
use them as-is. Do not invent carriers.

## 7. Design direction

This is a workbench for insurance professionals. Do **not** default to
the generic AI-SaaS purple-gradient look. Lean into an
**underwriting-memo aesthetic**:

- Serif display type for headings (Source Serif 4 or similar)
- Parchment/ink palette: cream/off-white background, dark navy ink,
  restrained gold accents
- Monospace for structured data (premiums, dates, carrier names)
- Stamp-style status badges (e.g., "REMARKET" in a rotated bordered box)
- The brief itself should read like a real document — sections with
  rule lines, submission number in the header, effective dates in a
  ledger row

Read `/mnt/skills/public/frontend-design/SKILL.md` before writing UI if
it's available.

## 8. Definition of done

The project is done when:

1. `pnpm dev` runs locally, all four scenarios produce a clean brief.
2. `pnpm build` succeeds with no TypeScript or lint errors.
3. README covers: what this is, why (link to Harper), how to run, tech
   choices, and one screenshot.
4. Deployed to Vercel with a public URL.
5. Loom demo script noted at the bottom of the README.

Everything past that is scope creep for this artifact. Resist it.
