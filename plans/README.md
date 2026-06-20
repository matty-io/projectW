# Feature plans & roadmap

Design docs for everything not yet built. **None of these are implemented.** `ARCHITECTURE.md` remains
the source of truth for schema/endpoints; these capture the agreed approach (2026-06-20) so we can
implement later. Files are prefixed so the folder lists in build order.

Two phases:
- **`mvp-*`** — completes the MVP (ARCHITECTURE §11 Week 5 block).
- **`growth-*`** — post-MVP competitive features (ARCHITECTURE §12), backed by
  [competitive-gap-analysis.md](competitive-gap-analysis.md).

---

## Phase 1 — Finish the MVP (do these before the Week 6 soft launch)

| # | Plan | Why this slot | Urgency | Size |
|---|---|---|---|---|
| 1 | [Contact CSV import](mvp-01-contact-csv-import.md) | Beta customers arrive with spreadsheets; builds the shared `S3Service` | **Beta-blocker** | M–L |
| 2 | [Dashboard analytics](mvp-02-dashboard-analytics.md) | Makes the product feel finished for launch; cheap | Beta polish | S–M |
| 3 | [Billing & monetization](mvp-03-billing-monetization.md) | Needed for Week 8 "convert to paid"; fixes the broken "unlimited" pricing | Revenue | **L** |
| 4 | [Quick replies](mvp-04-quick-replies.md) | Inbox productivity; table + endpoints already spec'd | Productivity | **S** |

Note: billing's design (subscription + prepaid wallet) is the highest-leverage decision and worth
locking before launch sets pricing expectations — even though the *code* lands at #3.

## Phase 2 — Close the competitive gap (post-MVP)

| # | Plan | Why it matters | Impact | Size |
|---|---|---|---|---|
| 1 | [WhatsApp Flows (native forms)](growth-01-whatsapp-flows.md) | Booking/lead-gen; Meta hosts rendering → best ROI:effort, fast win | ★★★★ | S–M |
| 1b | [Attach a Flow to a campaign](growth-01b-flow-campaign-attach.md) | Phase B follow-up; initiate flows to a whole audience outside the 24h window | ★★★ | S–M |
| 2 | [No-code flow builder](growth-02-flow-builder.md) | Category's #1 buying criterion; foundation for automation + AI | ★★★★★ | L–XL |
| 3 | [Shopify + abandoned-cart recovery](growth-03-shopify-cart-recovery.md) | Direct ROI story for the core D2C target | ★★★★★ | M–L |
| 4 | [Public API + webhooks + Zapier/Make](growth-04-integrations-api-zapier.md) | Unlocks long-tail integrations cheaply; qualifying criterion | ★★★ | M |
| 5 | [CTWA attribution](growth-05-ctwa-attribution.md) | Wins the performance-marketer D2C buyer | ★★★ | M |
| 6 | [AI agent (LLM/Claude)](growth-06-ai-agent.md) | Differentiation frontier; builds on the flow builder | ★★★★ | L |

> Growth-phase file numbers are **build order**, not the gap-table numbering in the research doc —
> WhatsApp Flows leads as the quick win, the flow builder is the strategic centerpiece. Dependencies:
> the flow builder (growth-02) underpins the AI agent (growth-06); CTWA (growth-05) consumes events
> from Flows (growth-01) and Shopify (growth-03).

## Cross-cutting business rules (apply to every plan)
The §13 rules don't relax for new features. The recurring traps:
- **24h-window (§13.2)** bites the flow builder (delay nodes), WhatsApp Flows (initiating outside
  window), cart recovery, and the AI agent — all free-text automation must gate on the window or fall
  back to an approved template.
- **Only APPROVED templates in campaigns / outside the window (§13.3)** — cart-recovery and re-engagement.
- **Never message unsubscribed/blocked contacts (§13.1)**; **everything stays workspace-isolated (§13.8)**.

## Per-plan status
Each doc lists its own phasing (Phase A/B/C) and open questions to resolve before implementation.

## Implementation depth
Every feature plan ends with an **"Implementation detail (deep-dive)"** section taking it to
implementation-ready depth, consistent with the existing backend:
- **Endpoint contracts** — concrete request/response DTOs + status codes for each endpoint.
- **Runtime sequence** — an ASCII sequence/swimlane of the runtime flow (and the §13 guard points).
- **Concrete class signatures** — entities (native-enum/jsonb mappings), repositories, services,
  controllers, integration clients — using the real package layout (`api.*` / `domain.*` /
  `integration.*` / `infrastructure.*`) and patterns (workspace-scoped `findByIdAndWorkspaceId`,
  atomic `@Modifying` counters, raw-body HMAC webhooks, `@Async` pools, the `AccessTokenConverter`).
- **Touch-points + tests** — the exact existing methods each feature hooks into, and a Testcontainers
  test outline.

These were matched against the codebase on 2026-06-20; they stop short of actual code (per "plan, not
implement"). External-API specifics (Meta Flows `nfm_reply`, Shopify webhook list/fields, CTWA
`referral`, Claude model IDs) are flagged to confirm against live docs at build time.
