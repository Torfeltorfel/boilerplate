# Boilerplate Setup Skill — Design Spec

**Date:** 2026-06-08  
**Status:** Approved

---

## Overview

A single Claude Code skill (`project-setup`) that guides any user through bootstrapping a new project. It asks high-level questions about what the app does, infers which tool categories are needed, researches current best tools via a subagent, validates the full stack for compatibility via a second subagent, and generates a GitHub issue list or SETUP.md as output.

**Design principles:**
- Always-on guardrails (TypeScript, ESLint, Prettier, Husky + lint-staged) are mandatory slots — but their exact configs are researched, not hardcoded
- Free or generous free-tier tooling is always preferred; paid-only tools are flagged
- Complexity matches stated scale — no Kafka for a solo side project
- GitHub issues are high-level only; each one triggers a fresh doc search at implementation time
- Publicly shareable: no personal config baked in

---

## Architecture

```
Phase 0+1: Discovery + Wizard  (main session — AskUserQuestion)
           │
           └─► project profile struct
                      │
Phase 2:   Research Agent  (subagent — web search + context7 if available)
           │
           └─► ranked tool recommendations per category
                      │
           Main session assembles proposed stack
           (guardrails injected as mandatory slots)
                      │
Phase 3:   Validator Agent  (subagent — reasoning only, no re-research)
           │
           └─► per-tool verdict: ✓ / ⚠ / ✗ with notes
                      │
           Main session applies fixes (swap ✗, annotate ⚠)
                      │
Phase 4:   Pre-output checks → Output generation
```

---

## Phase 0: App Discovery

Asked before any tool questions. Plain-language questions about behavior — the skill infers which tool categories to unlock from the answers. Users are not expected to know technical terms.

| # | Question | Derives |
|---|----------|---------|
| D1 | What kind of app is this? (SaaS / marketplace / internal tool / content site / API service / mobile app / not sure) | complexity baseline, project type |
| D2 | Will users have accounts? | auth, session management |
| D3 | Will the app accept payments? | payments, webhooks for payment events |
| D4 | Will the app send notifications to users? (email / push / SMS / none / not sure) | transactional email, push service |
| D5 | Will the app store files or images? | object storage (R2, S3, Uploadthing) |
| D6 | Will the app have real-time features? (e.g. live updates, chat, collaborative editing) | websockets, SSE, pub/sub |
| D7 | Will the app connect to external services? (receive events from them / send data to them / both / neither) | webhook handling, API integration layer |
| D8 | Does your app need to run work outside of a user's request? Scheduled/recurring tasks that run even when no user is active (e.g. nightly reports, cleanup jobs) — or — heavy processing too slow to block a user (e.g. image resizing, PDF generation). Options: scheduled only / heavy processing only / both / no | job queue, cron, worker tooling |
| D9 | What's the expected scale at launch? (prototype / MVP for real users / production from day one) | complexity tier for all recommendations |

---

## Phase 1: Tool Wizard

Only questions that cannot be inferred from Phase 0. Conditional branches are gated on project type and Phase 0 answers.

**Always asked:**
1. Project type — web frontend / backend API / full-stack / mobile (multiselect)
2. Primary language — TypeScript/JS / Python / Go / other
3. Runtime / bundler — Node / Bun / Deno; for web also: Vite / Turbopack / Webpack
4. Team size — solo / 2–5 / larger
5. Output format preference — GitHub Issues / SETUP.md

**If web or full-stack:**
6. Frontend framework — React / Vue / Svelte / other
7. SSR / routing — Next.js / Remix / SvelteKit / SPA only
8. UI component library? — yes / no
9. CSS approach — Tailwind / CSS Modules / styled-components / none
10. State management — server state only / client state too / not sure

**If backend or full-stack:**
11. API style — REST / GraphQL / tRPC / other
12. Database type — SQL / NoSQL / both / no

**If database selected:**
13. ORM / query layer — yes (suggest options) / raw queries / not sure

**Always (if backend or full-stack or any server):**
14. Testing scope — unit only / unit + integration / + e2e

**If mobile:**
15. Target platforms — iOS / Android / both
16. Framework preference — React Native / Flutter / Expo / no preference

**Observability (always asked):**
17. Error tracking? — yes / no / not sure
18. Structured logging? — yes / no
19. Analytics? — yes / no / not sure
20. Monitoring / APM? — yes / no / not sure

**Always last:**
21. Deployment target — Vercel / Fly.io / Railway / AWS / self-hosted / not decided
22. Any specific tools already decided? — open text

---

## Phase 2: Research Agent

**Input:** Full project profile from Phase 0 + Phase 1  
**Constraint:** "Prefer free or generous free-tier tools. Clearly flag paid-only tiers."  
**Approach:** Use context7 if available, otherwise web search.

### Always-on guardrail categories (always researched, never skipped):

| Category | What to find |
|----------|-------------|
| TypeScript config | Strictest recommended tsconfig for this stack and runtime |
| ESLint | Flat vs legacy config decision, which plugins for this framework/runtime |
| Prettier | Current version, relevant plugins (e.g. prettier-plugin-tailwind) |
| Pre-commit hooks | Husky + lint-staged current setup pattern vs alternatives (lefthook) |

### Conditional categories (unlocked by Phase 0/1 answers):

| Category | Condition |
|----------|-----------|
| Frontend framework | if not already decided |
| UI component library | if yes in wizard |
| CSS tooling | if Tailwind: plugins, v3 vs v4 considerations |
| Auth | if D2 = yes |
| State management | if client state needed |
| Database | if D2/D3/D7 or explicit yes |
| ORM / query layer | if database selected |
| API style tooling | tRPC setup, GraphQL codegen, etc. |
| Object storage | if D5 = yes |
| Real-time / WebSockets | if D6 = yes |
| Webhook handling | if D7 = receive events |
| Background jobs / queues | if D8 = heavy processing or both |
| Cron / scheduled tasks | if D8 = scheduled or both |
| Payments | if D3 = yes |
| Transactional email | if D4 includes email |
| Push notifications | if D4 includes push |
| SMS | if D4 includes SMS |
| Error tracking | if selected in wizard |
| Structured logging | if selected in wizard |
| Analytics | if selected in wizard |
| Monitoring / APM | if selected in wizard |
| Testing | always for any project with a server; implied for frontend (Vitest + Playwright) |
| Deployment | if not decided, suggest based on stack + free tier |
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

Red (✗) items are swapped before output. Yellow (⚠) items get a warning note attached to their issue/todo.

---

## Phase 4: Output Generation

### Pre-output checks (in order):

1. **git installed?** — if no, hard stop: tell user to install git first
2. **gh CLI installed?** — if no, prompt: `brew install gh` then `gh auth login`
3. **context7 installed?** — ask user yes/no; stored in session for all per-issue research
4. **Create GitHub repo?** — `gh repo create <name>` with visibility preference (public/private)

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
- [ ] TypeScript (strict) ...
- [ ] ESLint ...
- [ ] Prettier ...
- [ ] Husky + lint-staged ...

## Database
- [ ] Drizzle ORM ...

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
