# ADR-0005: Deferred items

**Status:** Deferred (tracked, not dropped)

## Implementation Lead hierarchy
The reporting-manager structure above Implementation Leads is unresolved (some manager references aren't present as employees). **Deferred to a research item.** M05 ships with a self-referencing manager FK that can represent the hierarchy once defined; the org-chart logic and any approval flows wait for that research.

## Donor portal
Authenticated donor access is a **later feature**. M09 initially delivers internal governance + curated, PII-safe impact views; a donor-facing portal is a future increment layered on M09.
