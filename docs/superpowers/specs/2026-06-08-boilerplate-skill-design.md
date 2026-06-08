# Boilerplate Setup Skill — Design Spec

**Date:** 2026-06-08  
**Status:** Approved (rev 2)

---

## Overview

A single Claude Code skill (`project-setup`) that guides any user through bootstrapping a new project. It asks a small number of high-signal anchor questions, infers as much as possible, shows a confirmation summary, then researches current best tools via a subagent and validates the full stack for compatibility via a second subagent. Output is a GitHub issue list or SETUP.md.

**Design principles:**
- Ask as few questions as possible — infer aggressively from anchor answers, confirm at the end
- Deployment target is asked early — it fundamentally constrains tool choices
- Pre-flight checks happen before any questions — never waste 20 minutes then hit a blocker
- Always-on guardrails (TypeScript, ESLint, Prettier, Husky + lint-staged) are mandatory slots — but their exact configs are researched, not hardcoded
- Free or generous free-tier tooling is always preferred; paid-only tools are flagged
- Complexity matches stated scale — no Kafka for a solo side project
- GitHub issues are high-level only; each one triggers a fresh doc search at implementation time
- Publicly shareable: no personal config baked in

---

## Architecture

```
Pre-flight      Check git / gh CLI / context7 → create GitHub repo
                      │
Phase 0:        Anchor questions (4–5 questions max)
                      │
                Inference engine → proposed stack draft
                      │
Phase 1:        Targeted clarifiers (only what can't be inferred)
                      │
                Confirmation summary → user corrects if needed
                      │
Phase 2:        Research Agent  (subagent — web search + context7 if available)
                      │
                Main session assembles proposed stack
                (guardrails injected as mandatory slots)
                      │
Phase 3:        Validator Agent  (subagent — reasoning only, no re-research)
                      │
                Conflict resolution  (swap ✗ → re-check → ask user if still ⚠/✗)
                      │
Phase 4:        Output generation
```

---

## Pre-flight

Run before any questions. Never let the user answer a long wizard then hit a blocker at the end.

```
1. git installed?
   → if no: hard stop — "Please install git first, then re-run this skill"

2. gh CLI installed?
   → if no: "gh CLI not found. Install it with `brew install gh` then run `gh auth login`"
   → wait for user to confirm before continuing

3. context7 MCP installed?
   → ask user yes/no
   → stored in session; used by all per-issue research passes later

4. Create GitHub repo now?
   → ask repo name + visibility (public / private)
   → `gh repo create <name> --public/--private`
   → if user skips: output will be SETUP.md only (no issues to link)
```

---

## Phase 0: Anchor Questions

Four to five questions maximum. These are the highest-signal inputs — most of the stack can be inferred from them.

```
A1. What kind of app is this?
    → SaaS / marketplace / internal tool / content site / API service / mobile app / not sure
    (derives: complexity baseline, likely framework family)

A2. Deployment target?
    → Vercel / Railway / Fly.io / AWS / self-hosted / not decided
    (derives: edge vs server runtime, cron availability, observability approach)
    WHY EARLY: Vercel Hobby → no real cron; Railway → no edge functions;
               self-hosted → different observability stack entirely

A3. Primary language?
    → TypeScript/JS / Python / Go / other
    (skip if already obvious from A1 — e.g. "Next.js SaaS" implies TS)

A4. Project scale + team size?
    → prototype (solo) / MVP for real users (small team) / production from day one (larger team)
    (derives: complexity tier for all recommendations)

A5. One-line description of what the app does?
    → free text — used by the inference engine and passed to the research agent
```

---

## Inference Engine

After Phase 0, the skill builds a proposed stack draft by applying inference rules before asking anything else. Examples:

| Anchor combination | Inferred (high confidence) |
|---|---|
| SaaS + Vercel + TS | Next.js, React, Turbopack, Tailwind |
| SaaS + Vercel + TS | Vitest (unit), Playwright (e2e) |
| API service + Railway + TS | Node, Express or Hono, no edge functions |
| API service + Railway + Python | FastAPI, Ruff, mypy, pytest |
| Mobile app | React Native / Expo (if TS), Flutter (if not sure) |
| Prototype + solo | lightweight tools over enterprise ones across all categories |
| Production from day one | error tracking and structured logging flagged as recommended |

Inferences are probabilistic, not absolute — they go into the confirmation summary where the user can correct them.

---

## Phase 1: Targeted Clarifiers

Only ask what the inference engine genuinely cannot determine. Typically 5–8 questions after inference, not 22.

```
For each of the following, only ask if NOT already inferable:

  - Auth needed?
    (skip if "SaaS" or "marketplace" was chosen — assume yes)

  - Database needed? SQL / NoSQL / both?
    (skip if "API service" + one-line description makes it obvious)

  - ORM / query layer?
    (only if database = yes)

  - Will the app accept payments?

  - Will users receive notifications? (email / push / SMS / none)

  - Will the app store files or images?

  - Real-time features? (live updates, chat, collaborative editing)

  - Connect to external services?
    (receive events from them / send data to them / both / neither)

  - Background work outside a user's request?
    Scheduled/recurring (e.g. nightly reports, cleanup jobs)
    — or — heavy processing too slow to block a user (e.g. image resizing, PDF generation)
    Options: scheduled only / heavy processing only / both / no

  - Error tracking? Structured logging? Analytics? Monitoring?
    (collapse into one multi-select if scale = prototype — skip unless explicitly yes)
```

---

## Confirmation Summary

Before dispatching any subagent, show the full inferred + answered stack in one message:

```
Here's what I'm working with — correct anything that looks wrong:

  Runtime:       Node (Bun available as alternative)
  Framework:     Next.js 14 (App Router)
  Language:      TypeScript (strict)
  Bundler:       Turbopack
  CSS:           Tailwind v4
  Testing:       Vitest + Playwright
  Auth:          ✓ (tool TBD — researching)
  Database:      PostgreSQL (tool TBD)
  ORM:           ✓ (tool TBD)
  Payments:      ✓ (tool TBD)
  Background:    scheduled jobs
  Deployment:    Vercel
  Guardrails:    ESLint, Prettier, Husky + lint-staged (always on)

  Anything to change before I start researching?
```

User can say "switch bundler to Vite" or "no payments" and the profile updates. Only after confirmation does the research agent run.

---

## Phase 2: Research Agent

**Input:** Confirmed project profile  
**Constraint:** "Prefer free or generous free-tier tools. Clearly flag paid-only tiers."  
**Approach:** Use context7 if available, otherwise web search.

### Always-on guardrail categories (always researched, never skipped):

| Category | What to find |
|----------|-------------|
| TypeScript config | Strictest recommended tsconfig for this stack and runtime |
| ESLint | Flat vs legacy config decision, which plugins for this framework/runtime |
| Prettier | Current version, relevant plugins (e.g. prettier-plugin-tailwind) |
| Pre-commit hooks | Husky + lint-staged current setup pattern vs alternatives (lefthook) |

### Conditional categories (unlocked by confirmed profile):

| Category | Condition |
|----------|-----------|
| Frontend framework | if not already confirmed |
| UI component library | if requested |
| CSS tooling | if Tailwind: plugins, v3 vs v4 considerations |
| Auth | if needed |
| State management | if client state needed |
| Database | if needed |
| ORM / query layer | if database selected |
| API style tooling | tRPC setup, GraphQL codegen, etc. |
| Object storage | if files/images needed |
| Real-time / WebSockets | if real-time needed |
| Webhook handling | if receiving external events |
| Background jobs / queues | if heavy processing needed |
| Cron / scheduled tasks | if scheduled work needed |
| Payments | if needed |
| Transactional email | if email notifications needed |
| Push notifications | if push needed |
| SMS | if SMS needed |
| Error tracking | if selected |
| Structured logging | if selected |
| Analytics | if selected |
| Monitoring / APM | if selected |
| Testing | always |
| Deployment tooling | if deployment target needs specific setup |
| Mobile framework | if mobile |

### Output format per tool:
```json
{
  "category": "ORM",
  "recommendation": "Drizzle ORM",
  "reason": "Lightweight, TypeScript-first, excellent DX with SQL-like syntax",
  "free_tier": true,
  "ts_support": "first-class",
  "maintenance": "active",
  "alternatives": ["Prisma", "Kysely"],
  "install_cmd": "npm install drizzle-orm drizzle-kit",
  "config_notes": "Requires a drizzle.config.ts and a schema file"
}
```

One entry per category. Alternatives noted but not expanded.

---

## Phase 3: Validator Agent

**Input:** Complete assembled stack as flat list of `{category, tool, version_range}` entries  
**Task:** Independent compatibility review — no re-research, reasoning only.

### Checks:

**Compatibility:**
- Known bad pairings (e.g. ESLint 9 flat config vs plugins that only support legacy config)
- Runtime conflicts (e.g. Bun + tools with Node-only native deps)
- Framework version alignment (e.g. Next.js version vs React version)
- Deployment constraint violations (e.g. cron tool chosen but deployment is Vercel Hobby)

**Maintenance:**
- Flag unmaintained or deprecated tools
- Flag tools with <6 months of activity if a better-maintained alternative exists

**Free tier integrity:**
- Cross-check the full stack holds together under free tier constraints
- Flag tools where free tier was recently removed (e.g. PlanetScale 2024)

**Gaps:**
- Category with no tool but an implicit dependency (e.g. tRPC chosen, no type-safe fetch layer)
- Missing testing for a stack that clearly needs it

**Complexity audit:**
- Flag over-engineered tools for stated scale (e.g. Kafka for a solo side project)

**Observability gaps:**
- Scale = production + no error tracking → ⚠ warn
- Backend present + no structured logging → ⚠ warn

### Output per tool:
```json
{ "tool": "Drizzle ORM", "verdict": "✓", "note": "" }
{ "tool": "PlanetScale", "verdict": "✗", "note": "Free tier removed 2024 — recommend Neon or Turso instead" }
{ "tool": "ESLint 9", "verdict": "⚠", "note": "Use eslint-config-prettier, not prettier-eslint (flat config incompatibility)" }
```

---

## Conflict Resolution

After the validator runs, apply fixes in this order:

```
FOR EACH ✗ tool:
  1. Auto-swap to the research agent's top alternative
  2. Re-run validator logic on the swapped tool (inline, no new subagent)
  3a. If swap is ✓ → proceed silently
  3b. If swap is still ⚠ → present to user:
        "I'd recommend [alternative] but it has a warning: [note].
         Other options: [A], [B]. Which do you prefer?"
  3c. If swap is also ✗ → always ask user:
        "No clean alternative found for [category]. Here are the options:
         [A] — [tradeoff], [B] — [tradeoff]. Which do you want?"

FOR EACH ⚠ tool (that wasn't already resolved above):
  → Attach warning note to the issue/todo — do NOT silently swap
  → User sees it when they work on that issue
```

Silent swaps only happen when the replacement is a clean ✓. Any unresolved uncertainty goes to the user — never hidden.

---

## Phase 4: Output Generation

### First issue is always git — hardcoded, never researched:
```
[Setup] Initialize git repository
  - git init, .gitignore for this stack, initial commit
```

### GitHub Issues (high-level only):

```
Title:   [Setup] Drizzle ORM
Labels:  setup, database
Body:
  Install and configure Drizzle ORM as the query/migration layer.

  Stack context: PostgreSQL, TypeScript, Node runtime.
  Free tier: yes.
  Validator note: none.

  > When picking this up, run a fresh doc search before installing.
  > Use context7 if available.
```

- One issue per tool/category
- No install commands or config snippets in the issue body
- Always-on guardrails labeled `guardrail` to distinguish them
- Issues created in dependency order (runtime → guardrails → framework → features)

### SETUP.md (alternative format):

```markdown
# Project Setup

## Guardrails (always required)
- [ ] git — initialize repository
- [ ] TypeScript (strict)
- [ ] ESLint
- [ ] Prettier
- [ ] Husky + lint-staged

## Database
- [ ] Drizzle ORM

## Auth
- [ ] ...

> ⚠ ESLint 9: use eslint-config-prettier, not prettier-eslint (flat config incompatibility)
```

### Per-issue implementation (when Claude works through them):

```
IF context7 available:
  → resolve library via context7 → fetch latest docs → install + configure
ELSE:
  → web search for latest official docs → install + configure
```

Each issue is treated as an isolated task with a fresh research pass — never relying on wizard-time knowledge for actual installation.

### Final prompt after output:

> "All issues created / SETUP.md written. Want me to start working through them now?"

---

## Skill File Structure

- Single `.md` skill file — no plugin required, shareable as a standalone file
- No personal config baked in (public-ready)
- Skill name: `project-setup`
- Location when published: user's `.claude/skills/` or a public plugin

---

## Out of Scope (v1)

- Generating actual boilerplate code files (only installs and configures tools)
- CI/CD pipeline setup
- Docker / containerization
- Infrastructure as code (Terraform, Pulumi)
- Multi-repo / monorepo orchestration
