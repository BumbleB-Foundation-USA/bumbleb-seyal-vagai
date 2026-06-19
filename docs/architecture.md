# Architecture

*High-level only. Per-module tech decisions are made in each module's `/speckit.plan` step.*

## Two surfaces, one system of record
1. **Capture surface** — frictionless, offline-tolerant PWA used by TCs, teachers, and HMs in the field. Few taps, no typing, works offline.
2. **Admin & reporting surface** — used by BumbleB staff, leads, board, (later) donors. Master-data management and dashboards.

Both read/write the same normalized **D1** system of record.

## Stack
- **Frontend / app:** Next.js (App Router), deployed to Cloudflare via the **OpenNext** adapter (`@opennextjs/cloudflare`).
- **Compute:** Cloudflare **Workers**.
- **Database:** Cloudflare **D1** (SQLite at the edge) — the system of record. FK constraints enforced; migrations versioned.
- **Object storage:** Cloudflare **R2** — media attachments (optional photo/voice feedback), exports.
- **Auth:** **BetterAuth** — email + OTP, OTP-first, long-lived sessions. See `docs/decisions/0003-authentication-approach.md`.
- **Offline:** Service worker + local queue (e.g., IndexedDB) with background sync to Workers/D1.

## Environments
- Local dev via Wrangler; secrets in `.dev.vars` (gitignored). Staging and production as separate Workers/D1 bindings.

## Cross-cutting concerns (owned by M11)
Offline sync, Tamil/English localization, audit logging, R2 media handling, performance budgets. These are consumed by every feature module rather than re-specified each time.
