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

## The Spec Kit development process

We build **spec-first**: every module becomes a written, reviewable specification before any code is planned or generated. Spec Kit drives this through a sequence of slash commands you run inside your agent. Each command reads the project `constitution.md` and the artifacts before it, then writes the next artifact into `specs/NNN-slug/`.

| Command | Produces | Purpose |
|---|---|---|
| `/speckit.specify` | `spec.md` (+ a new `NNN-slug` branch) | Turn a module brief into a structured spec — *what* and *why*, no tech yet |
| `/speckit.clarify` | updated `spec.md` | Surface and resolve open questions / ambiguities |
| `/speckit.plan` | `plan.md` | Land the *how* — tech choices (Cloudflare/Next/BetterAuth), architecture |
| `/speckit.tasks` | `tasks.md` | Break the plan into dependency-ordered tasks |
| `/speckit.analyze` | report | Cross-artifact consistency check across spec/plan/tasks |
| `/speckit.implement` | code | Execute the tasks (only after the spec + plan are approved) |

The split to keep in mind: **`docs/` is ours to edit freely; `specs/` is managed by Spec Kit's commands.** Modules are specified in **phase order** (see `docs/modules.md`), one at a time — not all at once.

## Iterate on a branch, merge via PR

Requirements are never edited directly on `main`. Each module is specified on its own feature branch so the spec can be reviewed and approved before it lands.

1. **Refine the brief.** Update the module brief in `docs/modules/MNN-*.md` so it captures the agreed scope (this can itself go through a quick PR).
2. **Start the spec on a branch.** Run `/speckit.specify` and feed it the module brief. Spec Kit creates a `NNN-slug` feature branch and writes `specs/NNN-slug/spec.md` onto it. You are now off `main`.
3. **Iterate on the requirements.** Stay on that branch and run `/speckit.clarify`, `/speckit.plan`, `/speckit.tasks`, and `/speckit.analyze` as needed. Re-run any step and hand-edit the artifacts until the spec and plan reflect what the team actually wants — all of this churn stays on the branch, never on `main`.
4. **Open a PR for approval.** Push the `NNN-slug` branch and open a pull request into `main`. Reviewers read `spec.md` (and `plan.md`) and request changes; keep iterating on the branch and pushing until they approve.
5. **Merge.** Once approved, merge into `main`. Update the row in `docs/modules.md` (status: Brief → Spec → Plan → Tasks → Building → Done).

```bash
# After /speckit.specify has put you on the NNN-slug branch:
git push -u origin NNN-slug          # publish the branch
gh pr create --base main --fill      # open the PR for review
# …address feedback on the branch, push again, then merge once approved
```

## Constitution

All specs must conform to `constitution.md` (frictionless-first, offline-tolerant, data-minimal, bilingual, integrity-by-design). Spec Kit references it automatically on every command.
