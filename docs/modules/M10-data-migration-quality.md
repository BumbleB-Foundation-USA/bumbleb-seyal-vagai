# M10 — Data Migration & Quality

**Phase:** 0 (then ongoing) · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
One-time ETL of the spreadsheet into D1, plus the ongoing validation rules that keep it clean.

## In scope
- Schema-first load per `docs/data-model.md`; mint stable IDs.
- Resolve name-links → FKs; orphan/exception reporting.
- Phone normalization (E.164); multi-number splitting.
- Status resolution; drop placeholder/throwaway columns.
- Ongoing validation rules (FK integrity, required fields, phone format).

## Out of scope (for this module)
- Day-to-day editing UIs (M02–M06).

## Key entities (see `docs/data-model.md`)
Touches all entities. See `docs/migration-plan.md`.

## Primary users
Engineering, admins (clean-up pass).

## Dependencies
Prerequisite for M01–M09. Pairs with M02/M03.

## Decisions already made
ADR-0001 (system of record).

## Open questions for `/speckit.clarify`
- Source-of-truth for ambiguous duplicates.
- Who owns the post-load clean-up pass and on what timeline.
