# M03 — School Master

**Phase:** 1 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
The 408-school registry — the central record everything attaches to.

## In scope
- CRUD + search for schools: bilingual profile, HM contact, grade-band student/teacher counts, geo (coords/map link), status, model.
- Links to district, school_type, donor, project (enforced FKs).
- Kalvi40 registration numbers (multi-number child table).

## Out of scope (for this module)
- Project rollups (M04), capture against schools (M07), reporting (M08).

## Key entities (see `docs/data-model.md`)
`school`, `school_grade_count`, `school_kalvi40_number`. References dimensions + `project`.

## Primary users
Admins, Implementation Leads.

## Dependencies
Depends on M02 (dimensions) and M10 (migrated data). Required by M04, M06, M07, M08.

## Decisions already made
ADR-0001, ADR-0004 (HM linkage).

## Open questions for `/speckit.clarify`
- Status taxonomy beyond Active/Inactive?
- Required vs optional fields for a valid school record.
- Geo capture path for the 95% missing coords.
