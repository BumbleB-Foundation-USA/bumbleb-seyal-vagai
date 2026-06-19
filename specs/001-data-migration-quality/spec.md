# Feature Specification: Data Migration & Quality (Phase 0)

**Feature Branch**: `001-data-migration-quality`

**Created**: 2026-06-19

**Status**: Draft

**Module**: M10 — Data Migration & Quality · **Phase**: 0 (then ongoing)

**Input**: User description: "Let's create a branch and iterate on the specs required for phase 0. Review all the markdowns, create the specs for these phases and surface all open questions that need to be addressed."

## Overview

Phase 0 turns the Kalvi40 master spreadsheet (`Copy_of_Kalvi40_Master_Database`, 9 tabs, ~1,100 real records) into a clean, normalized **system of record**, per ADR-0001 (full cutover, no two-way sync). This is a prerequisite for every other module: identity, master-data CRUD, workforce, and field capture all build on the data this migration produces.

The work has two halves:

1. **A one-time load (ETL)** — move every tab into the canonical schema in `docs/data-model.md`, minting stable IDs and replacing name-string links with enforced foreign keys.
2. **An ongoing quality regime** — an exception report the BumbleB team resolves, plus validation rules that prevent the same defects (orphans, bad phones, missing required fields) from recurring once the system is live.

This spec stays at the level of **what** clean data must look like and **why**, referencing `docs/data-model.md` as the shared schema contract and `docs/migration-plan.md` for the agreed transformation steps. Concrete tooling and load mechanics are decided in `/speckit-plan`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Load the spreadsheet into a normalized system of record (Priority: P1)

Engineering runs a one-time migration that reads every tab of the master spreadsheet and writes it into the canonical schema. Every record receives a system-minted stable ID (`SCHOOL-####`, `EMP-####`, `TC-####`, `PROJ-####`, `DONOR-####`), the original name is retained as `legacy_name`, and every cross-entity relationship that was a name string in the spreadsheet becomes an enforced foreign key. After this runs, the BumbleB team can query a single, internally consistent dataset instead of a name-linked workbook.

**Why this priority**: Nothing else in the system can be built on name-string master data (per `docs/strategy.md` §4). A clean, FK-enforced dataset with stable IDs is the irreducible Phase 0 deliverable — the MVP. On its own it already lets the team retire the spreadsheet as the editing surface.

**Independent Test**: Run the load against a copy of the spreadsheet; confirm every entity in `docs/data-model.md` is populated, every row has a minted primary key and a `legacy_name`, and a sample of cross-entity links (e.g. school → donor, school → project, TC → school) resolves to a valid foreign key rather than a name.

**Acceptance Scenarios**:

1. **Given** the master spreadsheet with empty `Employee ID` / `TC ID` columns, **When** the migration runs, **Then** every employee and TC record is assigned a unique, stable, system-minted ID and its original name is preserved as `legacy_name`.
2. **Given** a `Project Code` of the form `"Donor - District"` (e.g. `RAMCO CEMENTS - Ariyalur`), **When** the migration runs, **Then** the project is linked to the correct `donor_id` and `district_id` by foreign key, not by the composite string.
3. **Given** a school that references its donor, project, type, and district by name, **When** the migration runs, **Then** each reference is stored as a foreign key and any attempt to load a school whose referenced parent does not exist is rejected (no orphan is silently created).
4. **Given** the 42 real projects sum to exactly 408 schools, **When** the load completes, **Then** the loaded counts reconcile to the same totals (408 schools, 42 projects, 262 TCs, 31 employees, 40 donors, 39 districts).

---

### User Story 2 - Exception & orphan reporting for unresolved records (Priority: P1)

Anything the migration cannot resolve cleanly — a name link with no match, an ambiguous duplicate, a missing required field, a reporting-manager reference that isn't itself an employee — is captured in a human-readable exception report rather than dropped or guessed. The BumbleB team uses this report to clean the source and re-run, so the final dataset is trustworthy.

**Why this priority**: "Clean" is only credible if the gaps are visible. Silent drops or silent guesses would corrupt the system of record at its foundation. The report is what makes the load reviewable (constitution, Principle VII) and is co-critical with the load itself.

**Independent Test**: Seed the source with a deliberately broken record (e.g. a TC pointing at a non-existent school, a school with a blank required field, a duplicate donor name). Run the migration; confirm each defect appears in the exception report with enough detail (tab, row, field, reason) for a non-engineer to locate and fix it, and that the defective record is quarantined rather than loaded as an orphan.

**Acceptance Scenarios**:

1. **Given** a name link that matches no parent record, **When** the migration runs, **Then** the unresolved link is listed in an orphan report identifying the source tab, row, the offending value, and the expected target entity.
2. **Given** a record missing a field the schema requires, **When** the migration runs, **Then** the record is flagged in the exception report and is not loaded with an invalid placeholder.
3. **Given** two records that appear to describe the same real-world entity (duplicate name), **When** the migration runs, **Then** the pair is flagged as an ambiguous duplicate for human resolution rather than auto-merged.
4. **Given** a corrected source after a clean-up pass, **When** the migration is re-run, **Then** previously reported exceptions that have been fixed no longer appear and counts move toward zero unresolved items.

---

### User Story 3 - Phone normalization & Kalvi40 number splitting (Priority: P2)

All phone numbers across schools, employees, TCs, contractors, and donor contacts are normalized to E.164 (`+91…`): floats like `9843585455.0`, embedded spaces like `79044 91923`, and text variants are cleaned to a single canonical form. Where a school holds multiple comma-separated Kalvi40 registration numbers, each is split into its own child record so the school can have many registered app numbers.

**Why this priority**: Kalvi40 app registration is keyed on phone number, and the auth strategy (ADR-0003) leans on phone OTP for field staff. Lossy or merged phone data directly breaks onboarding and capture later. It is high-value but separable from the structural load.

**Independent Test**: Feed the migration a mix of float, spaced, text, and multi-number phone values; confirm each is stored as a valid E.164 string (or flagged if it cannot be normalized), and that a school with comma-separated numbers produces one child record per number.

**Acceptance Scenarios**:

1. **Given** a phone stored as `9843585455.0`, **When** the migration runs, **Then** it is stored as `+919843585455`.
2. **Given** a phone stored as `79044 91923`, **When** the migration runs, **Then** the space is removed and it is stored in canonical E.164 form.
3. **Given** a school whose Kalvi40 registration field holds two comma-separated numbers, **When** the migration runs, **Then** two `school_kalvi40_number` child records are created, each with one normalized number.
4. **Given** a phone value that cannot be normalized to a valid Indian number, **When** the migration runs, **Then** it is flagged in the exception report rather than stored in a malformed state.

---

### User Story 4 - Lifecycle resolution & noise removal (Priority: P3)

The migration resolves unmaintained lifecycle state and strips spreadsheet noise: the 366-of-408 schools with `Status = "TBD"` are converted to a definite Active/Inactive state by an agreed rule, the ~950 fill-down placeholder rows behind the 42 real projects are excluded, bidirectional denormalization is collapsed to a single source of truth (TC↔school held once; Implementation Lead held as an employee FK), and the throwaway verification columns are dropped after the team signs off on the figures.

**Why this priority**: This is cleanup that makes the dataset honest and compact, but the structural load and exception reporting can ship and be reviewed before every lifecycle question is finally settled. It depends on team decisions, so it follows the load.

**Independent Test**: Run the migration over the Project and School tabs; confirm placeholder project rows are absent, no school is left in `TBD`, each denormalized fact appears exactly once, and the throwaway columns are not present in the loaded schema.

**Acceptance Scenarios**:

1. **Given** the Project tab with ~950 fill-down placeholder rows behind 42 real projects, **When** the migration runs, **Then** only the 42 real projects are loaded.
2. **Given** 366 schools with `Status = "TBD"`, **When** the migration runs, **Then** every school has a definite `Active` or `Inactive` status per the agreed resolution rule and none remain `TBD`.
3. **Given** the Implementation Lead represented as a repeated name across School, Project, Employee, and Key, **When** the migration runs, **Then** the lead is represented once as an employee foreign key and not duplicated.
4. **Given** columns labelled "…to be deleted after verification" / "…Need to check", **When** the team has signed off on the verified figures, **Then** those columns are absent from the loaded data.

---

### User Story 5 - Ongoing data-quality validation rules (Priority: P3)

Once the system is live, the same defects must not creep back in. The system enforces ongoing validation rules — referential integrity (no name-string links, no orphans), required-field presence, and phone-format (E.164) validity — so that any later edit or import that would reintroduce a known defect is rejected or flagged.

**Why this priority**: This converts the one-time clean-up into a durable guarantee (the "then ongoing" half of M10). It is essential for keeping the system of record clean but logically follows the initial load and can be hardened iteratively.

**Independent Test**: After load, attempt to create a record that violates each rule (a dangling foreign key, a missing required field, a malformed phone); confirm each attempt is rejected or flagged with a clear reason.

**Acceptance Scenarios**:

1. **Given** the live system of record, **When** any attempt is made to create a cross-entity link to a non-existent parent, **Then** the operation is rejected by referential-integrity enforcement.
2. **Given** the live system, **When** a record is saved without a schema-required field, **Then** the save is rejected with a message identifying the missing field.
3. **Given** the live system, **When** a phone number is entered in a non-E.164 form, **Then** it is normalized or rejected, never persisted malformed.

---

### Edge Cases

- **Reporting-manager points to a non-employee** — some manager references (e.g. *Karthick Subramaniam*) are not present as employee rows. The self-referencing manager FK is left null and the dangling reference is reported (org-hierarchy resolution is deferred per ADR-0005).
- **Multiple plausible matches for one name** — a name resolves to more than one candidate parent; treated as an ambiguous duplicate (US2), never auto-resolved.
- **Whitespace / case / script variants of the same name** — "RAMCO CEMENTS" vs "Ramco Cements " should resolve to the same donor; near-but-not-exact matches are surfaced for confirmation.
- **Bilingual name present in only one script** — a school or type with a Tamil name but no English (or vice versa) loads with the available script and flags the missing pair rather than blocking.
- **Geo present as a map link but no coordinates** — coordinates are backfilled where a map link exists; the remaining ~95% without coordinates are flagged for later field collection (ties to M07), not blocked.
- **Re-running the migration** — a second run must not create duplicate IDs or double-load child records; the load is repeatable to a consistent end state.
- **Zero-real-data tab anomalies** — a tab whose only rows are placeholders/blank yields no loaded records and a note in the report, not an error.

## Requirements *(mandatory)*

### Functional Requirements

**Load & identity**

- **FR-001**: The migration MUST load every entity defined in `docs/data-model.md` from the corresponding spreadsheet tab(s): districts, school types, donors, departments, roles, projects, schools, employees, coordinators (TC), contractors, and the `Key`-tab enumerations.
- **FR-002**: The migration MUST mint a unique, stable, system-generated primary key for every record using the agreed prefixes (`SCHOOL-####`, `EMP-####`, `TC-####`, `PROJ-####`, `DONOR-####`, and equivalent for other entities) and MUST retain each record's original spreadsheet name as `legacy_name`.
- **FR-003**: The migration MUST replace every cross-entity name-string link with an enforced foreign key; no relationship may be stored as a name string in the loaded data.
- **FR-004**: The migration MUST parse the composite `Project Code` (`"Donor - District"`) into the correct `donor_id` and `district_id` foreign keys.
- **FR-005**: The migration MUST reconcile loaded record counts to the known totals (408 schools, 42 projects, 262 TCs, 31 employees, 40 donors, 39 districts, 57 school types, 15 contractors) and report any discrepancy.
- **FR-006**: The migration MUST be idempotent/repeatable — re-running it MUST converge to the same end state without duplicating IDs or child records.

**Exception & orphan reporting**

- **FR-007**: The migration MUST emit a human-readable exception report listing every unresolved name link (orphan), missing required field, ambiguous duplicate, and dangling hierarchy reference, each identified by source tab, row, field, value, and reason.
- **FR-008**: The migration MUST quarantine (not load as an orphan, and not silently drop) any record it cannot resolve, so the report is the authoritative list of what still needs human attention.
- **FR-009**: The migration MUST NOT auto-merge records flagged as ambiguous duplicates; resolution is a human decision recorded in the source and applied on re-run.
- **FR-010**: The system MUST produce a validation report for the BumbleB team **before and after** the load so the team can review data quality at both points (per `docs/migration-plan.md` step 9).

**Phone & contact normalization**

- **FR-011**: The migration MUST normalize all phone numbers to E.164 (`+91…`), removing float artifacts (`.0`), embedded spaces, and text formatting.
- **FR-012**: The migration MUST split multi-number Kalvi40 registrations (comma-separated) into one `school_kalvi40_number` child record per number.
- **FR-013**: The migration MUST flag any phone value that cannot be normalized to a valid number in the exception report rather than store it malformed.

**Lifecycle & noise resolution**

- **FR-014**: The migration MUST exclude the ~950 fill-down placeholder rows on the Project tab and load only the 42 real projects.
- **FR-015**: The migration MUST resolve every `Status = "TBD"` school (366 of 408) to a definite `Active` or `Inactive` value per the agreed resolution rule, leaving none as `TBD`.
- **FR-016**: The migration MUST collapse bidirectional denormalization so each fact (TC↔school linkage, Implementation Lead assignment) is stored exactly once, as a foreign key.
- **FR-017**: The migration MUST drop the throwaway verification columns ("…to be deleted after verification", "…Need to check") only after the team signs off on the verified figures.
- **FR-018**: The migration MUST backfill geo-coordinates where a map link is available and flag schools lacking coordinates for later field collection, without blocking their load.

**Bilingual handling**

- **FR-019**: The migration MUST preserve bilingual (`*_ta` / `*_en`) name pairs for districts, school types, and schools, and MUST flag records where only one script is present rather than fabricating the missing one.

**Ongoing validation**

- **FR-020**: The system MUST enforce referential integrity on an ongoing basis so no future edit or import can create an orphan or a name-string link.
- **FR-021**: The system MUST enforce required-field presence on an ongoing basis and reject saves that omit a required field, with a message identifying it.
- **FR-022**: The system MUST enforce E.164 phone validity on an ongoing basis, normalizing or rejecting non-conformant input rather than persisting it.
- **FR-023**: The migration and validation process MUST NOT introduce any student-level personal data; student figures are counts only (constitution, Principle IV).

### Key Entities *(include if feature involves data)*

The migration touches **all** entities in `docs/data-model.md`. The ones most affected by transformation rules:

- **district / school_type / donor / department / role** — dimension tables loaded first so operational records have valid FK targets; carry bilingual names where applicable.
- **project** — 42 real records; composite `"Donor - District"` code parsed into FKs; ~950 placeholder rows excluded.
- **school** — the 408-row core; resolves donor/project/type/district name links to FKs, normalizes HM phone, holds grade-band counts and geo, resolves `TBD` status, and owns `school_kalvi40_number` (multiple app numbers) as a child table.
- **employee** — 31 records; minted IDs, self-referencing manager FK (left null where the manager isn't an employee), project assignments via `employee_project`.
- **coordinator (TC)** — 262 records; minted IDs, single-source TC↔school linkage, hiring batch retained.
- **contractor** — 15 records; minted IDs, department FK.
- **Exception report** *(migration artifact)* — the authoritative list of unresolved orphans, missing fields, ambiguous duplicates, and dangling references for the team to clean.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of loaded records have a system-minted stable primary key and a retained `legacy_name`.
- **SC-002**: 100% of cross-entity relationships in the loaded dataset are foreign keys; zero relationships remain stored as name strings.
- **SC-003**: Loaded counts reconcile exactly to the known totals (408 schools, 42 projects, 262 TCs, 31 employees, 40 donors, 39 districts) with any variance itemized in the report.
- **SC-004**: Zero schools remain in `TBD` status after the load; zero placeholder project rows are present.
- **SC-005**: 100% of stored phone numbers are valid E.164, and every school's Kalvi40 registration numbers are individually addressable child records.
- **SC-006**: Every record that cannot be cleanly migrated appears in the exception report with enough detail (tab, row, field, reason) for a non-engineer to act on it — zero silent drops.
- **SC-007**: A non-engineer on the BumbleB team can read the exception report and resolve a flagged item in the source without engineering help.
- **SC-008**: After cutover, attempts to create an orphan, omit a required field, or store a malformed phone are rejected or flagged — none persist in the system of record.
- **SC-009**: The migration can be re-run end-to-end and converge to the same result (no duplicate IDs, no double-loaded children).

## Assumptions

- **Source format**: The migration consumes a one-time export of `Copy_of_Kalvi40_Master_Database` (9 tabs). The live spreadsheet is frozen at the agreed export and retired afterward (ADR-0001); there is no two-way sync.
- **Schema contract**: `docs/data-model.md` is the authoritative target schema; entity shapes, FK directions, and bilingual/phone conventions follow it. Schema changes are PR-reviewed there, not invented here.
- **ID scheme**: Stable IDs use the `ENTITY-####` prefix convention from `docs/migration-plan.md`; zero-padding width is an implementation detail for `/speckit-plan`.
- **Contact backfill**: Missing employee/TC email and phone, and sparse donor contacts, are **not** fully resolved by this migration. They are flagged for backfill through the M01 sign-up / profile-completion flow (per `docs/migration-plan.md`); this spec only normalizes and reports what exists.
- **Org hierarchy**: The reporting-manager structure above Implementation Leads is deferred (ADR-0005). The self-referencing manager FK is populated where the manager is a known employee and left null (and reported) otherwise.
- **Geo**: Only ~5% of schools have coordinates and ~43% have map links; full geo coverage is a later field-collection effort (M07), not a Phase 0 exit condition.
- **Privacy**: No student PII is migrated; student data is counts only (constitution, Principle IV).
- **Validation surface**: Ongoing validation rules (FR-020–022) are defined here as requirements; whether they are enforced at the database, application, or both layers is an implementation choice for `/speckit-plan`.

## Open Questions

These need team answers before or during `/speckit-clarify`. The three most decision-critical are also embedded as `[NEEDS CLARIFICATION]` in context:

1. **TBD status resolution rule** — [NEEDS CLARIFICATION: How should the 366 schools with `Status = "TBD"` be converted to Active/Inactive? Options: (a) a derivable rule, e.g. "Active if it belongs to a real project / has an active TC", (b) a team-supplied list/decision per school, (c) default all TBD → Active pending review.] Impacts FR-015 and SC-004.
2. **Source-of-truth for ambiguous duplicates** — [NEEDS CLARIFICATION: When two records describe the same real-world entity or a denormalized fact conflicts across tabs, which source wins? e.g. is School's copy or TC's copy authoritative for the TC↔school link?] Impacts FR-009, FR-016.
3. **Definition of "done" / cutover gate** — [NEEDS CLARIFICATION: Can the system go live with a non-empty exception backlog (tracked for clean-up), or must the exception report reach zero unresolved items before cutover?] Impacts SC-006 and the Phase 0 exit criteria.
4. **Post-load clean-up ownership & timeline** — Who on the BumbleB team owns resolving the exception report, and on what timeline? (M10 open question.) *Process question — to confirm in clarify, not blocking the spec.*
5. **Donor contact backfill ownership** — Who is responsible for backfilling the mostly-blank donor point-of-contact / email / phone, and through what channel (the M01 flow, or a manual admin pass)? (M02 open question, surfaces here because donors load during migration.)
6. **Idempotency expectation** — Is re-running the full migration the agreed clean-up loop (fix source → re-run), or should corrections be applied as in-place edits after the first load? Affects FR-006 / SC-009 framing.
