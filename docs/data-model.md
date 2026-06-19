# Canonical Data Model (shared contract)

*Every module spec references this file. Changes here are PR-reviewed because multiple modules depend on them. Derived from the master spreadsheet, normalized.*

## Conventions
- Every table has a system-minted stable primary key (e.g., `school_id = SCHOOL-0001`). Original spreadsheet names are retained as `legacy_name` for traceability during migration.
- All relationships are foreign keys. **No name-string links.**
- Phone numbers stored normalized to E.164 (`+91…`).
- Bilingual fields stored as `*_ta` / `*_en` pairs.

## Dimensions
- **district**(`district_id`, name_en, name_ta, code) — 39 rows
- **school_type**(`school_type_id`, code, name_en, name_ta, mgmt_category, grade_level) — 57 rows; mgmt_category ∈ {Govt, GovtAdiDravidar, GovtTribal, GovtAided, Panchayat, Private, CSI}
- **donor**(`donor_id`, name, donor_type {CSR, Non-CSR}, poc_name?, email?, phone?, postal_address?) — 40 rows
- **department**(`department_id`, name) — Operations, Admin, HR, Finance, Management, Content, Communication
- **role**(`role_id`, name, scope) — staff taxonomy (CEO, Operations Head, Head of Dept, Manager, Asst Manager, Project Manager, Implementation Lead, Coordinator, Jr Coordinator) + TC + report-consumer tiers (Board, Volunteer, Donor, HM)

## Operational entities
- **project**(`project_id`, donor_id→donor, district_id→district, finance_code?, model {LITE, TC}, status, start_period, end_period, lead_employee_id?→employee, coordinator_employee_id?→employee) — 42 rows. Legacy composite key was `"Donor - District"`.
- **school**(`school_id`, name_en, name_ta, school_type_id→school_type, district_id→district, donor_id→donor, project_id→project, mgmt_category, block?, model {LITE, TC}, status {Active, Inactive}, hm_name, hm_phone, postal_address?, map_link?, lat?, lng?) — 408 rows.
  - **school_grade_count**(`school_id`→school, band {1-2,3-5,6-8,9-10,11-12}, student_count, teacher_count) — replaces the spreadsheet's wide count columns.
  - **school_kalvi40_number**(`school_id`→school, phone_e164) — child table; a school may register multiple app numbers.
- **employee**(`employee_id`, name, email?, phone?, role_id→role, department_id→department, manager_employee_id?→employee (self-ref), doj, dol?, status {Active, Inactive}, district_id?→district) — 31 rows.
  - **employee_project**(`employee_id`→employee, project_id→project, assignment_role) — M:N (Implementation Lead / Project Coordinator assignments).
- **coordinator** (TC) (`tc_id`, name, email?, phone?, school_id→school, district_id→district, project_id→project, batch, doj, dol?, status {Active, Inactive}) — 262 rows.
- **contractor**(`contractor_id`, name, email?, phone?, department_id→department, joining_date, leaving_date?, status) — 15 rows.

## Capture entities (new)
- **usage_session**(`session_id`, school_id→school, tc_id?→coordinator, recorded_by→user, date, student_count, modality {smartboard, tablet, both}, lesson_id?→lesson, created_offline_at, synced_at?)
- **feedback**(`feedback_id`, school_id→school, recorded_by→user, type {lesson_content, general, on_the_spot}, lesson_id?→lesson, rating?, text?, media_r2_key?, created_offline_at, synced_at?)
- **lesson** (reference/catalog from the Kalvi40 app)(`lesson_id`, subject, title_ta, grade_band) — enables content feedback to map to real lessons.

## Identity
- **user**(`user_id`, principal {email|phone}, auth_method, linked_employee_id?/tc_id?/hm_school_id?, role_id→role, status) — BetterAuth-backed; links an authenticated principal to a workforce record and role scope.

## Relationship summary
```
donor 1─* project 1─* school *─1 school_type
                 │          │
          district 1────────┤
                 │          1─* coordinator(TC)
                 │          1─* usage_session, feedback
employee *─* project   (via employee_project)
employee 1─* employee  (manager self-ref)
user 1─1 {employee | coordinator | hm}   (role + assignment scope)
```
