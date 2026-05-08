# Project Conventions Template Pack

A complete set of engineering standards documents that you can drop into any new project to establish conventions on day one — and (importantly) give AI coding agents the context they need to work consistently within your codebase.

## What's in here

| File | Purpose |
|---|---|
| `AGENTS.md` | Quick-reference summary for AI coding agents (Claude Code, Cursor, Codex, etc.) — read first |
| `SETUP_INSTRUCTIONS.md` | How to integrate this pack into your project and AI workflow |
| `AZURE_CONVENTIONS.md` | Azure resource naming conventions |
| `DATABASE_CONVENTIONS.md` | SQL Server / EF Core database object naming |
| `API_CONVENTIONS.md` | REST API conventions (URL structure, HTTP verbs, error formats) |
| `CSHARP_CONVENTIONS.md` | C# / .NET conventions using Vertical Slice Architecture |
| `NEXTJS_CONVENTIONS.md` | Next.js / React frontend conventions |
| `GIT_CONVENTIONS.md` | Branching strategy, commit messages, PR conventions |
| `AUTH_STRATEGY.md` | Authentication (Auth0) and authorization architecture |

## Reference stack

These templates assume the following stack — but the **patterns and conventions translate cleanly to any modern web stack**. Adapt the specifics; keep the structure.

| Layer | Technology |
|---|---|
| Backend | C# / .NET 10, ASP.NET Core, Vertical Slice Architecture |
| Frontend | Next.js 16, React 19, TypeScript, App Router |
| Database | Azure SQL Database, Entity Framework Core 10 |
| Hosting | Azure App Service (backend), Azure Static Web Apps (frontend) |
| Identity | Auth0 (free tier), with app-level authorization |
| Source control | Git on Azure DevOps |
| Observability | Azure Application Insights |

If your stack is different (Postgres instead of SQL Server, AWS instead of Azure, Vue instead of React, etc.), the **principles still apply** — the docs serve as a model for what to write for your own context.

## Project name

The templates use **OrderHub** as the example project — a fictional SaaS order-management product. You'll find:
- Repos: `orderhub-api`, `orderhub-web`, `orderhub-infra`
- Solution: `OrderHub.API.sln`
- Database: `OrderHub`
- Azure resources: `prod-orderhub-rg`, `prod-orderhub-app-api`, etc.

When adopting the templates for your project, **do a project-wide find-and-replace** of:
- `orderhub` → your project name (lowercase)
- `OrderHub` → your project name (PascalCase)
- `ORDERHUB` → your project name (uppercase)

That's it. The rest is reusable as-is.

## Why this exists

Most engineering teams reinvent these conventions on every new project — sometimes inconsistently, sometimes badly, sometimes never (which means juniors guess). This pack captures the conventions that work, with the reasoning baked in so future team members understand *why* — not just *what*.

The bonus: AI coding agents (Claude Code, Cursor, Codex) read markdown files in your repo. Drop these conventions in at the start of a project, and your AI assistants instantly know how to write code that matches your standards. No more re-explaining naming conventions every conversation.

## How to use

### For new projects
1. Copy this entire folder into your repo (commonly under `docs/` or at the repo root)
2. Find-and-replace `OrderHub` / `orderhub` with your project name
3. Adjust any stack-specific details to match your actual technology choices
4. Commit. Done.

### For existing projects
1. Pick the docs most relevant to where you have inconsistencies (often `GIT_CONVENTIONS.md` or `DATABASE_CONVENTIONS.md` are the biggest wins)
2. Adopt one or two at a time — don't try to retrofit everything at once
3. Reference them in PR reviews to help align the team

### For AI coding agents
See `SETUP_INSTRUCTIONS.md` and `AGENTS.md`. Most modern AI coding tools auto-detect markdown files in your repo as context — you'll typically get value with zero configuration.

## A note on opinions

These docs are opinionated. They don't cover every possible approach — they recommend specific ones with reasoning. That's the point: a team with consistent, mediocre conventions ships faster than a team with no conventions and constant debate.

If you disagree with a recommendation, change it. The structure is more valuable than the specific choices. **Pick one, document it, stick to it.**

## License

These templates are released as a starting point for your own engineering documentation. Adapt freely. No attribution required (but appreciated).
