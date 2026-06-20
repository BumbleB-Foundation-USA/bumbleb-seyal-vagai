# ADR-0006: OTP delivery provider for India

**Status:** Proposed — leaning **Plivo Verify**; final determination at M01 `/speckit-plan`

## Context

Authentication is OTP-first (ADR-0003), with phone OTP as the frictionless default for phone-centric field staff (TCs, HMs) and email OTP for office/board/donor users. All infrastructure is on Cloudflare (Workers, D1, R2; BetterAuth). We need to choose how OTPs are actually delivered to Indian phone numbers.

Two facts dominate the decision:

1. **There is no native Cloudflare SMS/voice service.** A Worker generates and verifies the OTP and stores state in D1/KV; the only external hop is an outbound `fetch()` to a delivery provider's REST API. Every candidate integrates this way, so "everything on Cloudflare" is preserved — only the last hop leaves our infra.
2. **BumbleB is a US entity** (the Cloudflare account is `info@bumblebusa.org`). India delivery is gated by entity domicile:
   - **SMS:** TRAI **DLT** registration is mandatory for transactional SMS to India, and **only an India-registered entity (PAN/GST) can register.** A US entity must ride a provider's pre-registered sender IDs/templates, or use international (ILDO) routes that cost 3–5× more and deliver worse.
   - **WhatsApp:** India's **domestic** authentication rate is ≈ ₹0.115/msg (~$0.0014), but the **authentication-international rate ($0.0304/msg, ~20×)** applies when the WhatsApp Business Account is registered outside India.

**Biggest cost lever (separate from provider choice):** if a Kalvi40 / BumbleB *Indian* Trust entity exists, register **that** entity for DLT and for the WhatsApp Business Account to unlock domestic rates. The provider's invoice can still be paid from a US card — compliance-entity and paying-card are independent decisions.

## Options considered

Pricing is approximate, 2026 figures (≈ $1 = ₹85); confirm against live pricing pages at plan stage.

### Option 1 — Twilio Verify (turnkey, US-account-native)
One API for SMS + WhatsApp + Email + Voice with automatic fallback, fraud protection, and Twilio-managed India sender-ID/route compliance.
- **Pros:** fastest to ship; multichannel auto-fallback; no DLT work; US account, USD billing, **US debit/credit card direct**; strong BetterAuth/Workers examples.
- **Cons:** most expensive; India SMS deliverability historically weaker than native players; sender-ID registration can take days.
- **Pricing:** ~**$0.05 / verification** + channel fee (India SMS ≈ ₹0.40–0.50 / ~$0.005 per segment; WhatsApp from $0.005; email ~$0.0013).

### Option 2 — Plivo Verify (best balance for a US org) — *leading candidate*
US-based CPaaS like Twilio, but the Verify product has a **$0 platform fee** (pay only delivery), ships pre-registered India sender IDs (e.g. `PLVRFY`) for instant go-live without your own DLT, and includes free Fraud Shield.
- **Pros:** no per-verification markup; instant India SMS; US-based, **USD billing, US card direct**; multichannel; low integration friction from a Worker.
- **Cons:** smaller ecosystem than Twilio; branded/high-volume SMS may still want your own DLT (Indian entity); fewer turnkey verification-UX extras.
- **Pricing:** **$0 verification fee** + India SMS delivery (~$0.04–0.05/segment) or WhatsApp/voice rates.

### Option 3 — India-native OTP provider (MSG91 / 2Factor.in / Message Central)
Purpose-built OTP platforms with smart SMS → WhatsApp → email → voice fallback, cheapest per-OTP rates, DLT handled under the provider's Principal Entity.
- **Pros:** cheapest (MSG91 ≈ ₹0.15/SMS OTP ~$0.0018; WhatsApp ₹0.35/conv); OTP-specific features; INR billing avoids forex on their side; live in minutes under pre-approved templates.
- **Cons:** best rates/branding want **your own DLT = an Indian entity**; **INR billing via Indian gateways** → US card adds ~3% forex and more declines; **auto-recharge is fragile** with foreign cards under RBI's E-mandate Framework 2026 (foreign-currency mandates may be rejected; >₹15,000 needs manual OTP each time); India-timezone support; **must confirm foreign-card/PayPal acceptance** before committing.
- **Pricing:** SMS OTP ~₹0.12–0.20 (~$0.0015–0.0024); WhatsApp ~₹0.35 (~$0.004); email near-free. DLT setup if self-registering: ~₹5,900 PE + ₹5,900 header + templates.

### Channel economics (per OTP to an Indian phone)

| Channel | Domestic (Indian entity) | As a US entity | Reach / fit |
|---|---|---|---|
| **SMS** | ~₹0.12–0.50 ($0.0015–0.006) | Same via pre-reg IDs; 3–5× on intl routes | Universal — any phone, no app; best default for TCs/HMs |
| **WhatsApp** | ~₹0.115 ($0.0014) | **$0.0304** (auth-international, ~20×) | Ubiquitous in India; cheap *only* with an Indian WABA |
| **Email** | ~$0.0013 → near-free (Resend/SES/Cloudflare) | Same | Near-free, but only ~29% of employees / 77% of TCs have email |

### Payment / US debit card
- **Twilio / Plivo / Vonage (US-based):** bill in USD, accept **US Visa/Mastercard debit and credit** directly. A **credit** card is preferred for international recurring billing (fewer declines, chargeback protection).
- **Indian providers:** INR billing via Indian gateways → ~3% forex, higher decline rates, fragile auto-recharge; prefer manual prepaid top-ups and confirm foreign-card acceptance first.

## Decision

**Proposed direction (not yet locked):**

- **Lead with Plivo Verify** for phone OTP (SMS, with WhatsApp/voice as fallback channels) — US-card-native, $0 verification fee, instant India delivery, clean `fetch()` integration from a Worker.
- **Add email OTP** (near-free via Resend / Amazon SES / Cloudflare Email) for users who have email — office/admin/board/donor tiers.
- **Decide the entity question early:** confirm whether a Kalvi40 / BumbleB Indian entity exists and can register for DLT + a WhatsApp Business Account. If so, an India-native provider (Option 3) becomes the cheapest at scale and WhatsApp's domestic rate unlocks — still payable from a US card.

**Final provider selection is deferred to M01's `/speckit-plan`**, where it will be made against confirmed live pricing, the entity-registration answer, and a deliverability test.

## Consequences

- Phone OTP delivery is an outbound REST call from a Worker; OTP generation, rate-limiting, and attempt state live in D1/KV. Provider API keys are Worker secrets (`wrangler secret put`).
- BetterAuth's `phoneNumber` plugin (`sendOTP()` callback) and `emailOTP` plugin are the integration seam — the provider is swappable behind that callback, so this choice is **low-lock-in** and safe to finalize at plan stage.
- At project volume (~hundreds of field users authenticating a few times a month), absolute OTP cost is trivial; **deliverability and ops simplicity outweigh per-OTP price** — do not over-optimize fractions of a cent.
- Revisit ADR-0003 alongside this ADR before M01's `/speckit-plan`.

## Open questions (resolve at M01 plan)

- Does a Kalvi40 / BumbleB **Indian entity** exist for DLT + WhatsApp Business Account registration? (Determines domestic vs international rates.)
- Confirm live Plivo Verify India pricing and run a **deliverability test** to real TC/HM numbers before committing.
- WhatsApp as primary vs SMS-only-with-email-fallback — depends on the entity answer and a quick field-coverage check (how many TCs/HMs use WhatsApp).
- Provider account ownership and the recurring-payment card (prefer a credit card; clarify who at BumbleB owns billing).

## References

- Internal: ADR-0003 (authentication approach); `docs/architecture.md`; `docs/strategy.md` §2 (contact-data coverage); M01 brief.
- External (2026): [Message Central — India DLT guide](https://www.messagecentral.com/sms-guideline/india); [Plivo — DLT for foreign entities](https://support.plivo.com/hc/en-us/articles/360046769131-DLT-Registration-Process-for-Sending-SMS-to-India); [Plivo Verify pricing](https://www.plivo.com/verify/pricing/); [Twilio Verify](https://www.twilio.com/en-us/user-authentication-identity/verify); [Twilio India SMS pricing](https://www.twilio.com/en-us/sms/pricing/in); [MSG91 OTP pricing](https://msg91.com/in/pricing/otp); [Meta — authentication-international rates](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing/authentication-international-rates/); [BetterAuth phone-number plugin](https://better-auth.com/docs/plugins/phone-number); [RBI E-mandate Framework 2026](https://the420.in/rbi-2026-e-mandate-framework-recurring-payments-upi-cards-india/).
