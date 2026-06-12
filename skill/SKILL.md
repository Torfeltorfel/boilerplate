---
name: boilerplate
description: TypeScript project bootstrapper. Asks a handful of questions, writes a project context doc for your review, then researches and validates a full tool stack via subagents. Outputs GitHub issues or a SETUP.md to work through.
---

## What this skill does

Asks a series of questions about what you're building (app type, scale, description, features, architect analysis, deployment). Writes a context doc for you to review and correct. Then dispatches a research subagent to find the best current tools for your specific project, validates the stack, and creates GitHub issues (or a SETUP.md checklist) to work through.

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
Check whether context7 is available by running:
```bash
claude mcp list 2>/dev/null | grep -i context7
```
- If found: set `context7_available = true`. Tell the user: "context7 detected — I'll use it for live docs during research."
- If not found: tell the user: "context7 isn't installed. It gives the research agent access to live, up-to-date docs and significantly improves install steps. Install it with: `npx -y @upstash/context7-mcp` (or follow https://context7.com/docs/getting-started)."
  Then ask (AskUserQuestion): "Install context7 before we continue?"
  Options: Yes, installing now / Skip for now
  If "Yes, installing now": wait for confirmation, then re-run the check above to verify.
  Set `context7_available` based on the final check result.

**Project name** (required — no skip)
Run `basename $(pwd)` to get the current directory name as a suggested default.
Ask (AskUserQuestion): "What should this project be named? (Used as the repo/folder name.)"
Options: [current directory name] / Other (type your own)
Do NOT offer a skip option. Wait until a non-empty name is provided.
If the name contains spaces, convert to kebab-case and confirm: "I'll use `[kebab-name]` — ok?"
Store as `project_name`.

**GitHub repo**
Ask (AskUserQuestion): "Create a GitHub repository for this project now?"
Options: Yes / Skip (output will be SETUP.md only)
If yes: ask (AskUserQuestion) "Public or private?" — Options: Public / Private.
Then run `gh repo create <project_name> --<public|private> --clone`.
Store as `repo_created`.

---

## Questions

Use AskUserQuestion. Ask one at a time.

**Q1 — App type**
"What kind of app is this?"
- SaaS (subscription product, user accounts)
- Marketplace (buyers + sellers)
- Internal tool (your team only)
- Content site (blog, docs, marketing)
- API / backend service (no frontend)
- Mobile app

Store as `app_type`.

**Q2 — Scale**
"What's the ambition level for this project?"
- Side project — ship fast, minimal ops, good enough beats perfect
- Product / startup — balanced: needs to grow but not over-engineered
- Enterprise / team — feature-rich, robust, observability and compliance matter

Normalize before storing: "Side project" → `side-project`, "Product / startup" → `product-startup`, "Enterprise / team" → `enterprise`.
Store as `project_scale`. This drives tool choices throughout:
- `side-project` → prefer the simplest tool that works; skip anything with significant ops overhead; free tier is a hard requirement
- `product-startup` → balance simplicity with room to grow; modest paid tiers acceptable
- `enterprise` → prefer battle-tested, well-supported tools; observability and auditability are first-class concerns; cost is secondary

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

**Architect Analysis** (after Q4, before Q5)

You now know `app_type`, `project_scale`, `one_liner`, and the selected feature flags. Before asking anything, reason from that context like a senior architect would in a design review:

1. **Synthesize what you know.** What kind of system is this really? What are its likely load patterns, user trust model, data sensitivity, and growth trajectory? Write this out internally before deciding what to ask.
2. **Generate hypotheses.** Based on that synthesis, identify the 2–4 concerns most likely to affect tool selection that haven't been covered yet. These should be specific to *this* project — not a generic checklist. Example reasoning: "This is a B2B SaaS for HR teams. HR data is sensitive, companies will need SSO, and somebody will definitely need audit logs for compliance. The real question is whether multi-tenancy is row-level or schema-level."
3. **Ask only about your hypotheses.** Phrase each question as an insight, not a form field: "Since this is a B2B product, enterprise customers will likely need to log in via their own SSO — is that something you need from day one?" This signals reasoning, not box-ticking.

Scale drives the tone and depth:
- **side-project**: flip the framing entirely. Instead of surfacing new needs, challenge the ones already selected. For each Q4 feature with real ops overhead, ask: "You selected [feature] — for a side project, [simpler alternative] often covers this at first. Want to remove it and add it back when you actually need it?" Remove anything the user agrees to defer. Then ask at most 1–2 questions about genuinely non-deferrable concerns.
- **product-startup**: surface 2–3 non-obvious concerns. Lead with the one you're most confident about.
- **enterprise**: surface 3–4 concerns weighted toward data isolation, compliance, access controls, and ops visibility.

When an answer sets a flag, store it:
`needs_sso`, `needs_audit_log`, `needs_search`, `needs_caching`, `needs_api_keys`, `needs_admin`, `needs_i18n`, `needs_bulk_data`, `needs_multitenancy`, `compliance_requirements`.

**Memory aid — categories worth considering** (use to jog your thinking, not as a checklist to run through):
multi-tenancy, SSO/SAML, compliance (GDPR/HIPAA/SOC2/PCI), audit logging, search, caching, external API access, admin dashboard, internationalization, bulk data import/export.

**Q5 — Disambiguation**

Some Q4 answers are underspecified in ways that directly change which tools get researched. Check each condition below. If the flag is set AND the answer isn't already clear from Q1–Q3, ask the follow-up immediately. Skip any where the answer is already obvious.

| Condition | Why it matters | Ask | Store as |
|-----------|---------------|-----|----------|
| `needs_background` is set | A job queue (heavy processing) and a cron scheduler (recurring tasks) are different tools — researching the wrong one wastes the install | "Is this for recurring scheduled tasks, heavy processing (e.g. image resize, PDF gen), or both?" Options: Scheduled tasks / Heavy processing / Both | `background_type`: `scheduled` \| `heavy` \| `both` |
| `needs_push_sms` is set | Push (APNs/FCM) and SMS (Twilio/Vonage) are entirely different services | "Push notifications, SMS, or both?" Options: Push / SMS / Both | `push_type`: `push` \| `sms` \| `both` |
| `needs_database` is set and `one_liner` gives no clear signal | SQL vs NoSQL leads to completely different database and ORM recommendations | "SQL (structured, relational) or NoSQL (flexible, document-based)?" Options: SQL / NoSQL / Not sure | `db_type`: `sql` \| `nosql` \| `unsure` |
| `app_type` is "Content site" | A static site needs a CDN/SSG setup; one with server features needs a CMS or API — different stacks entirely | "Mostly static pages, or does it need server-side features like a CMS or user comments?" Options: Mostly static / Needs server-side features | `content_site_type`: `static` \| `dynamic` |

**Q6 — Deployment**

Before asking, derive a suggestion from what you know:

| Signals | Suggest |
|---------|---------|
| `project_scale` = `side-project`, frontend present | Vercel — free tier, zero config, instant deploys |
| `needs_background` = heavy or both, or `needs_realtime` | Railway or Fly.io — persistent workers, real cron |
| `app_type` = "API / backend service" | Railway or Fly.io — container-based, no serverless cold starts |
| `app_type` = "Mobile app" | Not decided — mobile apps deploy via app stores, not a server platform |
| `project_scale` = `enterprise`, `needs_observability` | AWS / GCP / Azure — full control, compliance tooling available |
| Multiple signals conflict | Omit suggestion, let user decide |

Phrase it as: "Based on what you've told me, I'd suggest **[Platform]** — [one-sentence reason]. Does that work, or do you have something else in mind?"

Then ask (AskUserQuestion): "Where will this be deployed?"
- Vercel
- Railway
- Fly.io
- AWS / GCP / Azure
- Self-hosted
- Not decided

Store as `deploy_target`.

---

## Project Context Document

Write `PROJECT_CONTEXT.md` to the project root. This file is passed verbatim to all subagents — it is their only source of truth. Write in plain English.

**Scale-aware directions — use when writing the Features needed section:**

| Feature | side-project | product-startup | enterprise |
|---------|-------------|-----------------|------------|
| Auth | Social login, generous free tier, no per-MAU pricing cliff | Scalable MAU pricing, MFA, team invite flows | SAML/SSO, compliance-certifiable (SOC2/HIPAA), audit trail |
| Database | Managed, free tier, zero ops | Scales cleanly, good DX, connection pooling available | HA, automated backups, audit logging, compliance-aware |
| Observability | Error capture only — no full observability stack | Error tracking + structured logging | Full stack: errors, logs, distributed tracing, metrics, alerting |
| Background jobs | Simplest managed option, avoid self-hosted infra | Reliable with retries, job visibility | HA, dead-letter queues, predictable scaling |
| Real-time | Managed or simple WebSocket, no self-hosted infra | Managed, reasonable concurrent connection limits | High-availability, low-latency, enterprise connection limits |
| File storage | Managed, free tier | CDN-backed, predictable bandwidth pricing | Geo-redundant, access controls, compliance-certifiable |

Payments, email, and push/SMS don't vary significantly by scale — write WHY needed, no direction required.

```markdown
# Project Context

## What we're building
[one_liner]. [1–2 sentences on the core user flow, derived from app_type + one_liner.]

## Project scale
[project_scale — one of: side-project / product-startup / enterprise]
[One sentence on what this means for tool choices on this project. Examples:]
- side-project: "Choose boring technology by default. The stack has 3 innovation tokens total — spend them only where genuine value justifies the risk."
- product-startup: "Balance simplicity with room to grow. Prefer tools that won't need replacing at 10× scale."
- enterprise: "Observability, auditability, and team access controls are first-class concerns. Cost is secondary."

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
[One bullet per selected feature. For each: (1) one sentence on WHY it's needed for this app; (2) a `→` direction line from the scale-aware table above, using only the column that matches project_scale. Skip features not selected. Skip the direction line for payments, email, and push/SMS.]

## Disambiguation
[Only include flags that were set in Q5. Omit this section entirely if none apply.]
- db_type: [sql | nosql | unsure]
- background_type: [scheduled | heavy | both]
- push_type: [push | sms | both]
- content_site_type: [static | dynamic]

## Architect findings
[List each architect analysis flag that was set to yes, with one sentence on its implication. Skip any set to no. Examples:]
- Multi-tenancy: row-level isolation required — affects database schema and ORM query patterns.
- SSO / SAML: enterprise customers need IdP login — auth solution must support SAML or OIDC federation.
- Compliance: [list requirements] — affects data storage, retention, and logging decisions.
- Search: [level] — affects whether to add a dedicated search tool or use database FTS.
- Caching: yes — a caching layer is needed for repeated reads.
- External API access: yes — needs API key management and rate limiting.
- Admin dashboard: [yes/type] — non-dev users need a management interface.
- Internationalization: yes — app needs i18n library and locale management.
- Bulk data: yes — async import/export architecture required.

## Cost
[Write based on project_scale:]
- side-project: "Free tier required. Flag anything without a free tier."
- product-startup: "Prefer free or cheap. Generous free tiers ideal; low-cost paid tiers acceptable. Flag enterprise-priced tools."
- enterprise: "Cost is secondary to reliability, support, and compliance. Prefer tools with SLAs, enterprise support tiers, and BAA availability where compliance requires it."

## Guardrails (always on, non-negotiable)
TypeScript strict. Linting. Formatting. Pre-commit hooks. Testing. Secrets scanning (pre-commit block on committed credentials). These are required on every project regardless of other choices.

## Research notes
context7 available: [yes/no]
```

After writing the file, tell the user:
> "I've written `PROJECT_CONTEXT.md`. Review it — edit anything wrong or add context I missed — then say 'looks good' to start researching."

Re-read the file if the user edits it directly. Only proceed on explicit approval.

---

## Research Agent

Before dispatching, tell the user:
> "Spinning up a research agent. It will web-search each tool category — framework, runtime, auth, database, and anything else your project needs — to find what the community is actually using right now. This takes a minute or two."

See `research-agent.md` for the full prompt. Paste the full `PROJECT_CONTEXT.md` content where indicated before dispatching.

If the agent returns an error or unparseable output: "Research failed: [error]. Retry?" Wait for confirmation.

Parse the JSON. Note any selected feature with no matching result as a gap. Ensure all always-on guardrails are present.

---

## Validator Agent

Second subagent. Reasoning only — no web search.

See `validator-agent.md` for the full prompt. Paste the full JSON array from the research agent and `PROJECT_CONTEXT.md` where indicated before dispatching.

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

FOR EACH tool with verdict "suggestion":
  → Present to the user in the Stack Review table with a 💡 prefix. Let them decide whether to include it.
  → If included: web-search the tool to get install_cmd and any non-obvious config before creating the issue. Then treat it like any other tool.
  → If declined: drop silently.

Silent swaps only when the replacement is a clean "ok".
```

Print before output: "Stack validated: [N] ok, [N] warnings attached, [N] swapped ([list]), [N] suggestions."

---

## Stack Review

Before creating any issues, present the final stack to the user for approval.

Format it as a markdown table in installation order (the order they must be set up — dependencies first):

```
| # | Category | Tool | Why | Notes |
|---|----------|------|-----|-------|
| 1 | Runtime | Bun | Fast, native TypeScript, matches Vercel edge runtime | |
| 2 | Framework | Next.js 15 | ... | |
| 3 | TypeScript | tsconfig-next | ... | |
| 4 | Linting | ESLint 9 (flat config) | ... | ⚠️ Verify all plugins are flat-config compatible |
| … | … | … | … | |
```

Use ⚠️ for validator warnings, ✅ for clean, 💡 for suggestions. Leave Notes blank if none. Suggestions appear at the bottom of the table, clearly separated.

For tools with `bundled_from` set, show in the Tool column as: **[Tool] (via [Platform])** and set Why to "Bundled — no separate install needed." Do not create a GitHub issue for these; they are covered by the platform's own issue.

If `project_scale` is `side-project`, add a line above the table:
> **Innovation tokens used: [N]/3** — [list each non-boring choice and one-line justification, or "All boring choices — maximum speed ahead."]

This makes the token spend visible so the user can decide if a non-boring pick is worth it before committing.

Then ask (AskUserQuestion): "Ready to create the GitHub issues / SETUP.md from this stack?"
Options: Yes, looks good / Let me change something

If "Let me change something": ask what they'd like to swap or remove (open text), apply the change, reprint the updated table, and ask again. Repeat until approved.

Only proceed to Output after explicit approval.

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
13. Architect tools: multitenancy → sso → audit_log → search → caching → api_keys → admin → i18n → bulk_data → compliance
14. Observability: error_tracking → logging → analytics
15. deployment_tooling

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
gh label create "search" --color "0075ca" --description "" 2>/dev/null || true
gh label create "caching" --color "0075ca" --description "" 2>/dev/null || true
gh label create "admin" --color "cccccc" --description "" 2>/dev/null || true
gh label create "compliance" --color "e4e669" --description "" 2>/dev/null || true
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

Skip creating an issue for any tool where `bundled_from` is set — it's covered by the platform's own issue. **Exception:** if `compliance_requirements` is set, always create an issue for auth, storage, and logging tools even if bundled, with a note to verify the platform is BAA-capable / compliant for the stated requirements.

Add `guardrail` label to: typescript_config, linting, formatting, pre_commit, testing, and secrets_management — these are always present.

**First issue — hardcoded:**
```bash
gh issue create \
  --title "[Setup] Initialize git repository" \
  --label "setup,guardrail" \
  --body "$(cat <<'BODY'
Initialize git, add a .gitignore for this stack, and create the first commit.

- Add `.env.example` with all required env var keys (values blank or documented). Never commit `.env`.
- Ensure `.gitignore` includes `.env`, `.env.*`, and `!.env.example`.
- The pre-commit hook (set up separately) must include secret scanning (`detect-secrets` or equivalent).

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
