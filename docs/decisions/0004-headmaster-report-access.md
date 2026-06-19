# ADR-0004: Headmasters are users with school-scoped report access

**Status:** Accepted

## Context
HM coverage in the data is excellent (405/408). HMs are currently contacts only.

## Decision
HMs become **login-capable users** who can view reports **scoped to their own school**. For LITE schools (no on-site TC), the HM is also the natural capture path.

## Consequences
- M01 role model includes an HM role with school-scoped access (enabled now, surfaced in reporting later).
- M08/M09 expose school-scoped report views for HMs.
- M07 capture permits HM as a recorder for their school.
