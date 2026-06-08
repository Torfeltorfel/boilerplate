---
name: project-setup
description: Interactive project bootstrapper. Asks high-signal questions, infers stack, researches current tools via a subagent, validates compatibility, then outputs GitHub issues or a SETUP.md checklist.
---

## When to use this skill

Use when starting a brand new project from scratch and you want a researched, compatible tool list with GitHub issues to work through.

Do NOT use when adding a single library to an existing project or when you already have your full stack decided.

**What you'll need:** `git` installed, `gh` CLI installed and authenticated, a GitHub account. context7 MCP is optional but improves doc quality during installation steps.

---

**Announce:** "I'm using the project-setup skill to bootstrap your project. Let me run a few quick checks first."

---

## Pre-flight Checks

Run ALL of these before asking any questions.

### 1. git

```bash
git --version
```

If the command fails:
> "git is not installed. Please install it first (https://git-scm.com), then re-run this skill."

Stop. Do not continue until git is available.

### 2. gh CLI

```bash
which gh
```

If not found:
> "The GitHub CLI (`gh`) is not installed. You'll need it to create the repo and issues.
> Install: `brew install gh` (macOS) or visit https://cli.github.com
> Then run: `gh auth login`
> Let me know when you're done."

Wait for user confirmation before continuing, then re-run `which gh` to verify installation succeeded before proceeding.

```bash
gh auth status
```

If not authenticated:
> "`gh` is installed but not authenticated. Run `gh auth login` then let me know."

Wait for confirmation before continuing.

### 3. context7

Ask the user (AskUserQuestion):
- Question: "Is the context7 MCP server installed in your Claude Code setup? It gives me access to live library docs during installation steps."
- Options: Yes / No / Not sure

Store the answer as `context7_available`. Treat "not sure" as no.

### 4. GitHub repo

Ask using AskUserQuestion:
- Question: "Would you like me to create a GitHub repository for this project now?"
- Options: Yes, create it now / Skip for now (output will be SETUP.md only)

If yes:
- Ask repo name (open text)
- Ask visibility: Public / Private
- Run: `gh repo create <name> --<public|private> --clone`
- If the command fails, show the full error output and ask the user to resolve it before continuing.
- If skipped: set output format to SETUP.md only (no GitHub Issues option later).

Store the result as `repo_created` (true if repo was created, false if skipped). Later phases use this to determine whether GitHub Issues are available as an output format.

---

## Phase 0: Anchor Questions

Ask these questions using AskUserQuestion, one at a time. They are the highest-signal inputs — most of the stack is inferred from them.

**A1 — App type**
Question: "What kind of app is this?"
Options:
- SaaS (subscription product, multiple users with accounts)
- Marketplace (buyers + sellers, transactions between users)
- Internal tool (used only by your own team)
- Content site (blog, docs, marketing, mostly read-only)
- API / backend service (no frontend — consumed by other apps)
- Mobile app (iOS and/or Android)
- Not sure yet

If "Not sure yet": proceed anyway — rely on A5 and later questions to establish the app type. Mark app_type as "unknown" and let the inference engine default to standard web app assumptions.

Store as: **app_type**

**A2 — Deployment target**
Question: "Where will this be deployed?"
Options:
- Vercel (great for Next.js; Hobby plan has no real persistent cron)
- Railway (containers + real cron; no edge functions)
- Fly.io (containers, global regions, flexible)
- AWS / GCP / Azure (full control, more setup required)
- Self-hosted (VPS or bare metal)
- Not decided yet

*(For mobile apps: deployment target refers to the backend/API, if any. App Store / Play Store distribution is handled by the mobile framework — leave this as "Not decided yet" if building mobile-only.)*

Store as: **deploy_target**

**A3 — Primary language**
Question: "What's your primary language?"
Options:
- TypeScript / JavaScript
- Python
- Go
- Other

*Skip A3 if A1 already makes the language unambiguous (e.g. if the user typed "Next.js" or "React" when answering A1, TypeScript/JS is implied). When in doubt, ask. Note the inferred language in the confirmation summary instead.*

Store as: **language**

**A4 — Scale and team**
Question: "What's the expected scale and team size at launch?"
Options:
- Prototype / side project (solo, no real users yet)
- MVP for real users (small team, 1–5 people)
- Production from day one (larger team, real traffic expected immediately)

Store as: **scale**

**A5 — One-liner**
Ask as open text: "Describe what your app does in one sentence."

Store as: **one_liner**

This is passed verbatim to the research agent and used by the inference engine to resolve ambiguities not covered by A1–A4.

---

## Inference Engine

After Phase 0, build a draft project profile by applying these rules before asking any more questions. Inferences are probabilistic — they go into the confirmation summary where the user can correct them. Do NOT ask about inferred items in Phase 1.

### Runtime + framework

| A1 (app type) | Language | A2 (deploy) | Infer |
|---|---|---|---|
| SaaS / Marketplace / Internal tool | TypeScript | Vercel | Next.js (App Router), React, Turbopack |
| SaaS / Marketplace / Internal tool | TypeScript | Railway / Fly.io | Next.js — note as assumption, confirm in summary |
| SaaS / Marketplace / Internal tool | TypeScript | AWS / GCP / Azure / Self-hosted | Next.js or Express — note as assumption, confirm in summary |
| SaaS / Marketplace / Internal tool | Python | any | FastAPI + Jinja2 (server-rendered) or FastAPI (API only — ask in Phase 1) |
| SaaS / Marketplace / Internal tool | Go | any | net/http (stdlib) + templ or htmx — note as assumption, confirm in summary |
| Content site | TypeScript | Vercel | Next.js or Astro — note ambiguity, ask in Phase 1 |
| API service | TypeScript | any | Node + Hono (lightweight) |
| API service | Python | any | FastAPI |
| API service | Go | any | net/http (stdlib) |
| Mobile app | TypeScript | — | Expo (React Native) |
| Mobile app | other/unsure | — | Flutter |
| (no match) | any | any | Cannot infer — ask user in Phase 1 which framework they prefer |

### Bundler / build tool

| Framework | Infer |
|---|---|
| Next.js | Turbopack (dev), Next.js build (prod) — no separate bundler config needed |
| Astro / SvelteKit / Remix | Vite |
| Hono / Express / FastAPI / Go | No bundler |
| Expo | Metro bundler (built-in) |

*If no bundler row matches, note it as unknown and leave bundler choice to the confirmation summary.*

### CSS (web only)

| A1 | Infer |
|---|---|
| SaaS / Marketplace | Tailwind CSS |
| Content site | Tailwind CSS (static-friendly; consider Astro + Tailwind for purely static output) |
| Internal tool | Tailwind CSS (note shadcn/ui as likely addition) |
| API service / Mobile | N/A |

### Testing

| Language + type | Infer |
|---|---|
| TypeScript, web or full-stack | Vitest (unit + integration), Playwright (e2e) |
| TypeScript, API only | Vitest (unit + integration) |
| Python | pytest |
| Go | testing package (stdlib) |

### Complexity tier

| A4 | Apply everywhere |
|---|---|
| Prototype / side project | Lightest tools; skip observability unless user asks |
| MVP for real users | Standard tools; flag observability as recommended |
| Production from day one | Full stack; treat error tracking + logging as required |

### What is NEVER inferred (always ask in Phase 1)

- Auth (product decision — can't derive from app type alone)
- Database (might use external APIs instead)
- Payments
- Real-time features
- File / image storage
- Notifications (email, push, SMS)
- Background jobs and cron tasks
- Observability preferences

---

## Phase 1: Targeted Clarifiers

Ask ONLY questions the inference engine could not resolve. Skip any question whose answer is already known from Phase 0 or inference. A typical run is 5–8 questions. Never exceed 10.

Use AskUserQuestion for each. Ask one at a time.

**Auth** (skip if app type clearly implies no accounts, e.g. pure API service with no user concept):
Question: "Will users have accounts / need to log in?"
Options: Yes / No / Not sure yet

**Database**:
Question: "Does your app need to store data in a database?"
Options: Yes, SQL (PostgreSQL / MySQL / SQLite) / Yes, NoSQL (MongoDB / Redis / etc.) / Both / No / Not sure

**ORM / query layer** (only ask if database answer was yes):
Question: "Do you want an ORM or query builder to talk to the database?"
Options: Yes, suggest one / No, I'll write raw queries / Not sure

**Payments** (ask for SaaS, Marketplace; skip for internal tools and content sites unless A5 implies it):
Question: "Will the app accept payments?"
Options: Yes / No / Not sure

**Notifications**:
Question: "Will the app send notifications to users?"
Options: Email / Push notifications / SMS / None / Not sure
(multiselect)

**File storage**:
Question: "Will the app store user-uploaded files or images?"
Options: Yes / No / Not sure

**Real-time features**:
Question: "Will the app have real-time features? (live updates, chat, collaborative editing)"
Options: Yes / No / Not sure

**External services**:
Question: "Will the app connect to external services?"
Options: Receive events from them (webhooks in) / Send data to them (API calls out) / Both / Neither

Store as: **external_services**. Also derive: **webhooks_inbound** = true if answer includes "Receive events" or "Both".

**Background work**:
Question: "Does your app need to run work outside of a user's request?"
Description shown to user:
- Scheduled / recurring tasks that run even when no user is active (e.g. nightly reports, cleanup jobs, digest emails)
- Heavy processing too slow to block a user (e.g. image resizing, PDF generation, video encoding)
Options: Scheduled / recurring only / Heavy processing only / Both / No

**Observability**:
- If scale = prototype: Ask as one multi-select: "Any observability tools? (skip is fine for a prototype)" Options: Error tracking / Structured logging / Analytics / Monitoring+APM / None for now
- If scale = MVP or production: Ask each separately with a note that error tracking + logging are recommended for production.

**Framework disambiguation** (only if the inference engine flagged an ambiguity, e.g. "Next.js or Astro"):
Ask the specific choice flagged. Example: "For a content site on Vercel — Next.js (more dynamic, API routes) or Astro (faster static output, less JS)?"

**Output format** (ask last, only if repo_created = true):
Question: "How should I deliver the setup checklist?"
Options: GitHub Issues (one per tool, in order) / SETUP.md file
(If repo_created = false, output format is automatically SETUP.md — do not ask.)

Store answers as: **auth_needed**, **database_type**, **orm_needed**, **payments_needed**, **notifications**, **file_storage**, **realtime**, **external_services**, **background_work**, **observability**, **output_format**.

---

## Confirmation Summary

Before dispatching any subagent, present the complete inferred + answered profile in one message. Wait for explicit approval before continuing. The user can correct any row.

Present the summary in this exact format (omit rows that don't apply to this project):

---

Here's what I'm working with before I start researching. Correct anything that looks wrong — or say "looks good" to proceed.

| Category | Value |
|---|---|
| Runtime | [e.g. Node 20 / Python 3.12 / Go 1.22] |
| Framework | [e.g. Next.js 14 App Router / FastAPI / Expo] |
| Language | TypeScript (strict) / Python (typed) / Go |
| Bundler | [e.g. Turbopack / Vite / none] |
| CSS | [e.g. Tailwind v4 / none] |
| Testing | [e.g. Vitest + Playwright / pytest / stdlib] |
| Auth | ✓ needed / ✗ not needed |
| Database | [e.g. PostgreSQL (SQL) / MongoDB / none] |
| ORM | ✓ needed / ✗ not needed |
| Payments | ✓ needed / ✗ not needed |
| Notifications | [e.g. email / push / none] |
| File storage | ✓ needed / ✗ not needed |
| Real-time | ✓ needed / ✗ not needed |
| Webhooks | [e.g. inbound (Stripe events) / outbound / none] |
| Background jobs | [e.g. scheduled + heavy processing / none] |
| Observability | [e.g. error tracking + logging / none] |
| Deployment | [e.g. Vercel / Railway / Fly.io] |
| Output format | GitHub Issues / SETUP.md |
| Guardrails (always on) | TypeScript strict, ESLint, Prettier, Husky + lint-staged |

---

If the user corrects a row: update the profile for that row and acknowledge the change in one line. Do NOT re-show the full summary unless they ask. Do NOT re-run inference.

Only after the user says "looks good", "yes", "proceed", or equivalent — continue to Phase 2.

---

## Phase 2: Research Agent

Dispatch a single subagent using the Agent tool. Substitute the confirmed project profile into `[PROJECT_PROFILE]` before dispatching. Pass `context7_available` as part of the profile. When substituting `[PROJECT_PROFILE]`, append this line to the profile table: `| context7 available | [yes / no] |` so the research agent knows whether to use context7.

```
Agent({
  description: "Research current best tools for project stack",
  prompt: `
You are researching the best current tools for a new project.

INSTRUCTIONS:
- Use web search to verify current versions, free tier status, and maintenance activity.
- If context7 is available (noted in profile), use it to fetch official docs for each library.
- Do NOT rely solely on training data — web-search each recommendation.
- Prefer free or generous free-tier tools. Mark paid-only tools with "free_tier": false.
- Prefer TypeScript-first tools for TypeScript projects.
- Match complexity to stated scale: lightest tools for prototypes, standard for MVPs, full-featured for production.

PROJECT PROFILE:
[PROJECT_PROFILE — paste the full confirmed profile table here]

TASK:
For each applicable category below, find the best current tool. Return a JSON array.

ALWAYS RESEARCH (never skip):
- "typescript_config" — best tsconfig.json for this stack + runtime (or mypy config for Python, go vet for Go)
- "eslint" — flat config vs legacy, plugins for this specific framework (or Ruff for Python)
- "prettier" — current version + relevant plugins, e.g. prettier-plugin-tailwind (or Black/ruff format for Python)
- "precommit_hooks" — husky + lint-staged current setup, or lefthook, or pre-commit (Python)
- "testing_unit" — unit test framework for this stack + runtime

RESEARCH IF APPLICABLE (skip if not in profile):
- "testing_e2e" — e2e test framework; skip this category if the project is API-only (no frontend) or mobile-only
- "frontend_framework" (if not already confirmed)
- "ui_component_library" (if requested)
- "css_tooling" (if Tailwind — research current version and relevant plugins)
- "auth" (if auth = needed)
- "state_management" (if client state complexity warrants it)
- "database" (if database = needed)
- "orm" (if orm = needed)
- "api_tooling" (e.g. tRPC, GraphQL codegen — if applicable)
- "object_storage" (if file storage = needed)
- "realtime" (if real-time = needed)
- "webhook_handling" (if webhooks inbound = needed)
- "background_jobs" (if heavy processing = needed)
- "cron_scheduler" (if scheduled jobs = needed)
- "payments" (if payments = needed)
- "email_transactional" (if email notifications = needed)
- "push_notifications" (if push = needed)
- "sms" (if SMS = needed)
- "error_tracking" (if selected in observability)
- "logging" (if structured logging = needed)
- "analytics" (if analytics = needed)
- "monitoring_apm" (if monitoring = needed)
- "deployment_tooling" (if deployment target needs specific setup, e.g. Vercel CLI, Fly CLI)
- "mobile_framework" (if mobile)

OUTPUT — return ONLY a valid JSON array, no prose:
[
  {
    "category": "orm",
    "recommendation": "Drizzle ORM",
    "reason": "Lightweight, TypeScript-first, SQL-like syntax, active maintenance",
    "free_tier": true,
    "ts_support": "first-class",
    "maintenance": "active",
    "alternatives": ["Prisma", "Kysely"],
    "install_cmd": "npm install drizzle-orm drizzle-kit",
    "config_notes": "Needs drizzle.config.ts and a schema file under src/db/schema.ts"
  }
]
`
})
```

After the agent returns:

If the agent returns an error, empty output, or non-parseable content: tell the user "The research step encountered an error: [error]. Would you like me to retry?" Wait for confirmation before retrying.

1. Parse the JSON array.
2. For any category in the confirmed profile that has no matching result, note the gap.
   For any category where the Phase 1 answer was "Not sure" or "Not sure yet": research it anyway and include it in the results, but add `"conditional": true` to the JSON object so the validator knows to flag it as optional for the user to confirm.
3. Add always-on guardrails as mandatory entries if not already in the results.
4. Assemble the full proposed stack as a flat list for the validator.

---

## Phase 3: Validator Agent

Dispatch a second subagent. No web search — reasoning only. Substitute the assembled stack into `[STACK_JSON]`.

```
Agent({
  description: "Validate proposed stack for compatibility, gaps, and free-tier integrity",
  prompt: `
You are reviewing a proposed tech stack. Use your training knowledge to reason about compatibility, maintenance, and gaps. Do NOT perform web searches.

PROPOSED STACK:
[STACK_JSON — paste the full JSON array returned by the research agent, including all fields: category, recommendation, reason, free_tier, ts_support, maintenance, alternatives, install_cmd, config_notes]

PROJECT CONTEXT:
- Scale: [prototype / MVP / production]
- Deployment: [Vercel / Railway / Fly.io / AWS / self-hosted]
- Language: [TypeScript / Python / Go]

CHECKS TO PERFORM:

1. COMPATIBILITY
   - Known bad pairings (e.g. ESLint 9 flat config with plugins that only support legacy config format)
   - Runtime conflicts (e.g. Bun runtime with tools that have Node.js-only native addons)
   - Framework + library version misalignment (e.g. Next.js 14 requires React 18)
   - Deployment constraint violations (e.g. a cron tool that needs a persistent worker process chosen with Vercel Hobby, which only has serverless functions)

2. MAINTENANCE
   - Flag any tool that appears unmaintained or deprecated
   - Flag tools with no visible activity in the past 6 months IF a better-maintained alternative exists in the alternatives list

3. FREE TIER INTEGRITY
   - Does the full stack hold together under free tier constraints?
   - Flag tools where free tier was removed or severely restricted recently (e.g. PlanetScale removed free tier in 2024)

4. GAPS
   - Any category with an implicit dependency that has no tool? (e.g. tRPC selected but no HTTP client layer mentioned; Stripe webhooks but no webhook verification middleware)
   - Missing testing for a stack that clearly needs it?

5. COMPLEXITY AUDIT
   - Any tool that is significantly over-engineered for the stated scale? (e.g. Apache Kafka chosen for a solo prototype with background jobs)

6. OBSERVABILITY GAPS
   - Scale = production + no error tracking tool in stack → emit: {"tool": "error_tracking (missing)", "category": "observability", "verdict": "warn", "note": "Production app with no error tracking — consider Sentry (free tier available)."}
   - Backend present + no structured logging → emit: {"tool": "logging (missing)", "category": "observability", "verdict": "warn", "note": "Backend with no structured logging — consider Pino (Node) or structlog (Python)."}

OUTPUT — return ONLY a valid JSON array, no prose:
[
  {"tool": "Drizzle ORM", "category": "orm", "verdict": "ok", "note": ""},
  {"tool": "PlanetScale", "category": "database", "verdict": "fail", "note": "Free tier removed March 2024. Recommend Neon (PostgreSQL, generous free tier) or Turso (SQLite, edge-friendly)."},
  {"tool": "ESLint 9", "category": "eslint", "verdict": "warn", "note": "prettier-eslint is not yet compatible with ESLint 9 flat config. Use eslint-config-prettier instead."}
]

verdict values: "ok" | "warn" | "fail"
`
})
```

Before dispatching, transform the research agent's array: rename each object's `"recommendation"` field to `"tool"` so it matches the validator's output schema (which uses `"tool"` as the key).

---

## Conflict Resolution

Apply fixes in this exact order after the validator returns. Do not skip steps.

If the validator returns an error or non-parseable content: treat all tools as "ok" and note "Validator unavailable — proceeding without validation." in the summary line.

```
FOR EACH tool with verdict "fail":

  1. Look up the research agent's `alternatives` list for that category.
  2. Select the first alternative.
  3. Reason inline (no new subagent) about whether that alternative is clean:
     - Apply the same validator checks mentally for the alternative.

  CASE A — alternative is clean ("ok"):
    → Swap silently. Do NOT ask the user.
    → Log internally: "[category]: swapped [old tool] → [new tool]"

  CASE B — alternative has a concern ("warn"):
    → Before presenting to user, check the next alternative(s) in the list the same way.
    → If a later alternative is clean ("ok"), use that one via Case A (silent swap).
    → Only present to user if ALL alternatives have concerns:
      "I'd swap [old tool] but all alternatives have concerns:
       [option A]: [concern]
       [option B]: [concern]
       Which do you prefer?"
    → Wait for user choice. Update the stack with chosen tool.

  CASE C — no clean alternative found (all alternatives also fail or warn):
    → Always ask the user:
      "No clean option found for [category]. Here are the trade-offs:
       [option A]: [issue]
       [option B]: [issue]
       Which do you want to use?"
    → Wait for user choice.

FOR EACH tool with verdict "warn" NOT already resolved above:
  → Keep the tool as-is.
  → Attach the warning note to its issue body / SETUP.md entry.
  → Do NOT silently swap a warned tool without asking.
```

After all conflicts are resolved, print a one-line summary before output:
> "Stack validated: [N] tools confirmed ✓, [N] warnings attached to issues, [N] swapped ([list of swaps])."

---

## Phase 4: Output Generation

### Dependency order

Always output items in this order:

1. git init (hardcoded — always first)
2. TypeScript config (guardrail)
3. ESLint (guardrail)
4. Prettier (guardrail)
5. Husky + lint-staged (guardrail)
6. Runtime / bundler (if any config needed)
7. Frontend framework
8. CSS tooling
9. Testing framework
10. Database
11. ORM / query layer
12. Auth
13. API tooling (tRPC, GraphQL, etc.)
14. Feature tools in this order: payments → email → file storage → real-time → webhooks → background jobs → cron
15. Observability: error tracking → logging → analytics → monitoring
16. Deployment tooling

### Create GitHub Labels

Before creating any issues, ensure the required labels exist. Run these commands:

```bash
gh label create "setup" --color "0075ca" --description "Project setup task" 2>/dev/null || true
gh label create "guardrail" --color "e4e669" --description "Always-on quality guardrail" 2>/dev/null || true
gh label create "database" --color "d93f0b" --description "" 2>/dev/null || true
gh label create "auth" --color "0e8a16" --description "" 2>/dev/null || true
gh label create "testing" --color "bfd4f2" --description "" 2>/dev/null || true
gh label create "payments" --color "5319e7" --description "" 2>/dev/null || true
gh label create "observability" --color "f9d0c4" --description "" 2>/dev/null || true
gh label create "deployment" --color "c2e0c6" --description "" 2>/dev/null || true
```

Create a label for each tool category present in the final stack. The `|| true` ensures the command doesn't fail if the label already exists.

### GitHub Issues

For each tool create one issue using the gh CLI. Use this template:

```bash
gh issue create \
  --title "[Setup] <Tool Name>" \
  --label "setup,<category>[,guardrail]" \
  --body "$(cat <<'BODY'
Install and configure <Tool Name> as the <category> layer.

**Stack context:** <2–3 relevant stack facts, e.g. "PostgreSQL, TypeScript strict, Node 20, Vercel deployment">
**Free tier:** <yes / no — note if paid-only>
**Validator note:** <warning text, or "none">

> When picking this up, run a fresh doc search before installing.
> Use context7 if available.
BODY
)"
```

Add the `guardrail` label to the four always-on tools: TypeScript config, ESLint, Prettier, Husky + lint-staged.

**First issue is always git — hardcoded, never from research:**

```bash
gh issue create \
  --title "[Setup] Initialize git repository" \
  --label "setup,guardrail" \
  --body "$(cat <<'BODY'
Initialize the repository with git, add a .gitignore appropriate for this stack, and create the first commit.

**Stack context:** [one-line stack summary]
**Free tier:** yes
**Validator note:** none

> When picking this up, use context7 or web search to find the correct .gitignore template for this stack.
BODY
)"
```

### SETUP.md

If output format is SETUP.md, write this file to the project root:

```markdown
# Project Setup

> Generated by the project-setup skill. Work through items in order.
> For each item: run a fresh doc search before installing. Use context7 if available.

## Guardrails (always required)

- [ ] **git** — initialize repository, add .gitignore, first commit
- [ ] **TypeScript** — [reason from research agent] | Validator: [note or "none"]
- [ ] **ESLint** — [reason] | Validator: [note or "none"]
- [ ] **Prettier** — [reason] | Validator: [note or "none"]
- [ ] **Husky + lint-staged** — [reason] | Validator: [note or "none"]

## [Category name]

- [ ] **[Tool]** — [reason from research agent] | Validator: [note or "none"]

[Repeat for each category in dependency order]
```

### Final prompt

After all issues are created (or SETUP.md is written):

> "Done! [N] issues created / SETUP.md written.
>
> Want me to start working through them now? I'll tackle them in dependency order, running a fresh doc search for each one[, using context7 for live library docs]."

Include the context7 note only if `context7_available` is true.
