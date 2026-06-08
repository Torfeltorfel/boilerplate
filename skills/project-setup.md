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
| Content site | TypeScript | Vercel | Next.js or Astro — note ambiguity, ask in Phase 1 |
| API service | TypeScript | any | Node + Hono (lightweight) |
| API service | Python | any | FastAPI |
| API service | Go | any | net/http (stdlib) |
| Mobile app | TypeScript | — | Expo (React Native) |
| Mobile app | other/unsure | — | Flutter |

### Bundler / build tool

| Framework | Infer |
|---|---|
| Next.js | Turbopack (dev), Next.js build (prod) — no separate bundler config needed |
| Astro / SvelteKit / Remix | Vite |
| Hono / Express / FastAPI / Go | No bundler |
| Expo | Metro bundler (built-in) |

### CSS (web only)

| A1 | Infer |
|---|---|
| SaaS / Marketplace / Content site | Tailwind CSS |
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
- Payments, real-time, file storage, notifications, background jobs
- Observability preferences
