# Kalvi40 School Management System (kalvi40-sms)

System of record + field-data-capture platform for BumbleB Trust / Kalvi40.
Built spec-first with [GitHub Spec Kit](https://github.com/github/spec-kit).

## Repo map

| Path | Owner | What it holds |
|---|---|---|
| `constitution.md` | us | Source for `.specify/memory/constitution.md` (the governing principles) |
| `docs/` | us | Cross-module context: strategy, architecture, data model, migration, decisions, module briefs |
| `docs/modules/` | us | One brief per module (M01–M11) — the collaborative backlog that seeds each spec |
| `docs/decisions/` | us | Architecture Decision Records (ADRs) |
| `specs/` | Spec Kit | Per-feature specs (`NNN-slug/spec.md`, `plan.md`, `tasks.md`) — auto-generated |
| `.specify/` | Spec Kit | Templates, scripts, memory (created by `specify init`) |
| `.github/` | us | PR template, CODEOWNERS, issue templates |

**The split:** `docs/` is ours to edit freely; `specs/` is managed by Spec Kit's commands. Spec numbers (`001`, `002`…) follow build order, not the M# label — see `docs/modules.md` for the map.

## First-time setup

```bash
git init
# 1) Scaffold Spec Kit (pick the agent you collaborate with: claude, copilot, cursor, …)
uvx --from git+https://github.com/github/spec-kit.git specify init --here --integration claude --script sh
# 2) Install our project constitution over the default template
cp constitution.md .specify/memory/constitution.md
# 3) Commit and push
git add -A && git commit -m "chore: scaffold Kalvi40 SMS (docs + constitution + Spec Kit)"
git branch -M main && git remote add origin <your-github-url> && git push -u origin main
```

> Note: `specify init --here` may warn that it's adding to an existing directory — expected. Avoid `--force` later; it overwrites `.specify/memory/constitution.md`.

## Working a module (the per-module loop)

Modules are specified in **phase order** (see `docs/modules.md`), not all at once.

1. Refine the module brief in `docs/modules/MNN-*.md` via a PR; merge when the team agrees on scope.
2. In your agent, run the Spec Kit flow — each `/speckit.specify` creates a `NNN-slug` branch + `specs/NNN-slug/spec.md`:
   ```
   /speckit.specify   (feed it the module brief)
   /speckit.clarify    → resolve open questions
   /speckit.plan       → tech choices land here (Cloudflare/Next/BetterAuth)
   /speckit.tasks      → dependency-ordered tasks
   /speckit.analyze    → cross-artifact consistency check
   ```
3. Open a PR from the `NNN-slug` branch. Review the `spec.md` (and later `plan.md`) before merge.
4. Update the row in `docs/modules.md` (status: Brief → Spec → Plan → Tasks → Building → Done).

## Constitution

All specs must conform to `constitution.md` (frictionless-first, offline-tolerant, data-minimal, bilingual, integrity-by-design). Spec Kit references it automatically on every command.
