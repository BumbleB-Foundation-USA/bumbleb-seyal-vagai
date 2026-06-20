# Feature Specification: Phase 0 — Data Foundation (Migration & Reference Data)

**Feature Branch**: `001-data-migration-quality`

**Created**: 2026-06-19

**Status**: Draft

**Modules**: M10 — Data Migration & Quality · M02 — Organization & Reference Data · **Phase**: 0 (M02 spans 0–1)

**Input**: User descriptions:
- "Let's create a branch and iterate on the specs required for phase 0. Review all the markdowns, create the specs for these phases and surface all open questions that need to be addressed."
- "Let's fold module 02 (organization and reference data) into the current feature, and spec it out. Review the high level scope set forth for this module and add the user stories."

## Overview

Phase 0 stands up the **data foundation** for the Kalvi40 system of record. It folds two tightly-coupled modules into one feature because they share the same tables and ship together:

- **M10 — Data Migration & Quality**: the one-time ETL of the master spreadsheet (`Copy_of_Kalvi40_Master_Database`, 9 tabs, ~1,100 real records) into a clean, normalized **system of record** (ADR-0001, full cutover), plus the ongoing validation rules that keep it clean.
- **M02 — Organization & Reference Data**: the admin/operations surface that **manages the dimension tables everything else references** — districts, school types, departments, roles, donors, and the `Key`-tab enumerations — replacing day-to-day spreadsheet editing for that data.

They belong together: M10 *loads* the dimension tables; M02 *maintains* them. Once migration lands the reference data, M02 is how the team curates it going forward (constitution, Principle III — integrity by design; Principle V — bilingual; Principle VI — role-scoped access).

The work has three strands:

1. **A one-time load (ETL, M10)** — move every tab into the canonical schema in `docs/data-model.md`, minting stable IDs and replacing name-string links with enforced foreign keys.
2. **Reference-data management (M02)** — CRUD for the dimensions and controlled lookups, with first-class Tamil/English names and referential safety.
3. **An ongoing quality regime (M10)** — an exception report the BumbleB team resolves, plus validation rules that prevent the same defects (orphans, bad phones, missing required fields) from recurring once the system is live.

This spec stays at the level of **what** the data foundation must do and **why**, referencing `docs/data-model.md` as the shared schema contract and `docs/migration-plan.md` for the agreed transformation steps. Concrete tooling, UI, and load mechanics are decided in `/speckit-plan`.

**Module scope boundary**: M02 manages the *dimension / reference* entities only. The operational entities that reference them — schools (M03), projects (M04), workforce (M05/M06) — are out of scope here and get their own specs.

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

### User Story 6 - Manage core reference dimensions (Priority: P2)

An admin maintains the dimension tables the rest of the system references — districts, school types, departments, and roles — through create / edit / deactivate actions, without touching a spreadsheet or asking engineering. School-facing names are entered in both Tamil and English, and each dimension carries its defining attributes (district `code`; school-type `mgmt_category` and `grade_level`; role `scope`).

**Why this priority**: This is the M02 MVP and the capability that lets the team retire the spreadsheet as the *editing* surface for reference data (migration only retires it as the system of record). Every other module depends on these lists being correct and current. It follows the load (US1) because there is nothing to manage until the data is there.

**Independent Test**: After the dimensions are loaded, create a new district with bilingual names and a code, edit a school type's management category, and deactivate an unused department — confirm each change persists and is reflected wherever that dimension is offered as a choice.

**Acceptance Scenarios**:

1. **Given** an admin on the reference-data surface, **When** they create a district with an English name, a Tamil name, and a code, **Then** the district is saved with a system-minted `district_id` and becomes available wherever districts are referenced.
2. **Given** an existing school type, **When** an admin sets its `mgmt_category`, **Then** a value within the allowed set ({Govt, GovtAdiDravidar, GovtTribal, GovtAided, Panchayat, Private, CSI}) is saved and an out-of-set value is rejected.
3. **Given** a role, **When** an admin edits its name or `scope`, **Then** the change is saved and applied wherever the role is referenced.
4. **Given** a department not referenced by any record, **When** an admin deactivates it, **Then** it no longer appears in pickers for new records but remains intact for historical records.

---

### User Story 7 - Manage donors and complete their contact details (Priority: P2)

An admin maintains donor records — name, `donor_type` (CSR / Non-CSR), and the point-of-contact details (POC name, email, phone, postal address) that migration mostly found blank. This is where the sparse donor contacts flagged during migration get backfilled, and where new donors are added as the trust signs them.

**Why this priority**: Donors are a dimension referenced by projects and schools, and donor-contact completeness is a known migration gap. Giving admins a place to fill it closes that gap without an engineering pass. Separated from US6 because donors carry contact and classification nuance the other dimensions don't.

**Independent Test**: Open a migrated donor whose POC/email/phone are blank, fill them in, and confirm they persist as valid (E.164 phone); create a brand-new donor with a `donor_type`; confirm both appear wherever donors are selectable.

**Acceptance Scenarios**:

1. **Given** a migrated donor with blank contact fields, **When** an admin enters a POC name, email, and phone, **Then** the values are saved, the phone is stored in E.164 form, and an invalid phone or email is rejected.
2. **Given** a new sponsor, **When** an admin creates a donor with a name and a `donor_type` of CSR or Non-CSR, **Then** the donor is saved with a system-minted `donor_id`.
3. **Given** a donor referenced by one or more projects, **When** an admin attempts to delete it, **Then** deletion is prevented and deactivation is offered instead (see US9).

---

### User Story 8 - Govern controlled enumerations / managed lookups (Priority: P3)

An admin manages the `Key`-tab-derived lookups (e.g. donor type, project delivery model {LITE, TC}, management categories) as governed lists. Some lists are **system-fixed** because their values carry behaviour the application relies on; others are **admin-editable**. Adding or renaming a value in an editable list flows through to every dependent picker and validation without a code change.

**Why this priority**: Controlled vocabularies keep data consistent, but the system can ship with sensible defaults for which lists are fixed vs editable and refine later. It is governance polish on top of the core CRUD.

**Independent Test**: Add a new value to an admin-editable lookup and confirm it appears in the dependent picker; attempt to change a system-fixed value (e.g. the LITE/TC model set) and confirm the system prevents it with an explanation.

**Acceptance Scenarios**:

1. **Given** an admin-editable lookup, **When** an admin adds a new allowed value, **Then** the value becomes selectable wherever that lookup is used, with no deployment required.
2. **Given** a system-fixed lookup (e.g. delivery model {LITE, TC}), **When** an admin attempts to add, rename, or remove a value, **Then** the action is prevented and the reason is shown.
3. **Given** a lookup value currently in use by records, **When** an admin renames it, **Then** referencing records reflect the new label without losing their link (the underlying identifier is stable).

---

### User Story 9 - Referential safety for reference data (Priority: P3)

The reference-data surface cannot be used to break integrity. A dimension row still referenced by operational records (a district with schools, a donor with projects, a role held by staff) cannot be hard-deleted; the admin is steered to **deactivate** instead, preserving history while removing the row from new-record pickers. Bilingual completeness is encouraged — saving a dimension with only one script present surfaces a warning.

**Why this priority**: This protects the integrity guarantees migration establishes (US5 / FR-020) from being undone through the admin UI. Important for trustworthiness, but it layers on top of the CRUD surface.

**Independent Test**: Attempt to delete a district that has schools attached and confirm it is blocked with a clear reason and a deactivate option; deactivate it and confirm existing schools keep their link while the district disappears from new-record pickers.

**Acceptance Scenarios**:

1. **Given** a dimension row referenced by at least one operational record, **When** an admin attempts to delete it, **Then** deletion is prevented with a message naming what still references it, and deactivation is offered.
2. **Given** a deactivated dimension row, **When** admins create new records, **Then** the row is not offered as a choice, but records that already reference it remain valid and display it.
3. **Given** an admin saving a bilingual dimension with only one script filled, **When** they save, **Then** the record saves but a missing-translation warning is recorded/surfaced (consistent with migration's bilingual flagging, FR-019).

---

### Edge Cases

- **Reporting-manager points to a non-employee** — some manager references (e.g. *Karthick Subramaniam*) are not present as employee rows. The self-referencing manager FK is left null and the dangling reference is reported (org-hierarchy resolution is deferred per ADR-0005).
- **Multiple plausible matches for one name** — a name resolves to more than one candidate parent; treated as an ambiguous duplicate (US2), never auto-resolved.
- **Whitespace / case / script variants of the same name** — "RAMCO CEMENTS" vs "Ramco Cements " should resolve to the same donor; near-but-not-exact matches are surfaced for confirmation.
- **Bilingual name present in only one script** — a school or type with a Tamil name but no English (or vice versa) loads with the available script and flags the missing pair rather than blocking.
- **Geo present as a map link but no coordinates** — coordinates are backfilled where a map link exists; the remaining ~95% without coordinates are flagged for later field collection (ties to M07), not blocked.
- **Re-running the migration** — a second run must not create duplicate IDs or double-load child records; the load is repeatable to a consistent end state.
- **Zero-real-data tab anomalies** — a tab whose only rows are placeholders/blank yields no loaded records and a note in the report, not an error.

*Reference-data management (M02):*

- **Duplicate code or name on create** — creating a district with an existing `code`, or a donor with a name that already exists, is flagged for confirmation (possible duplicate) rather than silently accepted.
- **Reactivating a deactivated dimension** — a previously deactivated row can be reactivated and returns to pickers, retaining its original ID and history.
- **Editing across the fixed/editable boundary** — attempting to edit a system-fixed enumeration is blocked; the boundary between fixed and editable lists is explicit, not decided per-edit.
- **Concurrent edits to the same reference row** — last-write-wins per the constitution unless a conflict policy says otherwise; reference data is low-churn, so this is rare.

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

**Organization & reference-data management (M02)**

- **FR-024**: Admins MUST be able to create, view, edit, and deactivate **districts** (`name_en`, `name_ta`, `code`).
- **FR-025**: Admins MUST be able to create, view, edit, and deactivate **school types** (`code`, `name_en`, `name_ta`, `mgmt_category` constrained to the allowed set, `grade_level`).
- **FR-026**: Admins MUST be able to create, view, edit, and deactivate **departments** and **roles** (role carrying its `scope`).
- **FR-027**: Admins MUST be able to create, view, edit, and deactivate **donors** (`name`, `donor_type` ∈ {CSR, Non-CSR}, `poc_name`, `email`, `phone`, `postal_address`), including backfilling contact fields the migration left blank.
- **FR-028**: The system MUST treat Tamil and English names as first-class for every bilingual dimension — both editable, both stored, neither auto-translated (constitution, Principle V).
- **FR-029**: The system MUST manage the `Key`-derived controlled lookups (donor type, delivery model, management categories, etc.), distinguishing **system-fixed** lists from **admin-editable** lists, and MUST prevent edits to fixed lists.
- **FR-030**: Changes to an admin-editable lookup MUST propagate to all dependent pickers and validations without a code change or redeploy.
- **FR-031**: Renaming a lookup or dimension value MUST preserve referencing records' links via the stable identifier — no relink required and no orphaning.
- **FR-032**: The system MUST prevent hard-deletion of any reference row still referenced by an operational record and MUST offer deactivation instead; deactivated rows MUST be excluded from new-record pickers while remaining valid for existing references.
- **FR-033**: Reference-data create/edit/deactivate actions MUST be restricted to admin/operations roles (constitution, Principle VI).
- **FR-034**: Reference-data edits MUST be captured in an audit trail (who changed what, when), since these tables are shared across modules (audit mechanism owned by M11).
- **FR-035**: Reference records created after migration MUST receive system-minted stable IDs consistent with the migration's ID scheme, so post-migration records are indistinguishable in form from migrated ones.

### Key Entities *(include if feature involves data)*

The migration touches **all** entities in `docs/data-model.md`. The ones most affected by transformation rules:

- **district / school_type / donor / department / role** — dimension tables loaded first so operational records have valid FK targets; carry bilingual names where applicable.
- **project** — 42 real records; composite `"Donor - District"` code parsed into FKs; ~950 placeholder rows excluded.
- **school** — the 408-row core; resolves donor/project/type/district name links to FKs, normalizes HM phone, holds grade-band counts and geo, resolves `TBD` status, and owns `school_kalvi40_number` (multiple app numbers) as a child table.
- **employee** — 31 records; minted IDs, self-referencing manager FK (left null where the manager isn't an employee), project assignments via `employee_project`.
- **coordinator (TC)** — 262 records; minted IDs, single-source TC↔school linkage, hiring batch retained.
- **contractor** — 15 records; minted IDs, department FK.
- **Exception report** *(migration artifact)* — the authoritative list of unresolved orphans, missing fields, ambiguous duplicates, and dangling references for the team to clean.

Under **M02**, the dimension entities above (`district`, `school_type`, `donor`, `department`, `role`) are not only loaded but **maintained** — created, edited, deactivated/reactivated, with bilingual names and referential safety. Additionally:

- **Controlled lookups / `Key` enumerations** *(M02-managed)* — governed value lists (donor type, delivery model {LITE, TC}, management categories) classified as **system-fixed** or **admin-editable**; referenced by dimension and operational records, edited through stable identifiers so labels can change without breaking links.

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
- **SC-010**: An admin can create, edit, or deactivate any reference dimension (district, school type, department, role, donor) entirely through the application — zero spreadsheet edits, no engineering involvement.
- **SC-011**: 100% of bilingual dimensions can store and display both a Tamil and an English name.
- **SC-012**: Attempts to delete a referenced reference row are prevented 100% of the time, with deactivation offered — no orphan is ever created through the reference-data surface.
- **SC-013**: A new value added to an admin-editable lookup is selectable in dependent pickers immediately, with no deployment.
- **SC-014**: Every donor record can hold complete contact details (POC, email, phone, address), enabling the team to drive migration-flagged blank donor contacts toward full coverage.
- **SC-015**: 100% of reference-data edits are attributable to a user via the audit trail.

## Assumptions

- **Source format**: The migration consumes a one-time export of `Copy_of_Kalvi40_Master_Database` (9 tabs). The live spreadsheet is frozen at the agreed export and retired afterward (ADR-0001); there is no two-way sync.
- **Schema contract**: `docs/data-model.md` is the authoritative target schema; entity shapes, FK directions, and bilingual/phone conventions follow it. Schema changes are PR-reviewed there, not invented here.
- **ID scheme**: Stable IDs use the `ENTITY-####` prefix convention from `docs/migration-plan.md`; zero-padding width is an implementation detail for `/speckit-plan`.
- **Contact backfill**: Missing employee/TC email and phone, and sparse donor contacts, are **not** fully resolved by this migration. They are flagged for backfill through the M01 sign-up / profile-completion flow (per `docs/migration-plan.md`); this spec only normalizes and reports what exists.
- **Org hierarchy**: The reporting-manager structure above Implementation Leads is deferred (ADR-0005). The self-referencing manager FK is populated where the manager is a known employee and left null (and reported) otherwise.
- **Geo**: Only ~5% of schools have coordinates and ~43% have map links; full geo coverage is a later field-collection effort (M07), not a Phase 0 exit condition.
- **Privacy**: No student PII is migrated; student data is counts only (constitution, Principle IV).
- **Validation surface**: Ongoing validation rules (FR-020–022) are defined here as requirements; whether they are enforced at the database, application, or both layers is an implementation choice for `/speckit-plan`.
- **M02 surface**: Reference-data management is an admin/operations capability (not field-facing); its UI/UX is designed in `/speckit-plan`. "Deactivate" is the soft-delete mechanism; hard delete is reserved for unreferenced rows only.
- **Fixed vs editable lookups (default)**: Lists whose values drive behaviour — `donor_type` {CSR, Non-CSR}, delivery `model` {LITE, TC}, and the `mgmt_category` set — are treated as **system-fixed** by default; descriptive/extensible lists are admin-editable. The exact split is to be confirmed (see Open Questions).
- **Audit**: The audit-trail mechanism is a cross-cutting M11 concern consumed here, not re-specified; this spec only requires that reference-data edits are audited.

## Open Questions

These need team answers before or during `/speckit-clarify`. The three most decision-critical are also embedded as `[NEEDS CLARIFICATION]` in context:

1. **TBD status resolution rule** — [NEEDS CLARIFICATION: How should the 366 schools with `Status = "TBD"` be converted to Active/Inactive? Options: (a) a derivable rule, e.g. "Active if it belongs to a real project / has an active TC", (b) a team-supplied list/decision per school, (c) default all TBD → Active pending review.] Impacts FR-015 and SC-004.
2. **Source-of-truth for ambiguous duplicates** — [NEEDS CLARIFICATION: When two records describe the same real-world entity or a denormalized fact conflicts across tabs, which source wins? e.g. is School's copy or TC's copy authoritative for the TC↔school link?] Impacts FR-009, FR-016.
3. **Definition of "done" / cutover gate** — [NEEDS CLARIFICATION: Can the system go live with a non-empty exception backlog (tracked for clean-up), or must the exception report reach zero unresolved items before cutover?] Impacts SC-006 and the Phase 0 exit criteria.
4. **Post-load clean-up ownership & timeline** — Who on the BumbleB team owns resolving the exception report, and on what timeline? (M10 open question.) *Process question — to confirm in clarify, not blocking the spec.*
5. **Donor contact backfill ownership** — Who is responsible for backfilling the mostly-blank donor point-of-contact / email / phone, and through what channel (the M01 flow, or a manual admin pass)? (M02 open question, surfaces here because donors load during migration.)
6. **Idempotency expectation** — Is re-running the full migration the agreed clean-up loop (fix source → re-run), or should corrections be applied as in-place edits after the first load? Affects FR-006 / SC-009 framing.

**Organization & Reference Data (M02):**

7. **Which enumerations are admin-editable vs system-fixed?** The default above treats behaviour-bearing lists (donor type, LITE/TC model, mgmt_category) as fixed and the rest as editable. Confirm the exact boundary — getting this wrong either ossifies lists the team needs to extend, or lets someone edit a value the application logic depends on. (M02 open question.) Impacts FR-029, US8.
8. **Deactivate vs delete semantics** — is soft-deactivation always the rule (never hard-delete, even for unreferenced rows), or is hard-delete allowed only when nothing references a row? (M02.) Impacts FR-032, US9.
9. **Reference-data edit roles** — is "admin/operations" the right edit scope for all reference data, or do some lists (e.g. roles, donors) warrant a narrower role? (M02, ties to M01 role model.) Impacts FR-033.

*Note: items 7–9 have documented defaults in Assumptions, so they are confirm-only and do not block the spec. The donor-contact backfill ownership question (item 5) is shared M10/M02 and already surfaced above.*
