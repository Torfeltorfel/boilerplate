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
