# SETUP_INSTRUCTIONS.md — Using This Template Pack

This guide explains how to set up these convention docs in your project, including how to make them work with AI coding agents like Claude Code, Cursor, GitHub Copilot, and Codex.

## Initial setup

### Step 1 — Copy the templates into your repo

Choose where the docs live in your repo:

**Option A: Repo root (recommended for solo/small repos)**
```
your-repo/
├── README.md
├── AGENTS.md
├── AZURE_CONVENTIONS.md
├── DATABASE_CONVENTIONS.md
├── API_CONVENTIONS.md
├── CSHARP_CONVENTIONS.md
├── NEXTJS_CONVENTIONS.md
├── GIT_CONVENTIONS.md
├── AUTH_STRATEGY.md
├── SETUP_INSTRUCTIONS.md
├── images/
│   └── git-flow-diagram.png
└── src/
```

**Option B: Under `docs/` (recommended for larger repos)**
```
your-repo/
├── README.md
├── AGENTS.md          ← Keep at root for AI agents to find easily
├── docs/
│   ├── AZURE_CONVENTIONS.md
│   ├── DATABASE_CONVENTIONS.md
│   ├── API_CONVENTIONS.md
│   ├── CSHARP_CONVENTIONS.md
│   ├── NEXTJS_CONVENTIONS.md
│   ├── GIT_CONVENTIONS.md
│   ├── AUTH_STRATEGY.md
│   ├── SETUP_INSTRUCTIONS.md
│   └── images/
│       └── git-flow-diagram.png
└── src/
```

If you go with Option B, update the cross-reference links in each doc accordingly.

> **Important:** Keep `AGENTS.md` at the **repo root**, regardless of where the other docs live. Most AI coding agents look for it there.

### Step 2 — Find-and-replace the project name

The templates use **`OrderHub`** as the example project. Replace with your actual project name:

```bash
# In your repo root, after copying the templates:
# Replace lowercase
find . -name "*.md" -exec sed -i '' 's/orderhub/yourproject/g' {} +

# Replace PascalCase
find . -name "*.md" -exec sed -i '' 's/OrderHub/YourProject/g' {} +

# Replace uppercase
find . -name "*.md" -exec sed -i '' 's/ORDERHUB/YOURPROJECT/g' {} +
```

(For Linux, omit the `''` after `-i`. For Windows, use PowerShell or do it via your IDE's find-and-replace.)

### Step 3 — Adjust stack-specific details

Review each doc and update anything that doesn't match your actual stack:

| If you're using... | Update these docs |
|---|---|
| PostgreSQL instead of SQL Server | `DATABASE_CONVENTIONS.md` (data types, schema syntax, casing conventions differ) |
| AWS instead of Azure | `AZURE_CONVENTIONS.md` (resource types and abbreviations) |
| Vue or Angular instead of Next.js | Replace `NEXTJS_CONVENTIONS.md` with framework-specific equivalent |
| Java/Kotlin/Python backend | Replace `CSHARP_CONVENTIONS.md`; keep `API_CONVENTIONS.md` mostly as-is |
| GitHub Actions instead of Azure DevOps | Update CI/CD references in `GIT_CONVENTIONS.md` |
| Different auth provider (Clerk, Cognito, Firebase Auth) | Adjust `AUTH_STRATEGY.md` |

The **patterns** translate cleanly across stacks. The specific commands, abbreviations, and tools may not.

### Step 4 — Commit and push

```bash
git add .
git commit -m "docs: add engineering convention templates"
git push
```

Done. Your team and your AI coding agents now have a shared reference.

## Using with AI coding agents

### Claude Code

Claude Code automatically reads `AGENTS.md` (or `CLAUDE.md`) at the repo root. No additional configuration needed.

For best results:
1. Keep `AGENTS.md` at the repo root
2. Reference specific docs in your prompts: *"Add a new endpoint following the conventions in API_CONVENTIONS.md and CSHARP_CONVENTIONS.md"*
3. For repo-wide changes, ask Claude Code to read the relevant convention doc first: *"Read CSHARP_CONVENTIONS.md, then refactor the OrderService following the slice structure"*

You can also explicitly mention the `AGENTS.md` file in your initial prompt: *"Follow the conventions in AGENTS.md"* — though Claude Code typically picks this up automatically.

### Cursor

Cursor reads `.cursorrules` files for project-specific instructions. Two options:

**Option 1: Reference the existing docs**

Create `.cursorrules` at the repo root:

```
# Project conventions

Follow the engineering conventions defined in these files:
- AGENTS.md (overview)
- AZURE_CONVENTIONS.md (Azure resources)
- DATABASE_CONVENTIONS.md (database)
- API_CONVENTIONS.md (REST API)
- CSHARP_CONVENTIONS.md (C# backend)
- NEXTJS_CONVENTIONS.md (Next.js frontend)
- GIT_CONVENTIONS.md (git workflow)
- AUTH_STRATEGY.md (auth)

Read the relevant doc before generating code in that area.
Match the established casing and naming conventions exactly.
```

**Option 2: Inline the most critical rules**

Copy the "Critical conventions" section of `AGENTS.md` directly into `.cursorrules` for guaranteed inclusion in every Cursor request. Trade-off: more verbose, but Cursor doesn't have to load other files.

### GitHub Copilot

GitHub Copilot reads `.github/copilot-instructions.md` for repo-specific instructions:

```bash
mkdir -p .github
cp AGENTS.md .github/copilot-instructions.md
```

Or reference the existing files:

```markdown
<!-- .github/copilot-instructions.md -->

# Engineering Conventions

This repository follows the conventions defined in `AGENTS.md` at the repo root,
plus the detailed `*_CONVENTIONS.md` documents.

When generating code:
- Match the casing conventions defined in those docs
- Use the established patterns (Vertical Slice Architecture, plural table names, etc.)
- For non-trivial changes, refer to the specific naming doc for that area
```

### Codex / OpenAI Codex CLI

Codex reads `AGENTS.md` at the repo root by default. Same setup as Claude Code — just keep `AGENTS.md` at the root and you're set.

For more specific guidance, you can pass `--instructions` flags to Codex referencing individual convention docs.

### Aider

Aider reads `CONVENTIONS.md` by default. You can either:
- Rename `AGENTS.md` to `CONVENTIONS.md`
- Or run aider with `--read AGENTS.md` to explicitly include it

### Other AI tools

Most AI coding tools have a way to include project context. Check their docs for the specific filename or configuration:

| Tool | Convention file |
|---|---|
| Claude Code | `AGENTS.md` or `CLAUDE.md` |
| Cursor | `.cursorrules` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Codex | `AGENTS.md` |
| Aider | `CONVENTIONS.md` |
| Continue.dev | `.continue/config.json` (with `systemMessage`) |
| JetBrains AI | Project-level instructions in IDE settings |

If a tool isn't listed, it likely supports either `AGENTS.md` or a similar filename. The point is the same: **make project conventions discoverable**.

## Best practices

### Keep `AGENTS.md` concise

`AGENTS.md` is the entry point — it should be scannable in under 5 minutes. The detailed conventions live in the individual `*_CONVENTIONS.md` files. AI agents (and humans) read the summary first, then dive into specifics as needed.

If `AGENTS.md` exceeds ~10 pages, consider trimming it. The full detail belongs in the topic-specific docs.

### Reference convention docs in PR templates

Add a checkbox to your PR template:

```markdown
## Checklist

- [ ] Code follows project conventions (see AGENTS.md and relevant *_CONVENTIONS.md files)
- [ ] Tests added for new behavior
- [ ] No secrets committed
```

This nudges contributors (and reviewers) to actually consult the docs.

### Keep docs up to date

Conventions evolve. When the team changes a convention, update the relevant doc in the same PR. Stale docs are worse than no docs — they teach the wrong thing to new team members and AI agents.

A rough heuristic: **if a code review comment is "we don't do it that way," check whether that's actually documented somewhere.** If not, document it.

### Use the docs in onboarding

New engineer joining? Their first day is reading `AGENTS.md`, then skimming the `*_CONVENTIONS.md` files relevant to their first project. Better than tribal knowledge.

### Periodic review

Once a quarter, skim the docs and check for:
- Outdated tool versions
- Conventions the team no longer follows in practice
- Missing conventions for things the team now debates regularly

Update accordingly.

## What this approach costs

Honest assessment of tradeoffs:

### What you give up
- **Time to write/maintain.** Initial adoption is cheap (these templates); ongoing maintenance is real but small (5–10 min per quarter).
- **Some team flexibility.** Conventions reduce expressiveness — there's a "right" way that constrains creative choices. This is a feature, not a bug.

### What you gain
- **AI-assisted code that matches your style.** Biggest single win.
- **Faster code review.** Reviewers focus on logic, not bikeshedding naming.
- **Faster onboarding.** New engineers (and AI agents) get up to speed without a 2-week shadow period.
- **Consistent codebase.** Code written 2 years apart looks like the same team wrote it.

### When this approach doesn't help
- **Solo prototypes.** If you're building a quick prototype that may not survive the week, conventions are overhead.
- **Highly variable codebases.** Some research codebases or ML repos legitimately benefit from per-file or per-script conventions rather than project-wide ones.
- **Teams that won't follow them.** Documented conventions only work if reviewers enforce them. Without that, the docs become decoration.

For most professional SaaS / product teams: the value is high, the cost is low. Worth doing.

## FAQ

### Do AI agents really read these files?

Yes. Modern AI coding tools either:
- Auto-detect convention files (Claude Code, Codex, Cursor with `.cursorrules`)
- Are given them explicitly via configuration (Copilot via `.github/copilot-instructions.md`)
- Pull them into context when relevant files are being edited

You can verify by asking the AI: *"What naming conventions does this project use for stored procedures?"* — if it answers correctly without looking, the docs are working.

### What if my team doesn't follow the docs?

Conventions only matter if reviewers enforce them. Three things help:
1. Reference the relevant doc in PR comments: *"Per CSHARP_CONVENTIONS.md, slice handlers should accept CancellationToken — please add"*
2. Add the docs to your PR template checklist
3. For chronic issues, automate via linters (`.editorconfig`, `eslint`, `dotnet format`)

### Is this overkill for a small team?

For a true solo project (one developer, no AI agents): probably yes.
For 2+ developers, or any developer using AI agents: no — the consistency wins are real.

### Can I use this for non-SaaS projects?

The patterns translate, but the specifics are SaaS-flavored. For mobile apps, libraries, ML projects, or desktop apps, you'd adapt heavily. Use the structure as a model; rewrite the contents.

### What's the most important doc to start with?

In rough order of impact for most teams:
1. **`GIT_CONVENTIONS.md`** — branch and commit standards apply to literally every change
2. **`AGENTS.md`** — biggest leverage if you're using AI coding agents
3. **`DATABASE_CONVENTIONS.md`** — schema decisions are hard to undo later
4. **`API_CONVENTIONS.md`** — same reason — API contracts are sticky once published

The other docs add value but can be adopted later.

---

That's it. Drop these files in your repo, find-and-replace the project name, and you have a documented set of engineering conventions that both your team and your AI coding agents can use.
