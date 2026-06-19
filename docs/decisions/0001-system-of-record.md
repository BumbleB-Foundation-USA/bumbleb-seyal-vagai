# ADR-0001: D1 is the system of record (full cutover)

**Status:** Accepted

## Context
All Kalvi40 operational data lives in an Apple Numbers spreadsheet, linked by name strings, edited manually. We are building a normalized application on Cloudflare D1.

## Decision
Once the system is live and migration is complete, **D1 becomes the single source of truth**. The spreadsheet is exported once as a backup and retired (read-only). No ongoing two-way sync.

## Consequences
- Migration (M10) is a prerequisite for everything else.
- The app must cover day-to-day master-data editing previously done in the sheet (M02–M06).
- Integrity (stable IDs, enforced FKs) is mandatory — see the constitution, Principle III.
