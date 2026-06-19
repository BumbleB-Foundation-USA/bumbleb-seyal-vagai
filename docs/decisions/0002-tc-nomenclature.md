# ADR-0002: "TC" is the canonical term for the paid parent worker

**Status:** Accepted

## Context
Early scoping referred to "paid parent workers." The master data calls the same field role **TC** (Teaching Coordinator).

## Decision
Use **TC** consistently across data model, UI, and specs. "Paid parent worker" is treated as a synonym only in onboarding/help text where helpful.

## Consequences
- The `coordinator` entity (M06) is the TC; 262 records, organized in hiring batches.
- TCs are app users on the capture surface (M07), scoped to their assigned school(s).
