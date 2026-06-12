# Boilerplate Skill — Research Agent

## Research Agent

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
- Lean principle: recommend the simplest tool that genuinely solves the problem. Do not add tools for problems the project doesn't have yet.
- Platform bundling principle: before recommending a dedicated tool for any feature, check whether a platform already in the stack covers it. Many platforms bundle multiple capabilities — use them before adding a new dependency. Common examples:
  - Supabase → database + auth + storage + realtime + edge functions
  - Firebase → database + auth + storage + realtime + hosting
  - PlanetScale / Neon / Turso → database only (no auth)
  - Vercel → hosting + edge functions + cron (Pro) + blob storage
  - Cloudflare → hosting + D1 (SQLite) + R2 (storage) + KV + queues
  - Clerk / Auth.js → auth only
  If the chosen database platform also provides auth, storage, or realtime: recommend using it for those features and set "bundled_from": "<platform>" on those result objects instead of recommending a separate tool. Only add a dedicated tool if the bundled capability is clearly insufficient for the project's needs.
- Scale principle: let "Project scale" in the context drive your recommendations:
  - side-project → apply the Innovation Token rule (see below); free tier required; skip anything needing meaningful ops setup
  - product-startup → balance simplicity with growth headroom; modest paid tiers ok; pick tools that won't need replacing at 10× scale.
  - enterprise → prioritize battle-tested, well-supported tools with strong observability, audit logging, and team access controls; cost is not a primary filter
- Innovation Token rule (side-project only): You have 3 innovation tokens for the entire stack. Each "non-boring" tool choice spends one token. Boring means: Postgres, Express, Next.js, Node, React, SQLite, S3, Redis, Stripe — established, widely understood, well-documented. Non-boring means: a newer/trendier alternative that the community is excited about but which adds meaningful learning curve or risk. If you spend all 3 tokens, every remaining category MUST use the boring option, no exceptions. In each recommendation's reason field, note whether it's boring or non-boring (and if non-boring, explain why it justifies a token). Add a field "innovation_token": true/false to each result object for side-project stacks.

PROJECT CONTEXT:
[paste full PROJECT_CONTEXT.md here]

RESEARCH THESE FOR EVERY PROJECT:

"runtime" — What is the current best JavaScript runtime for this framework and deployment target? Consider the deployment constraints in the project context.

"framework" — What is the current best framework for this app type and deployment target? Consider what the community is actively choosing for new projects today.

"typescript_config" — What is the current recommended TypeScript compiler setup for this framework and runtime? How strict should it be? Is there a maintained base config package for this stack? config_snippet required: a working tsconfig.json.

"linting" — What is the current community standard for TypeScript code linting on this stack? Has the tooling changed recently (e.g. config format changes, new tools)? Does the recommended setup also handle code formatting, or is that separate? Record in config_notes: handles_formatting: true/false. config_snippet required: a working lint config file.

"formatting" — Based on the linting recommendation: if linting already handles formatting (handles_formatting: true), what is needed here to prevent conflicts? If linting does not handle formatting, what is the current standard formatting tool and config for this stack? config_snippet required only if a standalone formatter is recommended.

"pre_commit" — What is the current recommended approach for running linting and formatting checks automatically before a commit in TypeScript projects? config_snippet required: a working hook config.

"secrets_management" — What is the current recommended approach for secrets management in this stack? Cover: (1) local development (.env + .env.example convention, never committed), (2) pre-commit secret scanning tool (detect-secrets, git-secrets, or truffleHog), (3) production secrets store — for side-project: environment variables via platform UI is fine; for product-startup: platform env vars + consider Doppler or similar; for enterprise: dedicated vault (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager) with secret rotation. config_snippet required: a working detect-secrets baseline config or equivalent.

"testing" — What is the current recommended unit/integration test setup for TypeScript + this framework? If the project has a frontend, what is the current recommended end-to-end testing tool? config_snippet required: a working test config file.

RESEARCH THESE IF THE FEATURE IS IN THE PROJECT CONTEXT:

"css" — if frontend: what is the current standard CSS approach for this framework?
"auth" — if auth needed: what is the current best auth solution for this stack? For enterprise or if needs_sso is set, also cover RBAC patterns — how the chosen auth solution handles role definitions, permission scopes, and least-privilege token design.
"database" — if database needed: let `db_type` guide the recommendation — `sql` → Postgres/SQLite/etc; `nosql` → MongoDB/DynamoDB/etc; `unsure` → reason from the project description and recommend what fits best.
"orm" — if database needed: what is the current best query layer for this database + TypeScript?
"payments" — if payments needed: what is the current standard payments integration for this stack?
"file_storage" — if file uploads needed: what is the current best object storage for this deployment?
"realtime" — if real-time needed: what is the current best real-time solution for this framework?
"email" — if email notifications needed: what is the current best transactional email service?
"push_sms" — if push or SMS needed: research based on `push_type` — if `push`: focus on APNs/FCM integration; if `sms`: focus on SMS providers (Twilio, Vonage); if `both` or unset: cover both.
"background_jobs" — if `background_type` is `heavy` or `both`: what is the current best job queue for this runtime?
"cron" — if `background_type` is `scheduled` or `both`: what is the current best scheduler given the deployment constraints?
"webhooks" — if receiving external events: what is the current best approach for webhook handling and verification?
"ci_cd" — for product-startup and enterprise: what is the current best CI/CD pipeline setup for this stack and deployment target? Cover: pipeline definition file, branch protection rules, environment secrets injection, preview environment setup if applicable. config_snippet required: a working CI pipeline file (e.g. .github/workflows/ci.yml).
"error_tracking" — if observability selected: what is the current best error tracking option?
"logging" — if observability selected: what is the current best structured logging library for this runtime?
"analytics" — if observability selected: what is the current best analytics option?
"tracing" — if observability selected and project_scale is enterprise or product-startup: what is the current best distributed tracing setup for this stack? Consider OpenTelemetry as the instrumentation standard. What backend (Datadog APM, Honeycomb, Jaeger, Grafana Tempo) fits this deployment and cost profile?
"deployment_tooling" — if the deployment target needs a specific CLI or config: what is the current setup?
"dependency_scanning" — for product-startup and enterprise: what is the current best tool for automated dependency vulnerability scanning and license compliance? (Snyk, Socket.dev, Dependabot, FOSSA, or native npm audit in CI). config_snippet required: a working GitHub Actions step or CI config fragment.
"sso" — if needs_sso: what is the current best SSO/SAML solution for this auth stack?
"audit_log" — if needs_audit_log: what is the current best approach for structured audit logging (append-only, queryable)?
"search" — if needs_search: what is the current best search solution for this database and scale?
"caching" — if needs_caching: what is the current best caching layer for this runtime and deployment?
"api_keys" — if needs_api_keys: what is the current best approach for API key issuance, rate limiting, and usage tracking?
"admin" — if needs_admin: what is the current best admin dashboard option for this stack?
"i18n" — if needs_i18n: what is the current best internationalization library for this framework?
"bulk_data" — if needs_bulk_data: what is the current best approach for async bulk import/export at this scale?
"multitenancy" — if needs_multitenancy: research data isolation end-to-end, not just at the ORM layer. Cover: (1) database-level isolation (Postgres RLS policies, schema-per-tenant, or database-per-tenant — with trade-offs), (2) ORM/query layer — how to enforce tenant scope on every query and prevent cross-tenant leaks, (3) auth layer — how the chosen auth solution enforces tenant membership and scopes tokens to a tenant, (4) background jobs — how tenant context is propagated into async workers so jobs cannot access other tenants' data, (5) caching — how cache keys are namespaced per tenant to prevent cross-tenant cache poisoning.
"compliance" — if compliance_requirements is set: research each stated requirement in depth.
  - GDPR: cookie consent library, data subject request workflow (deletion + export API), data retention policy enforcement, data residency options for chosen database/storage providers.
  - HIPAA: which vendors in the recommended stack offer BAAs (flag any that do not — they cannot be used), encryption at rest and in transit for all PHI, access logging for all PHI access, minimum-necessary access controls.
  - SOC2: audit logging (who accessed what, when), access control evidence (RBAC, MFA enforcement), change management hooks (approved deploy pipeline), vendor security review checklist for each recommended tool.
  - PCI: never store raw card data (use Stripe.js / Stripe Elements to keep card data off your servers), TLS everywhere, dependency vulnerability scanning, access logs for cardholder data environment.
  Flag any recommended tool that does not meet the stated compliance requirement — it must be swapped.

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
    "config_snippet": "",
    "bundled_from": "",
    "innovation_token": false
  }
]

Fields:
- "bundled_from": set to the platform name (e.g. "Supabase") if this capability is already covered by an existing stack tool. Leave empty string otherwise.
- "innovation_token": true if this is a non-boring choice that spends an innovation token (side-project stacks only). false otherwise.

config_snippet REQUIRED for: typescript_config, linting, formatting, pre_commit, testing.
config_snippet optional for all others — include only if there is a non-obvious config step.
`
})
```

---
