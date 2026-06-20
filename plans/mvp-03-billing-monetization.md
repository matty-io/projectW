# Plan: Billing & Monetization

> Status: **planned, not built.** Source of truth for schema/endpoints remains `ARCHITECTURE.md`;
> this doc captures the design agreed on 2026-06-20. ARCHITECTURE §1 pricing must be updated when
> this is built (drop "unlimited", document wallet + markup).

**Model:** org-level platform **subscription** (Razorpay Subscriptions) + workspace-level **prepaid
wallet** that debits each sent message at `Meta rate × (1 + markup)`.

## Why this model
Since 1 July 2025 Meta bills **per message**, not per 24h conversation. ARCHITECTURE §1's
"Pro ₹2,499/mo = unlimited messages" is economically broken — marketing messages cost the provider
~₹0.89–1.09 each. Competitors (AiSensy/Wati) use subscription + prepaid wallet; the markup
(AiSensy ~12% on marketing) is itself a revenue stream.

## Core design decisions
1. **Two money concepts, two Razorpay products.**
   - *Subscription* = recurring platform fee on `organizations` (already has `razorpay_customer_id` /
     `razorpay_subscription_id`). Gates features + agent seats. Razorpay **Subscriptions** API.
   - *Wallet* = prepaid balance for message costs. Top-ups are one-time Razorpay Orders / Checkout.
2. **Wallet is per-workspace, subscription is per-org.** Sending flows through a workspace's WABA, so
   cost attribution belongs at the workspace; fits agency/white-label resale and workspace isolation
   (§13.8). One-brand customers just have one wallet.
3. **Money stored in paise as `BIGINT` — never floats.** All debits/credits are integer arithmetic.
4. **Wallet gates the campaign send path** (woven into the SQS broadcast engine, §5):
   - *Pre-flight* at `CampaignService.initiateSend()`:
     `estimatedCost = recipientCount × rate(template.category) × (1+markup)`. If `balance < estimatedCost`
     → reject (HTTP 402), don't enqueue. Sits after the unsubscribed-filter + approved-template checks.
   - *Per-message debit* in `BroadcastConsumer` on successful Meta send, via atomic conditional decrement
     (`UPDATE wallet SET balance = balance - :amt WHERE id=:id AND balance >= :amt`); 0 rows affected →
     mark `campaign_contact` failed with `error_code=INSUFFICIENT_BALANCE`.
   - *Refund* when a message resolves to `failed` (delivery webhook) → credit back. Hooks into existing
     `handleStatusUpdate`.
   - Customer-initiated/service messages and free utility-in-window are ₹0 — no debit.

## Schema — new migration `V3__billing.sql`
```
wallets               (id, workspace_id UNIQUE, balance_paise BIGINT,
                       low_balance_threshold_paise, currency, created_at)
wallet_transactions   (id, wallet_id, workspace_id, type[credit|debit],
                       amount_paise, balance_after_paise,
                       reference_type[topup|campaign_message|reply|refund|adjustment],
                       reference_id, meta_category, created_at)   -- append-only ledger
message_rates         (id, category, country_code, meta_cost_paise,
                       markup_bps, effective_from)                -- rate card; bps = basis points
razorpay_events       (event_id UNIQUE, type, processed_at)       -- webhook idempotency/dedup
```
`organizations` reuses existing Razorpay columns (+ maybe a `subscription_status` enum). Balance is a
cached column **plus** the ledger for audit — same atomic-counter pattern as `campaigns.sent_count`.

## New endpoints (Billing section, not yet in §7)
```
GET  /api/billing/subscription            — current plan, status, renewal
POST /api/billing/subscription            — create/checkout (returns Razorpay subscription)
POST /api/billing/subscription/cancel
GET  /api/billing/wallet                  — balance + rate card + low-balance threshold
POST /api/billing/wallet/topup            — create Razorpay order for an amount
GET  /api/billing/wallet/transactions     — paginated ledger
GET  /api/billing/usage                   — current-period message count + spend
POST /webhook/razorpay                    — public, signature-verified (mirrors Meta webhook rules)
```
Razorpay webhook reuses the Meta playbook: **verify signature → 200 fast → process async → dedup on
event id**. `payment.captured` → credit wallet; `subscription.charged/activated/halted/cancelled` →
update org plan + `plan_expires_at`.

## Plan-limit enforcement
- **Agent seats** — count workspace members vs plan cap on invite/add.
- **Workspaces** — Agency-only multi-workspace; block creation past cap.
- **Sending** — requires positive wallet balance (pre-flight above).

## Build order (per §11) + phasing
Migration → entities (`Wallet`, `WalletTransaction`, `MessageRate`, `RazorpayEvent`) → repositories →
`WalletService` + `BillingService` + `integration/razorpay/RazorpayClient` → `BillingController` +
`RazorpayWebhookController` → Next.js `settings/billing` page.

- **Phase A — Wallet core**: schema, ledger, top-up, debit-in-send-path, pre-flight, refund-on-failure.
- **Phase B — Subscription**: Razorpay Subscriptions, plan gates, seat limits.
- **Phase C — Usage view + low-balance notifications** (banner + SES email).

## Frontend
`app/(dashboard)/settings/billing/page.tsx`: plan upgrade/downgrade, wallet balance, top-up (Razorpay
Checkout), ledger table, current-period usage, global low-balance banner.

## Open questions to resolve before implementation
1. Rate-card scope: India-only single rate per category (recommended) vs multi-country day one.
2. Markup: global flat % (recommended, ~12% mktg / 8% utility) vs per-plan.
3. Top-up UX: Razorpay Checkout (recommended, FE exists) vs Payment Links.
4. Phasing: A→B→C vs subscriptions-first.

---

# Implementation detail (deep-dive)

> Implementation-ready spec. New code under `domain/billing/*`, `api/billing/*`,
> `api/webhook/RazorpayWebhookController`, `integration/razorpay/RazorpayClient`. The Razorpay
> webhook **deliberately mirrors the existing Meta webhook** (`WhatsAppWebhookController` +
> `WebhookSignatureVerifier`): raw `byte[]` body, constant-time HMAC, 200-fast, `@Async`, dedup.
> Money is **paise as `BIGINT`/`long`**, integer arithmetic only. Matched to the repo on
> 2026-06-20. **Not built.**

## A. Migration `V3__billing.sql` (full DDL)
```sql
CREATE TYPE wallet_txn_type AS ENUM ('credit','debit');
CREATE TYPE wallet_txn_reference AS ENUM ('topup','campaign_message','reply','refund','adjustment');
CREATE TYPE subscription_status AS ENUM ('none','active','past_due','halted','cancelled');

CREATE TABLE wallets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID UNIQUE NOT NULL REFERENCES workspaces(id),
  balance_paise BIGINT NOT NULL DEFAULT 0,
  low_balance_threshold_paise BIGINT NOT NULL DEFAULT 0,
  currency VARCHAR(8) NOT NULL DEFAULT 'INR',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT wallets_balance_non_negative CHECK (balance_paise >= 0)
);

CREATE TABLE wallet_transactions (            -- append-only ledger; never UPDATE/DELETE
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_id UUID NOT NULL REFERENCES wallets(id),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  type wallet_txn_type NOT NULL,
  amount_paise BIGINT NOT NULL,               -- always positive; sign implied by type
  balance_after_paise BIGINT NOT NULL,
  reference_type wallet_txn_reference NOT NULL,
  reference_id VARCHAR(100),                  -- campaign_contact id, razorpay payment id, …
  meta_category template_category,            -- reuse existing enum (MARKETING/UTILITY/AUTHENTICATION)
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX wallet_txn_wallet_idx ON wallet_transactions(wallet_id, created_at DESC);
CREATE UNIQUE INDEX wallet_txn_debit_ref_idx                    -- idempotent per-message debit guard
  ON wallet_transactions(reference_type, reference_id) WHERE reference_type = 'campaign_message';

CREATE TABLE message_rates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  category template_category NOT NULL,
  country_code VARCHAR(4) NOT NULL DEFAULT 'IN',
  meta_cost_paise BIGINT NOT NULL,
  markup_bps INT NOT NULL DEFAULT 1200,       -- basis points: 1200 = 12%
  effective_from TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX message_rates_lookup_idx ON message_rates(category, country_code, effective_from DESC);

CREATE TABLE razorpay_events (                -- webhook idempotency (mirrors rule-5 dedup)
  event_id VARCHAR(100) PRIMARY KEY,
  type VARCHAR(60) NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE organizations ADD COLUMN subscription_status subscription_status NOT NULL DEFAULT 'none';
```
> The `wallet_txn_debit_ref_idx` unique index is the real safety net: if SQS redelivers a broadcast
> message, the second debit insert violates the index and is swallowed — **a contact is billed at most
> once**, independent of the in-memory check. Charge cost = `round(meta_cost_paise × (10000+markup_bps)/10000)`.

## B. Two runtime sequences

### B1. Send path — pre-flight check + per-message debit
```
admin                 CampaignService            WalletService          BroadcastConsumer/CampaignService.deliver
 │ POST /campaigns/{id}/send │                        │                            │
 │──────────────────────────▶│ initiateSend():        │                            │
 │                           │  rule1 filter + rule3 template (existing)           │
 │                           │  estimate = Σ rate(cat)×(1+markup) over recipients  │
 │                           │  walletService.assertSufficient(ws, estimate) ──────▶│ balance < est?
 │                           │ ◀───── InsufficientBalanceException (HTTP 402) ──────│  → reject, nothing enqueued
 │                           │  else enqueue SQS per recipient (existing)          │
 │ ◀──── 200 sending ────────│                        │                            │
 │                           │                        │   (per message, async) ───▶│ deliver(msg):
 │                           │                        │                            │  Meta send OK →
 │                           │   walletService.debitForMessage(ws, cc.id, cat) ◀───│
 │                           │     UPDATE wallets SET balance=balance-:amt          │
 │                           │       WHERE workspace_id=:ws AND balance>=:amt       │
 │                           │     0 rows? → cc.status=failed, INSUFFICIENT_BALANCE │
 │                           │     1 row?  → INSERT ledger(debit, balance_after)    │
 │                           │              (unique ref index ⇒ idempotent)         │
```
The debit happens **inside** `CampaignService.deliver` right after a successful `metaMessageClient.sendTemplate`
(before/with the `incrementSent` counter), so it shares the consumer's transaction and idempotency.

### B2. Razorpay webhook — mirror of the Meta webhook
```
Razorpay            RazorpayWebhookController        RazorpayWebhookProcessor (@Async)     BillingService
   │ POST /webhook/razorpay (raw body + X-Razorpay-Signature)                              │
   │───────────────────────▶│ verifier.isValid(body, sig)  (HMAC-SHA256, app webhook secret)
   │                        │  invalid → 403                                               │
   │                        │ processor.processAsync(body) ──▶│                            │
   │ ◀──── 200 (fast) ──────│                                 │ parse event_id, type       │
   │                        │                                 │ INSERT razorpay_events     │
   │                        │                                 │  (PK conflict ⇒ dup, stop) │
   │                        │                                 │ switch(type):              │
   │                        │                                 │  payment.captured ─────────▶│ wallet credit + ledger
   │                        │                                 │  subscription.charged/     │
   │                        │                                 │   activated/halted/        │
   │                        │                                 │   cancelled ───────────────▶│ org.plan + plan_expires_at
   │                        │                                 │                             │   + subscription_status
```

## C. Concrete class signatures

### Entities (`domain/billing/`)
```java
@Entity @Table(name="wallets") @Getter @Setter @NoArgsConstructor
public class Wallet {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="balance_paise", nullable=false) private long balancePaise;
  @Column(name="low_balance_threshold_paise", nullable=false) private long lowBalanceThresholdPaise;
  @Column(nullable=false, length=8) private String currency = "INR";
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}

@Entity @Table(name="wallet_transactions") @Getter @Setter @NoArgsConstructor
public class WalletTransaction {                       // append-only — no setters used after insert
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="wallet_id", nullable=false) private UUID walletId;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(nullable=false, columnDefinition="wallet_txn_type")
  private WalletTxnType type;
  @Column(name="amount_paise", nullable=false) private long amountPaise;
  @Column(name="balance_after_paise", nullable=false) private long balanceAfterPaise;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(name="reference_type", nullable=false, columnDefinition="wallet_txn_reference")
  private WalletTxnReference referenceType;
  @Column(name="reference_id", length=100) private String referenceId;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(name="meta_category", columnDefinition="template_category")
  private TemplateCategory metaCategory;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
// + MessageRate entity; RazorpayEvent entity (String PK event_id)
public enum WalletTxnType { credit, debit }
public enum WalletTxnReference { topup, campaign_message, reply, refund, adjustment }
public enum SubscriptionStatus { none, active, past_due, halted, cancelled }
```

### Repositories — the atomic conditional debit is the crux
```java
public interface WalletRepository extends JpaRepository<Wallet, UUID> {
  Optional<Wallet> findByWorkspaceId(UUID workspaceId);

  /** Atomic guarded debit — never goes negative. Returns rows-affected (1 = success, 0 = insufficient). */
  @Modifying
  @Query("UPDATE Wallet w SET w.balancePaise = w.balancePaise - :amt "
       + "WHERE w.workspaceId = :ws AND w.balancePaise >= :amt")
  int tryDebit(@Param("ws") UUID workspaceId, @Param("amt") long amountPaise);

  @Modifying
  @Query("UPDATE Wallet w SET w.balancePaise = w.balancePaise + :amt WHERE w.workspaceId = :ws")
  void credit(@Param("ws") UUID workspaceId, @Param("amt") long amountPaise);
}
public interface WalletTransactionRepository extends JpaRepository<WalletTransaction, UUID> {
  Page<WalletTransaction> findByWalletIdOrderByCreatedAtDesc(UUID walletId, Pageable pageable);
  boolean existsByReferenceTypeAndReferenceId(WalletTxnReference type, String referenceId);   // debit idempotency
}
public interface MessageRateRepository extends JpaRepository<MessageRate, UUID> {
  Optional<MessageRate> findFirstByCategoryAndCountryCodeOrderByEffectiveFromDesc(TemplateCategory c, String country);
}
public interface RazorpayEventRepository extends JpaRepository<RazorpayEvent, String> {}   // PK = event_id
```

### Services
```java
@Service
public class WalletService {
  long charge(TemplateCategory category, String country);                 // meta_cost × (1+markup) in paise
  @Transactional(readOnly=true) void assertSufficient(UUID ws, long estimatedPaise);   // throws InsufficientBalanceException (402)
  /** Called from CampaignService.deliver after a successful send. Idempotent on campaignContactId. */
  @Transactional DebitResult debitForMessage(UUID ws, UUID campaignContactId, TemplateCategory category);
  @Transactional void credit(UUID ws, long paise, WalletTxnReference ref, String refId);   // topup / refund
  @Transactional(readOnly=true) WalletResponse wallet(UUID ws);
  @Transactional(readOnly=true) Page<WalletTransactionResponse> ledger(UUID ws, Pageable p);
  enum DebitResult { DEBITED, INSUFFICIENT }
}

@Service
public class BillingService {                 // Razorpay subscription + webhook effects
  SubscriptionResponse subscription(UUID orgId);
  CheckoutResponse createSubscription(UUID orgId, String planCode);
  void cancelSubscription(UUID orgId);
  TopupOrderResponse createTopupOrder(UUID ws, long amountPaise);          // Razorpay order
  void onPaymentCaptured(String paymentId, UUID ws, long amountPaise);     // webhook → wallet credit
  void onSubscriptionEvent(String type, String razorpaySubId, ...);        // webhook → org plan/status
}
```

### New exception + controllers
```java
// common/exception/InsufficientBalanceException → mapped to 402 in GlobalExceptionHandler
@ResponseStatus(HttpStatus.PAYMENT_REQUIRED)
public class InsufficientBalanceException extends RuntimeException { … }

@RestController @RequestMapping("/api/billing")
public class BillingController {               // all JWT-scoped; admin-only (role check in service)
  @GetMapping("/subscription") SubscriptionResponse subscription(Authentication auth);
  @PostMapping("/subscription") CheckoutResponse subscribe(Authentication auth, @Valid @RequestBody SubscribeRequest r);
  @PostMapping("/subscription/cancel") @ResponseStatus(NO_CONTENT) void cancel(Authentication auth);
  @GetMapping("/wallet") WalletResponse wallet(Authentication auth);
  @PostMapping("/wallet/topup") TopupOrderResponse topup(Authentication auth, @Valid @RequestBody TopupRequest r);
  @GetMapping("/wallet/transactions") Page<WalletTransactionResponse> ledger(Authentication auth, Pageable p);
  @GetMapping("/usage") UsageResponse usage(Authentication auth);
}

// api/webhook/RazorpayWebhookController — structural twin of WhatsAppWebhookController
@RestController @RequestMapping("/webhook/razorpay")
public class RazorpayWebhookController {
  @PostMapping ResponseEntity<Void> receive(@RequestBody(required=false) byte[] body,
      @RequestHeader(name="X-Razorpay-Signature", required=false) String signature) {
    if (!razorpaySignatureVerifier.isValid(body, signature)) return ResponseEntity.status(FORBIDDEN).build();
    razorpayWebhookProcessor.processAsync(body);   // @Async("webhookExecutor")
    return ResponseEntity.ok().build();
  }
}
```
> Add `/webhook/razorpay` to the `permitAll()` list in `SecurityConfig` (it already permits
> `/webhook/**`, so this is covered — verify). `RazorpaySignatureVerifier` is a copy of
> `WebhookSignatureVerifier` keyed by `razorpay.webhook-secret` (config already present in §9).

## D. Touch-points in existing code (small, surgical)
- `CampaignService.initiateSend` → insert `walletService.assertSufficient(ws, estimate)` **after** the
  rule-1/rule-3 checks, **before** enqueue.
- `CampaignService.deliver` → after `metaMessageClient.sendTemplate` success and before/with
  `incrementSent`, call `walletService.debitForMessage(ws, cc.getId(), template.getCategory())`; on
  `INSUFFICIENT` set `cc` failed with `error_code=INSUFFICIENT_BALANCE` (reuse existing `fail(...)`).
- `CampaignService.applyDeliveryStatus` `failed` branch → `walletService.credit(ws, charge, refund, cc.id)`
  (only if that message was actually debited — check the ledger).
- `AuthService` signup → create a `Wallet` row for the default workspace (balance 0).

## E. Tests
- `WalletRepository.tryDebit` against Testcontainers: concurrent debits never drive balance negative;
  `tryDebit` returns 0 at the boundary; `CHECK` constraint backstops.
- Debit idempotency: calling `debitForMessage` twice with the same `campaignContactId` charges once
  (unique ledger ref index).
- Razorpay webhook: invalid signature → 403; duplicate `event_id` → second call no-ops; `payment.captured`
  credits exactly once.
- Pre-flight: `assertSufficient` throws 402 and no SQS messages are produced.
