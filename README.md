# boilerplate

A collection of Claude Code skills for bootstrapping and maintaining TypeScript projects.

## Skills

### `boilerplate`

Bootstraps a new TypeScript project from scratch. Asks a handful of questions about what you're building, writes a project context doc for you to review, then spawns a research subagent to find the current best tools for your specific stack — not a pre-printed list, but live web research. A validator subagent cross-checks compatibility and cost before anything is created.

**Output:** GitHub issues (one per tool, in dependency order) or a `SETUP.md` checklist to work through.

**Opinionated by design.** It makes choices — framework, runtime, CSS, testing, all of it. You review the proposed context before research runs and can push back on anything. TypeScript strict, linting, formatting, pre-commit hooks, and testing are non-negotiable on every project.

**Future-proof.** No tool names are hardcoded in the skill. The research agent searches the web for what the community is currently recommending, so it stays relevant as the ecosystem evolves.

#### Prerequisites

- `git` installed
- `gh` CLI installed and authenticated (`gh auth login`)
- A GitHub account
- [context7 MCP](https://github.com/upstash/context7) (optional — improves install steps with live library docs)

#### Usage

```
/project-setup
```

#### What it does

1. **Pre-flight** — verifies git, gh CLI, and context7; optionally creates the GitHub repo
2. **Questions** — 4–5 high-signal questions: app type, deployment target, one-liner, features needed
3. **Project context** — writes `PROJECT_CONTEXT.md` for you to review and correct before research starts
4. **Research** — subagent searches the web for the current best tool per category (framework, linting, testing, auth, database, etc.)
5. **Validation** — second subagent checks compatibility, maintenance, cost, and gaps
6. **Output** — GitHub issues or `SETUP.md`, in dependency order, each referencing `PROJECT_CONTEXT.md`

## Installation

Copy the skill file into your Claude Code skills directory:

```bash
cp skills/project-setup.md ~/.claude/skills/
```

Or install via a Claude Code plugin if this repo is published as one.
