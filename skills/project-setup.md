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

**A2 — Deployment target**
Question: "Where will this be deployed?"
Options:
- Vercel (great for Next.js; Hobby plan has no real persistent cron)
- Railway (containers + real cron; no edge functions)
- Fly.io (containers, global regions, flexible)
- AWS / GCP / Azure (full control, more setup required)
- Self-hosted (VPS or bare metal)
- Not decided yet

**A3 — Primary language**
Question: "What's your primary language?"
Options:
- TypeScript / JavaScript
- Python
- Go
- Other

*Skip A3 entirely if A1 + A5 already makes the language unambiguous (e.g. "Next.js SaaS" implies TypeScript). Note the inferred language in the confirmation summary instead.*

**A4 — Scale and team**
Question: "What's the expected scale and team size at launch?"
Options:
- Prototype / side project (solo, no real users yet)
- MVP for real users (small team, 1–5 people)
- Production from day one (larger team, real traffic expected immediately)

**A5 — One-liner**
Ask as open text: "Describe what your app does in one sentence."

This is passed verbatim to the research agent and used by the inference engine to resolve ambiguities not covered by A1–A4.
