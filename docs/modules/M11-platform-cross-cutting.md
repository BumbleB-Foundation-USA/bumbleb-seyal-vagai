# M11 — Platform (cross-cutting)

**Phase:** All · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
Shared platform capabilities consumed by every feature module.

## In scope
- Offline sync engine (queue + background sync).
- Tamil/English localization framework.
- Audit logging.
- R2 media handling.
- Performance budgets and the capture-path bundle constraints.

## Out of scope (for this module)
- Feature-specific logic (lives in the consuming module).

## Key entities (see `docs/data-model.md`)
Cross-cutting; no entities of its own beyond audit log.

## Primary users
All modules (esp. M07).

## Dependencies
Foundational; informed by constitution Principles I, II, V.

## Decisions already made
ADR-0001.

## Open questions for `/speckit.clarify`
- Conflict-resolution policy beyond last-write-wins for any entity that needs it.
- Audit-log retention and scope.
