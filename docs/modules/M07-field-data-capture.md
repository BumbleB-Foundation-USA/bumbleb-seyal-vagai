# M07 — Field Data Capture

**Phase:** 3 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
The frictionless, offline-tolerant capture app — the original core need, now on clean master data.

## In scope
- Usage capture: student count + modality (smartboard/tablet/both) + lesson/subject.
- Feedback: lesson-content, general, and on-the-spot (single-tap) variants.
- Offline queue + sync; pick-from-assignment (no typing school names).
- Optional media (photo/voice) via R2.

## Out of scope (for this module)
- Aggregation/reporting (M08); analytics views.

## Key entities (see `docs/data-model.md`)
`usage_session`, `feedback`, `lesson`. References `school`, `coordinator`, `user`.

## Primary users
TCs, teachers, HMs (LITE schools).

## Dependencies
Depends on M01 (identity/scope), M03 (schools), M06 (TC assignment), M11 (offline, R2, localization).

## Decisions already made
ADR-0002 (TCs capture), ADR-0004 (HMs capture for LITE schools). Constitution Principles I & II are hard requirements here.

## Open questions for `/speckit.clarify`
- Capture granularity: per class period / per day / per school per day?
- On-the-spot mode: shared/kiosk device vs personal?
- Lesson catalog source (sync from Kalvi40 app vs lightweight reference).
