---
name: project-setup
description: TypeScript project bootstrapper. Asks a handful of questions, writes a project context doc for your review, then researches and validates a full tool stack via subagents. Outputs GitHub issues or a SETUP.md to work through.
---

## What this skill does

Asks 4–5 questions about what you're building. Writes a context doc for you to review and correct. Then dispatches a research subagent to find the best current tools for your specific project, validates the stack, and creates GitHub issues (or a SETUP.md checklist) to work through.

**This skill is opinionated.** It makes choices — framework, runtime, tools. You review before research runs and can push back on anything. TypeScript strict is non-negotiable.

**You'll need:** `git`, `gh` CLI (authenticated), a GitHub account. context7 MCP is optional but improves install steps.

---

## Core Principle

Every tool adds learning curve, ops burden, and context noise. Before recommending anything, ask:

1. Can something already in the stack handle this? → Use it.
2. Is the simplest solution good enough for now? → Ship it, revisit later.
3. Does this justify its ops burden at this project's current scale? → If not, leave it out.

---

**Announce:** "I'm using the project-setup skill. Let me run a few quick checks first."

---

## Pre-flight

Run before asking any questions.

**git**
```bash
git --version
```
If missing: "Please install git (https://git-scm.com) then re-run." Stop.

**gh CLI**
```bash
which gh
```
If missing: "Install the GitHub CLI: `brew install gh` then `gh auth login`. Let me know when done."
Wait for confirmation, then verify with `gh auth status`. If not authenticated, wait again.

**context7**
Ask (AskUserQuestion): "Is context7 MCP installed in your Claude Code setup?"
Options: Yes / No / Not sure — store as `context7_available` (treat "not sure" as no).

**GitHub repo**
Ask (AskUserQuestion): "Create a GitHub repository for this project now?"
Options: Yes / Skip (output will be SETUP.md only)
If yes: ask repo name + Public / Private, run `gh repo create <name> --<public|private> --clone`.
Store as `repo_created`.

---

## Questions

Use AskUserQuestion. Ask one at a time. Five questions maximum.

**Q1 — App type**
"What kind of app is this?"
- SaaS (subscription product, user accounts)
- Marketplace (buyers + sellers)
- Internal tool (your team only)
- Content site (blog, docs, marketing)
- API / backend service (no frontend)
- Mobile app

Store as `app_type`.

**Q2 — Deployment**
"Where will this be deployed?"
- Vercel
- Railway
- Fly.io
- AWS / GCP / Azure
- Self-hosted
- Not decided

Store as `deploy_target`.

**Q3 — One-liner**
"Describe what your app does in one sentence." (open text)
Store as `one_liner`.

**Q4 — Features needed** (multi-select)
"Which of these does your app need?"
- User accounts / auth
- Database
- Payments
- File uploads or image storage
- Real-time (live updates, chat, collaborative editing)
- Email notifications
- Push or SMS notifications
- Background jobs or scheduled tasks
- Error tracking / observability

Store as feature flags: `needs_auth`, `needs_database`, `needs_payments`, `needs_file_storage`, `needs_realtime`, `needs_email`, `needs_push_sms`, `needs_background`, `needs_observability`.

**Q5 — Clarifiers** (only if Q4 left something ambiguous)
- If `needs_background`: "Scheduled/recurring tasks, heavy processing (image resize, PDF gen), or both?"
- If `needs_push_sms`: "Push, SMS, or both?"
- If `needs_database`: "SQL or NoSQL?"
- If `app_type` is "Content site": "Mostly static, or does it need server-side features like a CMS or user comments?"

Skip any clarifier already obvious from Q1–Q3.

---

## Project Context Document

Write `PROJECT_CONTEXT.md` to the project root. This file is passed verbatim to all subagents — it is their only source of truth. Write in plain English.

```markdown
# Project Context

## What we're building
[one_liner]. [1–2 sentences on the core user flow, derived from app_type + one_liner.]

## Deployment & constraints
Target: [deploy_target].
[Write the concrete implication for tool choices:]
- Vercel Hobby: cron limited to once/day, no persistent workers — need an external scheduler for real jobs.
- Vercel Pro: edge functions, cron, and persistent workers available.
- Railway / Fly.io: container-based, real cron, persistent processes.
- AWS / GCP / Azure: full control, CI/CD setup needed separately.
- Self-hosted: responsible for uptime, SSL, and scaling.
- Not decided: no deployment constraints — keep tool choices platform-agnostic.

## Features needed
[One bullet per selected feature. For each, write one sentence on WHY it's needed based on what the app does. Skip features not selected.]

## Cost
Prefer free or cheap. Generous free tiers ideal; low-cost paid tiers acceptable. Flag anything enterprise-priced.

## Guardrails (always on, non-negotiable)
TypeScript strict. Linting. Formatting. Pre-commit hooks. Testing. These are required on every project regardless of other choices.

## Research notes
context7 available: [yes/no]
```

After writing the file, tell the user:
> "I've written `PROJECT_CONTEXT.md`. Review it — edit anything wrong or add context I missed — then say 'looks good' to start researching."

Re-read the file if the user edits it directly. Only proceed on explicit approval.

---

## Research Agent

Dispatch a subagent. The agent determines the full stack — framework, runtime, and every feature tool. Paste the full PROJECT_CONTEXT.md content where indicated.

```
Agent({
  description: "Research current best tools for project stack",
  prompt: `
You are researching the complete tool stack for a new TypeScript project from scratch.

Your job is to find what the community is actually using RIGHT NOW — not what was popular when you were trained. The TypeScript ecosystem moves fast. Treat your training data as a starting hypothesis, not an answer. Web-search every category.

RULES:
- Web-search each category. Verify current version, community adoption, maintenance activity, and free tier status.
- Use context7 if available (noted in project context) to fetch official docs after you've chosen a tool.
- Prefer free or cheap tools. Flag anything expensive or enterprise-priced with free_tier: false.
- Lean principle: recommend the simplest tool that genuinely solves the problem. If something already in the stack covers it, use that. Do not add tools for problems the project doesn't have yet.

PROJECT CONTEXT:
[paste full PROJECT_CONTEXT.md here]

RESEARCH THESE FOR EVERY PROJECT:

"runtime" — What is the current best JavaScript runtime for this framework and deployment target? Consider the deployment constraints in the project context.

"framework" — What is the current best framework for this app type and deployment target? Consider what the community is actively choosing for new projects today.

"typescript_config" — What is the current recommended TypeScript compiler setup for this framework and runtime? How strict should it be? Is there a maintained base config package for this stack? config_snippet required: a working tsconfig.json.

"linting" — What is the current community standard for TypeScript code linting on this stack? Has the tooling changed recently (e.g. config format changes, new tools)? Does the recommended setup also handle code formatting, or is that separate? Record in config_notes: handles_formatting: true/false. config_snippet required: a working lint config file.

"formatting" — Based on the linting recommendation: if linting already handles formatting (handles_formatting: true), what is needed here to prevent conflicts? If linting does not handle formatting, what is the current standard formatting tool and config for this stack? config_snippet required only if a standalone formatter is recommended.

"pre_commit" — What is the current recommended approach for running linting and formatting checks automatically before a commit in TypeScript projects? config_snippet required: a working hook config.

"testing" — What is the current recommended unit/integration test setup for TypeScript + this framework? If the project has a frontend, what is the current recommended end-to-end testing tool? config_snippet required: a working test config file.

RESEARCH THESE IF THE FEATURE IS IN THE PROJECT CONTEXT:

"css" — if frontend: what is the current standard CSS approach for this framework?
"auth" — if auth needed: what is the current best auth solution for this stack?
"database" — if database needed: what is the current best database for these requirements?
"orm" — if database needed: what is the current best query layer for this database + TypeScript?
"payments" — if payments needed: what is the current standard payments integration for this stack?
"file_storage" — if file uploads needed: what is the current best object storage for this deployment?
"realtime" — if real-time needed: what is the current best real-time solution for this framework?
"email" — if email notifications needed: what is the current best transactional email service?
"push_sms" — if push or SMS needed: what is the current best provider?
"background_jobs" — if heavy processing needed: what is the current best job queue for this runtime?
"cron" — if scheduled tasks needed: what is the current best scheduler given the deployment constraints?
"webhooks" — if receiving external events: what is the current best approach for webhook handling and verification?
"error_tracking" — if observability selected: what is the current best error tracking option?
"logging" — if observability selected: what is the current best structured logging library for this runtime?
"analytics" — if observability selected: what is the current best analytics option?
"deployment_tooling" — if the deployment target needs a specific CLI or config: what is the current setup?

OUTPUT — return ONLY a valid JSON array, no prose:
[
  {
    "category": "orm",
    "recommendation": "Drizzle ORM",
    "reason": "Lightweight, TypeScript-first, SQL-like syntax, actively maintained",
    "free_tier": true,
    "maintenance": "active",
    "alternatives": ["Prisma", "Kysely"],
    "install_cmd": "npm install drizzle-orm drizzle-kit",
    "config_notes": "Needs drizzle.config.ts and schema at src/db/schema.ts",
    "config_snippet": ""
  }
]

config_snippet REQUIRED for: typescript_config, linting, formatting, pre_commit, testing.
config_snippet optional for all others — include only if there is a non-obvious config step.
`
})
```

If the agent returns an error or unparseable output: "Research failed: [error]. Retry?" Wait for confirmation.

Parse the JSON. Note any selected feature with no matching result as a gap. Ensure all always-on guardrails are present.

---

## Validator Agent

Second subagent. Reasoning only — no web search.

```
Agent({
  description: "Validate stack for compatibility, gaps, and cost",
  prompt: `
Review this proposed stack. Training knowledge only — no web search.

STACK:
[paste full JSON array from research agent]

PROJECT CONTEXT:
[paste PROJECT_CONTEXT.md]

CHECK:

1. COMPATIBILITY — known bad pairings, runtime conflicts, framework/library version misalignment, deployment violations (e.g. persistent worker required but deploying to Vercel Hobby serverless)

2. MAINTENANCE — flag unmaintained or deprecated tools; flag tools with no activity in 6+ months if a better alternative is in the alternatives list

3. COST — flag enterprise-priced tools; flag tools where free tier was removed recently (e.g. PlanetScale 2024)

4. GAPS — missing implicit dependency (e.g. Stripe with no webhook middleware; tRPC with no HTTP client); missing testing for a project that clearly needs it

5. LEAN — flag any tool more powerful than this project needs (e.g. Kafka for a simple queue → use BullMQ; Elasticsearch for basic search → use Postgres FTS first)

6. OBSERVABILITY — if backend present with no logging: emit {"tool": "logging (missing)", "category": "logging", "verdict": "warn", "note": "No structured logging — consider Pino."}
If no error tracking: emit {"tool": "error_tracking (missing)", "category": "error_tracking", "verdict": "warn", "note": "No error tracking — consider Sentry (free tier available)."}

OUTPUT — return ONLY a valid JSON array:
[
  {"tool": "Drizzle ORM", "category": "orm", "verdict": "ok", "note": ""},
  {"tool": "PlanetScale", "category": "database", "verdict": "fail", "note": "Free tier removed March 2024. Recommend Neon or Turso."},
  {"tool": "ESLint 9", "category": "eslint", "verdict": "warn", "note": "Verify all plugins are flat-config compatible before installing."}
]

verdict: "ok" | "warn" | "fail"
`
})
```

---

## Conflict Resolution

```
FOR EACH tool with verdict "fail":
  1. Auto-swap to the top alternative from research results.
  2. Re-check the swap against validator logic inline (no new subagent).
  3a. Swap is ok → proceed silently.
  3b. Swap has warn → check remaining alternatives first. If still no clean option, ask:
      "I'd use [X] but it has a warning: [note]. Other options: [A], [B]. Which do you prefer?"
  3c. All alternatives also fail → always ask:
      "No clean option for [category]. Trade-offs: [A] — [issue], [B] — [issue]. Which do you want?"

FOR EACH tool with verdict "warn" (not resolved above):
  → Keep the tool. Attach the warning note to its issue or SETUP.md entry.

Silent swaps only when the replacement is a clean "ok".
```

Print before output: "Stack validated: [N] ok, [N] warnings attached, [N] swapped ([list])."

---

## Output

### Dependency order
1. git init (hardcoded, always first)
2. typescript_config
3. linting
4. formatting
5. pre_commit
6. runtime (if any config needed)
7. framework
8. css
9. testing
10. database → orm → auth
11. API tooling
12. Feature tools: payments → email → file_storage → realtime → webhooks → background_jobs → cron
13. Observability: error_tracking → logging → analytics
14. deployment_tooling

### Create labels (run before any issues)
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

### GitHub Issues

One issue per tool, in dependency order. Use this template:

```bash
gh issue create \
  --title "[Setup] <Tool Name>" \
  --label "setup,<category>[,guardrail]" \
  --body "$(cat <<'BODY'
Install and configure <Tool Name>.

**Cost:** <free / cheap: note price / expensive: flag it>
**Validator note:** <note, or "none">

[Only if config_snippet is non-empty:]
<details>
<summary>Suggested starting config (verify against latest docs before using)</summary>

```
[config_snippet]
```
</details>

See PROJECT_CONTEXT.md for stack context and deployment constraints.

> Read PROJECT_CONTEXT.md first. Run a fresh doc search before installing. Use context7 if available.
BODY
)"
```

Add `guardrail` label to: typescript_config, linting, formatting, pre_commit, and testing — these are always present.

**First issue — hardcoded:**
```bash
gh issue create \
  --title "[Setup] Initialize git repository" \
  --label "setup,guardrail" \
  --body "$(cat <<'BODY'
Initialize git, add a .gitignore for this stack, and create the first commit.

See PROJECT_CONTEXT.md for the stack (use it to pick the right .gitignore template).
BODY
)"
```

### SETUP.md (if repo_created = false)

```markdown
# Project Setup
> Work through in order. For each item: read PROJECT_CONTEXT.md first, then run a fresh doc search before installing. Use context7 if available.

## Guardrails
- [ ] git — initialize, .gitignore, first commit
- [ ] TypeScript config — [reason] | [validator note or "none"]
- [ ] Linting — [recommendation + reason] | [validator note or "none"]
- [ ] Formatting — [recommendation + reason] | [validator note or "none"]
- [ ] Pre-commit hooks — [recommendation + reason] | [validator note or "none"]
- [ ] Testing — [recommendation + reason] | [validator note or "none"]

## [Category]
- [ ] **[Tool]** — [reason] | [validator note or "none"]
```

### After output
> "Done! [N] issues created. Want me to start working through them now?"
> *(If context7_available: "I'll use context7 for live docs on each install.")*
