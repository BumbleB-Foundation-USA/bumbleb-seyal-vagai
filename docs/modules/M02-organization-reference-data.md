# M02 — Organization & Reference Data

**Phase:** 0–1 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
Manage the dimension tables that everything else references.

## In scope
- CRUD for districts, school types, departments, roles, donors.
- The `Key`-tab enumerations (donor type, project model, etc.) as managed lookups.
- Bilingual name handling (Tamil/English).

## Out of scope (for this module)
- Schools (M03), projects (M04), workforce (M05/M06).

## Key entities (see `docs/data-model.md`)
`district`, `school_type`, `donor`, `department`, `role`.

## Primary users
Admins / operations.

## Dependencies
Pairs with M10 (these tables are loaded during migration). Referenced by nearly every module.

## Decisions already made
ADR-0001 (system of record).

## Open questions for `/speckit.clarify`
- Which enumerations are user-editable vs fixed?
- Donor contact backfill ownership.
