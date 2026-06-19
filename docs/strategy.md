# Kalvi40 School Management System — Implementation Strategy & Module Map

*Prepared from a full review of `Copy_of_Kalvi40_Master_Database.numbers` (9 tabs). This expands the originally scoped field-data-capture app into a complete system of record for BumbleB Trust / Kalvi40.*

---

## 1. Current state — what the spreadsheet actually holds

The workbook is already a recognizable relational model expressed in tabs. Nine tabs, ~1,100 real records across them:

| Tab | Real records | What it is | Role in the model |
|---|---|---|---|
| **Key** | ~18 | Dropdown value lists (Donor Type, Project Model, Roles, Implementation Leads, Departments, config Parameters) | Enumerations / config |
| **District** | 39 (18 active) | TN districts, Tamil names, codes | Dimension |
| **School_Type** | 57 | Type code → Tamil/English name → mgmt category → grade level | Dimension |
| **Donor** | 40 | CSR/Non-CSR sponsors + (sparse) contacts | Dimension |
| **Project** | 42 | One project per **Donor × District**; finance code, model, status, school/TC counts | Operational |
| **School** | 408 | The central master: profile, HM, counts, geo, donor/project/type links | Operational (core) |
| **Employee** | 31 (28 active) | Staff: role, dept, reporting manager, lifecycle, district/project | Operational (workforce) |
| **TC** | 262 (208 active) | Teaching Coordinators placed at schools, in hiring batches | Operational (workforce) |
| **Contractor** | 15 | Contract staff by department | Operational (workforce) |

**Relationship map (as currently linked by name strings):**

```
Donor ──< Project ──< School >── School_Type
                 │        │
            District ─────┤
                 │        └──< TC (Teaching Coordinator)
                 │
Employee ──(Implementation Lead / Project Coordinator)──> Project
Employee ──(Reporting Manager, self-reference)──> Employee
```

**Key structural facts confirmed in the data:**
- `Project Code` is a composite string: *"Donor - District"* (e.g. `RAMCO CEMENTS - Ariyalur`). This is the de-facto join key used by School and TC.
- Sum of `Number of Schools` across the 42 projects = **408** — exactly matches the School table. The rollups are internally consistent.
- Two delivery models: **LITE** (192 schools, no on-site coordinator) vs **TC** (216 schools, a Teaching Coordinator placed on-site).
- Workforce: 31 employees (Operations-heavy: 24 of 31; dominant role *Implementation Lead* ×18), 262 TCs in 4 hiring batches (Batch 3 = 192), 15 contractors.
- 7 departments (Operations, Admin, HR, Finance, Management, Content, Communication) and a 9-level role taxonomy (CEO → Operations Head → Head of Dept → Manager → Asst Manager → Project Manager → Implementation Lead → Coordinator → Jr Coordinator), plus the TC field role.
- Headmaster coverage is excellent: 405/408 schools have an HM name + contact.

---

## 2. Data-quality findings — the migration backlog

These are the concrete issues the spreadsheet has accumulated. Each one is a rule the new system must enforce so it can't recur:

1. **No stable primary keys.** `Employee ID` and `TC ID` columns exist but are **empty** — names are the de-facto key today. Must mint canonical IDs on migration.
2. **All cross-table links are name strings**, not enforced foreign keys. One typo or rename orphans a record. (`Current Donor`, `Implementation Lead`, `TC Name`, `Project Coordinator`, composite `Project Code`.)
3. **Phone numbers are inconsistent and lossy** — stored as floats (`9843585455.0`), with embedded spaces (`79044 91923`), or as text; some schools hold *multiple* comma-separated Kalvi40 registration numbers. Phones matter because Kalvi40 app registration is keyed on them.
4. **Lifecycle state is unmaintained.** School `Status` is **"TBD" for 366 of 408**; the Project tab carries ~950 fill-down placeholder rows behind the 42 real ones.
5. **Geolocation is sparse** — coordinates on only 19/408 (5%), map links on 174/408 (43%).
6. **Bidirectional denormalization.** School stores TC/Type/Donor/Lead by name *and* TC stores its School; Implementation Lead is duplicated across School, Project, Employee, and Key. No single source of truth.
7. **Donor contacts are mostly blank** (point-of-contact / email / phone) — only donor names are reliable.
8. **Mixed identity domains.** Only **9 of 31 employees** use `@bumbleb.org.in`; the rest use personal Gmail. TCs have email on 77% and phone on 80% of records. This directly shapes the auth strategy.
9. **Dangling hierarchy.** Reporting-manager references (e.g. *Karthick Subramaniam*) aren't all present as employees themselves.

---

## 3. Target system — from spreadsheet to system of record

**The core shift:** today the spreadsheet is *the system of record*, edited manually, linked by name, single-file. The new system makes **D1 the normalized system of record** — stable IDs, enforced foreign keys, validation rules, role-based edit access, and an audit trail. The spreadsheet is retired (exported once, then read-only as a backup).

**Target data model (entities → keys):**

- **Dimensions:** `district`, `school_type`, `donor`, `department`, `role`
- **Operational:** `project` (FK donor, district; assigned lead/coordinator employees), `school` (FK district, school_type, donor, project; embeds HM contact, grade-band counts, geo, status), `employee` (FK department, role, self-ref manager; M:N to project via assignment), `coordinator/tc` (FK school, district, project, batch), `contractor` (FK department)
- **Capture (new, from original scope):** `usage_session`, `feedback` — both FK to school (and optionally coordinator + lesson)
- **Reference to Kalvi40 app:** `lesson/content` catalog so content feedback maps to real lessons

**How the original frictionless capture app fits:** it becomes **Module 7** — a consumer of this master data. A coordinator opening the capture app no longer types a school name; they pick from *their* assigned schools (resolved through the workforce model), which is what makes the "frictionless" constitution principle achievable. Clean master data is the prerequisite for a fast capture UX.

---

## 4. Implementation strategy — phasing

The ordering is deliberate: **you cannot build a frictionless capture app on top of name-string master data.** Normalize first, capture second.

**Phase 0 — Migration & system of record.** Stand up the schema, run the one-time ETL, mint IDs, resolve name-links to FKs, normalize phones, produce an exception report for the BumbleB team to clean. *Deliverable: clean, queryable master data in D1.*

**Phase 1 — Identity + master-data CRUD.** BetterAuth, role model, and admin screens to manage schools / districts / types / donors. *This replaces day-to-day spreadsheet editing.*

**Phase 2 — Projects & workforce.** Project management (donor×district, finance codes, coverage rollups) and workforce management (employees with org hierarchy & lifecycle; TCs with school placement & batches; contractors).

**Phase 3 — Field data capture.** The original frictionless, offline-tolerant app: student counts, smart-board vs tablet, lesson/general/on-the-spot feedback — now sitting on real master data.

**Phase 4 — Reporting & impact.** Operational dashboards, governance reporting, and curated donor/volunteer impact views.

---

## 5. Module breakdown for SpecKit

Each module below becomes one `/specify` spec. Phase tags show suggested sequence; you set final priority.

| # | Module | Purpose & key data | Primary users | Phase |
|---|---|---|---|---|
| **M1** | **Identity, Roles & Access** | BetterAuth; role model derived from the 9-level staff taxonomy + TC + report-consumer tiers (board/volunteer/donor); per-class auth method | All | 1 |
| **M2** | **Organization & Reference Data** | Districts, school types, departments, roles, donors — the dimensions + the `Key` enumerations | Admins | 0–1 |
| **M3** | **School Master** | The 408-school registry: bilingual profile, HM contact, grade-band counts, geo, status, donor/project/type links | Admins, leads | 1 |
| **M4** | **Project & Donor Management** | Projects (donor×district), finance codes, LITE/TC model, status, school/TC coverage rollups; donor records | Operations, finance | 2 |
| **M5** | **Workforce — Employees** | Staff directory, departments, roles, self-referencing org hierarchy, lifecycle (DOJ/DOL/status), project assignment | HR, management | 2 |
| **M6** | **Workforce — Coordinators (TC) & Contractors** | TC placement at schools, hiring batches/cohorts, lifecycle; contractor records | Operations, HR | 2 |
| **M7** | **Field Data Capture** | Frictionless, offline: student-usage counts, smart-board vs tablet, lesson-content / general / on-the-spot feedback | Teachers, TCs, school reps | 3 |
| **M8** | **Reporting & Dashboards** | Operational views: coverage, adoption, modality split, feedback themes, workforce status | Trust staff, leads | 3–4 |
| **M9** | **Governance & Donor/Volunteer Impact** | Curated, PII-safe, shareable rollups and impact stories; exports & scheduled digests | Board, donors, volunteers | 4 |
| **M10** | **Data Migration & Quality** | One-time ETL + ongoing validation rules: ID minting, FK resolution, phone normalization, dedup, exception reporting | Engineering, admins | 0 (then ongoing) |
| **M11** | **Platform (cross-cutting)** | Offline sync, Tamil/English localization, audit log, media/R2 attachments, performance budgets | — | All |

---

## 6. Migration plan (Phase 0 detail)

1. **Schema-first.** Define D1 tables with FK constraints and `NOT NULL` where the data supports it.
2. **Mint stable IDs** — `SCHOOL-####`, `EMP-####`, `TC-####`, `PROJ-####`, `DONOR-####` — and keep the original name as a `legacy_name` for traceability.
3. **Resolve name-links to FKs.** Parse `Project Code` into donor + district; match School/TC by school name; emit an **orphan report** for anything that doesn't resolve.
4. **Normalize phones** to E.164 (`+91…`): strip spaces and trailing `.0`, split multi-number Kalvi40 registrations into a child table.
5. **Resolve lifecycle.** Convert the 366 School "TBD" statuses (active/inactive decision by the team); drop the ~950 placeholder Project rows.
6. **Collapse denormalization** to a single source of truth (e.g. TC↔School linkage held once; Implementation Lead is an employee FK, not a repeated string).
7. **Backfill geo** where map links exist; flag the rest for field collection (ties into M7).
8. **Drop the throwaway columns** (`…to be deleted after verification`, `…Need to check`) after the team signs off on the verified figures.
9. **Validation report** out to the BumbleB team for a clean-up pass before and after load.

---

## 7. Open decisions to resolve before writing specs

1. **System of record vs. sync** — recommend full cutover (retire the spreadsheet). Confirm.
2. **Terminology** — is the "TC / Teaching Coordinator" the same role you earlier called the *paid parent worker*? And do TCs use the capture app (they almost certainly should)?
3. **Auth method per user class** — given the data: employees → org email / OTP (but only 29% have official email today); TCs/coordinators → **phone OTP** (phone is the most complete field); report-consumers → email + role. Confirm the split.
4. **Headmaster as a user?** — HM coverage is 405/408. Are headmasters contacts only, or login-capable users who can do capture for LITE (no-TC) schools?
5. **Org hierarchy top** — who sits above the Implementation Leads (the dangling `Reporting Manager` references)?
6. **Donor portal** — read-only impact pages, or do donors need authenticated logins?

---

*Next step: pick the module priority order (and react to the phasing). The natural starting pair is **M10 + M2/M3** — get clean master data into D1 — after which everything else has solid ground to build on.*
