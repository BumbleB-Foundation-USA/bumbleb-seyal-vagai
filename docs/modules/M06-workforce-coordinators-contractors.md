# M06 — Workforce — Coordinators (TC) & Contractors

**Phase:** 2 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
TC placement at schools, hiring batches/cohorts, and lifecycle; plus contractor records.

## In scope
- CRUD for TCs: school placement, district, project, batch, DOJ/DOL, status.
- Batch/cohort views (Batch 1–4).
- Contractor records by department.
- TC↔school single-source linkage (no bidirectional duplication).

## Out of scope (for this module)
- TC's capture activity (M07) and reporting (M08).

## Key entities (see `docs/data-model.md`)
`coordinator` (TC), `contractor`. References `school`, `district`, `project`, `department`.

## Primary users
Operations, HR.

## Dependencies
Depends on M02, M03. TCs become capture users via M01/M07.

## Decisions already made
ADR-0001, ADR-0002 (TC nomenclature).

## Open questions for `/speckit.clarify`
- One TC ↔ one school always, or can a TC cover multiple schools?
- Contractor lifecycle parity with employees.
