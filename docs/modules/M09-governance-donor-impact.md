# M09 — Governance & Donor/Volunteer Impact

**Phase:** 4 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
Curated, PII-safe rollups and impact views for board, volunteers, and (later) donors.

## In scope
- Aggregate impact metrics and stories; shareable, PII-safe.
- Exports (CSV/PDF) and scheduled digests.
- Internal governance/board reporting.

## Out of scope (for this module)
- Authenticated donor portal (deferred, ADR-0005) — built later on this module.

## Key entities (see `docs/data-model.md`)
Read-only aggregates over operational + capture data.

## Primary users
Board, volunteers, donors (view), leadership.

## Dependencies
Depends on M08 (metrics) and M01 (roles). Constitution Principle IV (privacy) is binding.

## Decisions already made
ADR-0005 (donor portal deferred).

## Open questions for `/speckit.clarify`
- Which metrics are donor-shareable vs internal-only.
- Export formats and digest cadence.
