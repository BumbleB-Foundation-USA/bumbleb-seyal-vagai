# M01 — Identity, Roles & Access

**Phase:** 1 · **Status:** Brief · **Spec folder:** _tbd_

## Purpose
Authentication, the role model, and role-/assignment-scoped access for every user class.

## In scope
- BetterAuth integration (email + OTP, OTP-first, long-lived sessions).
- Role model: staff taxonomy + TC + report-consumer tiers (Board, Volunteer, Donor, HM).
- Sign-up / profile-completion flow that backfills missing email/phone.
- Linking an authenticated `user` to its `employee` / `coordinator` / HM scope.

## Out of scope (for this module)
- Donor portal logins (deferred, ADR-0005).
- Org-hierarchy approval flows (depends on lead-hierarchy research, ADR-0005).

## Key entities (see `docs/data-model.md`)
`user`, `role`. References `employee`, `coordinator`, `school` (HM scope).

## Primary users
All users; admins manage roles.

## Dependencies
Foundational. Needed before M03–M09 expose edit/report access. Consumes M11 (audit).

## Decisions already made
ADR-0003 (auth direction), ADR-0004 (HM access).

## Open questions for `/speckit.clarify`
- Token lifetimes, device binding, revocation, OTP channel & rate-limiting (see ADR-0003).
- Exact role→permission matrix per surface.
