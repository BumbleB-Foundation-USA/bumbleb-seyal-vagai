# M04 — Project & Donor Management

**Phase:** 2 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
Projects (one per donor × district), finance codes, delivery model, and coverage rollups.

## In scope
- CRUD for projects; finance code; model (LITE/TC); status; start/end periods.
- Coverage rollups (schools per project, TCs per project) computed from FKs, not stored counts.
- Donor management (extends the donor dimension with contacts).
- Employee assignments to projects (Implementation Lead / Project Coordinator).

## Out of scope (for this module)
- Donor-facing portal (deferred, ADR-0005).

## Key entities (see `docs/data-model.md`)
`project`, `employee_project`, `donor`.

## Primary users
Operations, finance, leadership.

## Dependencies
Depends on M02, M03, M05. Feeds M08/M09 reporting.

## Decisions already made
ADR-0001.

## Open questions for `/speckit.clarify`
- Finance-code format/validation rules.
- Project lifecycle states and transitions.
