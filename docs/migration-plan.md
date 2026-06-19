# Migration Plan (Phase 0)

Goal: load the spreadsheet into D1 as a clean, normalized system of record, with an exception report for the BumbleB team to resolve.

## Steps
1. **Schema first.** Create D1 tables per `docs/data-model.md` with FK constraints.
2. **Mint stable IDs** — `SCHOOL-####`, `EMP-####`, `TC-####`, `PROJ-####`, `DONOR-####`; keep `legacy_name`.
3. **Resolve name-links → FKs.** Parse `Project Code` ("Donor - District") into donor_id + district_id; match school/TC by name. Emit an **orphan report** for anything unresolved.
4. **Normalize phones** to E.164 (+91): strip spaces and trailing `.0`; split multi-number Kalvi40 registrations into `school_kalvi40_number`.
5. **Resolve lifecycle.** Convert 366 School "TBD" statuses (team decision); drop ~950 placeholder Project rows.
6. **Collapse denormalization** to single source of truth (TC↔school held once; Implementation Lead = employee FK).
7. **Backfill geo** where map links exist; flag the rest for field collection (ties to M07).
8. **Drop throwaway columns** (`…to be deleted after verification`, `…Need to check`) after sign-off.
9. **Validation report** to the team before and after load.

## Data-quality baseline (from review)
- Employee/TC IDs blank → mint on load.
- All cross-table links are name strings.
- Phones lossy/inconsistent; some schools hold multiple comma-separated numbers.
- School.Status "TBD" on 366/408; Project tab ~950 placeholder rows.
- Geo: coords 19/408 (5%), map links 174/408 (43%).
- Donor contacts mostly blank; only names reliable.
- Employee official email 9/31 (29%); TC email 202/262 (77%), phone 210/262 (80%).
  → The M01 sign-up / profile-completion flow is the mechanism to backfill missing email/phone.
