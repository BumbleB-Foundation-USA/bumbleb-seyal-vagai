# M05 — Workforce — Employees

**Phase:** 2 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
Staff directory with departments, roles, org hierarchy, and lifecycle.

## In scope
- CRUD for employees; role, department, manager (self-ref), DOJ/DOL, status, district.
- Project assignments (via `employee_project`).
- Lifecycle (joiners/leavers) and active-staff views.

## Out of scope (for this module)
- Full org-chart/approval logic (waits on lead-hierarchy research, ADR-0005).

## Key entities (see `docs/data-model.md`)
`employee`, `employee_project`. References `role`, `department`, `district`, `project`.

## Primary users
HR, management.

## Dependencies
Depends on M02. Provides assignees to M04; identities to M01.

## Decisions already made
ADR-0001.

## Open questions for `/speckit.clarify`
- Hierarchy above Implementation Leads (research item, ADR-0005).
- Official-email policy for staff onboarding.
