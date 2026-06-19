# Kalvi40 SMS Constitution

*This is the supreme document for the project. Every spec, plan, and task must conform to it. Copy this file to `.specify/memory/constitution.md` after running `specify init`.*

**Version:** 0.1.0 (draft) · **Ratified:** TBD · **Last amended:** initial

---

## Core Principles

### I. Frictionless First (non-negotiable)
Every data-capture interaction is optimized for the fewest taps and zero typing where possible. Defaults, last-used values, and pickers replace free entry. If a field cannot justify itself against a report someone actually reads, it does not ship. The capture surface must be usable one-handed on a low-end Android device by a non-technical user.

### II. Offline-Tolerant by Default
Field users operate in rural schools with unreliable connectivity. Capture must work fully offline, queue locally, and sync when a connection returns. No capture flow may block on the network. Conflict handling is last-write-wins unless a spec states otherwise.

### III. Integrity by Design (system of record)
This system is the single source of truth once live. Every entity has a stable, system-minted ID; every cross-entity link is an enforced foreign key, never a name string. Referential integrity, not convention, prevents orphans. No bidirectional duplication of the same fact.

### IV. Data Minimization & Privacy
Collect the minimum needed. Student-level data is counts and usage only — no student PII. Donor- and volunteer-facing views are aggregate and PII-safe. Personal contact data (staff, TC, HM) is access-controlled by role.

### V. Bilingual & Accessible (Tamil / English)
Tamil and English are first-class throughout. School and type names carry both scripts. UI targets large touch areas, high contrast, and low-literacy-friendly affordances.

### VI. Role-Scoped Access
Access is determined by role and assignment. Users see and edit only what their role and their school/project assignment permit (e.g., a TC sees their schools; an HM sees their school's reports). Auth is OTP-first with long-lived sessions to minimize friction (see ADR-0003).

### VII. Spec-Driven & Reviewable
No implementation without a merged spec. Specs are developed on branches and reviewed via PR. The data model in `docs/data-model.md` is the shared contract all module specs reference.

## Performance & Quality Constraints
- Capture screen interactive in < 1s on a mid-range device; actions feel instant (optimistic UI).
- Small edge-served bundles (Cloudflare Workers); avoid heavy client dependencies on the capture path.
- Accessibility and bilingual coverage are acceptance criteria, not afterthoughts.

## Governance
This constitution supersedes other practices. Amendments are proposed via PR, with rationale and a version bump (semver: MAJOR for principle removal/redefinition, MINOR for new principle, PATCH for clarifications). Every spec's `/speckit.analyze` step must verify conformance.
