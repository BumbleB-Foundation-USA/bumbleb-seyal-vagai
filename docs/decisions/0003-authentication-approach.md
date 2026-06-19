# ADR-0003: Authentication approach

**Status:** Accepted (needs deeper design)

## Context
Users span low-tech field staff (TCs, HMs) and office staff. Contact data is incomplete (employee official email 29%, TC email 77%, TC phone 80%). Friction must be minimal (constitution, Principle I).

## Decision (direction)
- Support **both email and OTP** sign-in via BetterAuth; **OTP-first** as the frictionless default (phone for TCs/HMs, who are phone-centric).
- **Long-lived sessions / refresh tokens** so a user rarely re-authenticates; trusted-device handling to avoid repeat OTPs.
- A **sign-up / profile-completion flow** that also backfills missing email/phone — making onboarding double as data cleanup for records the migration can't resolve.

## Open questions (deeper design needed)
- Token/session lifetimes and rotation; device binding and revocation.
- OTP channel (SMS/WhatsApp) and rate-limiting/abuse protection.
- BetterAuth plugin selection (email-OTP, phone-OTP, magic link) and how `user` links to `employee` / `coordinator` / HM scope.
- Account recovery for users with no email.

## Consequences
- Drives M01 (Identity, Roles & Access). Revisit this ADR before M01's `/speckit.plan`.
