# Plan: Shopify Integration + Abandoned-Cart Recovery

> Status: **planned, not built.** Post-MVP growth feature (ARCHITECTURE §12, "Shopify / WooCommerce
> integration"). The D2C revenue hook. Design 2026-06-20.

**Why:** your headline target is D2C brands, and the killer use case there is **abandoned-cart
recovery** — vendors pitch recovering ~30% of lost sales (vendor claim). Plus order-confirmation and
shipping-update automations. It's a concrete ROI story that converts D2C founders, and it reuses the
template + scheduler + send infra already built.

## How it works
1. **Connect a store** — Shopify OAuth (custom/public app); store the shop domain + offline access
   token (encrypted, like the Meta token via an `AttributeConverter`).
2. **Subscribe to Shopify webhooks** — `checkouts/create`, `checkouts/update`, `orders/create`,
   `orders/paid`, plus `customers/data_request|redact` + `shop/redact` (GDPR, mandatory for Shopify).
3. **Ingest events** at a new `POST /webhook/shopify` (HMAC-verified with the Shopify secret — mirror
   the Meta signature pattern; return 200 fast, process async).
4. **Abandoned cart**: a `checkouts/create` with no matching `orders/create` after N minutes →
   `@Scheduled` sweep enqueues a recovery **template** message (checkout link + name + optional
   discount) to the contact. Cancel if the order completes first.
5. **Order/shipping updates**: `orders/create|paid` → send a confirmation template.

## Schema — new migration
```sql
CREATE TABLE shopify_connections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID UNIQUE NOT NULL REFERENCES workspaces(id),   -- 1 store per workspace (MVP)
  shop_domain VARCHAR(255) NOT NULL,
  access_token TEXT NOT NULL,                                    -- encrypted at rest
  recovery_template_id UUID REFERENCES templates(id),
  recovery_delay_minutes INT NOT NULL DEFAULT 60,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TYPE cart_status AS ENUM ('open','recovered','message_sent','expired');

CREATE TABLE abandoned_carts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  connection_id UUID NOT NULL REFERENCES shopify_connections(id),
  contact_id UUID REFERENCES contacts(id),
  shopify_checkout_id VARCHAR(100) NOT NULL,
  checkout_url TEXT,
  total_amount NUMERIC(12,2),
  currency VARCHAR(8),
  status cart_status NOT NULL DEFAULT 'open',
  abandoned_at TIMESTAMPTZ NOT NULL,
  recovery_sent_at TIMESTAMPTZ,
  recovered_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX abandoned_carts_checkout_idx ON abandoned_carts(connection_id, shopify_checkout_id);
CREATE INDEX abandoned_carts_sweep_idx ON abandoned_carts(status, abandoned_at);
```

## Non-obvious decisions
1. **Recovery messages must use an approved MARKETING template** (§13.3) — the customer is outside any
   24h window. The store owner picks `recovery_template_id`; variables (name, cart URL, discount) map
   from the checkout. This ties the feature to the templates module, not free text.
2. **Honour unsubscribe** (§13.1) — never send recovery to `unsubscribed`/`blocked` contacts. Match the
   Shopify customer to a workspace contact by phone (create as `source=imported` if new, opted-in per
   store consent only).
3. **Verify Shopify webhook HMAC** and **dedup** on `shopify_checkout_id` / order id — Shopify retries.
   Same defensive posture as the Meta webhook (rules 5–7).
4. **Cancel-on-purchase race**: the sweep must re-check the cart isn't already `recovered` before
   sending (an `orders/create` can land between scheduling and send).
5. **Shopify GDPR webhooks are mandatory** for app approval — implement the three redaction endpoints
   from day one or the public app won't pass review.
6. **Token encrypted at rest**, reusing the existing `AccessTokenConverter` pattern.

## Build order (per §11)
1. Migration (`shopify_connections`, `abandoned_carts`, `cart_status`).
2. Entities + enum + encrypted-token converter reuse.
3. Repositories (workspace-scoped; due-cart sweep query).
4. `integration/shopify/ShopifyClient` (OAuth token exchange, register webhooks, fetch checkout) +
   `ShopifyWebhookVerifier`.
5. `ShopifyService` (connect/disconnect, ingest events, match contact) + `CartRecoveryScheduler`
   (sweep due carts → send recovery template via existing campaign/message send path).
6. `ShopifyWebhookController` (`/webhook/shopify`, public, HMAC-verified) + `ShopifyController`
   (connect flow, settings).
7. Frontend: settings/integrations → "Connect Shopify" (OAuth), pick recovery template + delay, and a
   small recovery-performance widget (carts / recovered / revenue).

## Phasing
- **Phase A** — connect store + abandoned-cart recovery (the ROI hook).
- **Phase B** — order confirmation + shipping-update automations.
- **Phase C** — catalog sync (browse/buy in chat) + WooCommerce via the same event-ingest shape.

## Open questions
1. Shopify **custom app** (faster, per-merchant) vs **public app** (App Store, needs review + GDPR) —
   recommend custom app for beta, public later.
2. One store per workspace (recommended) vs many.
3. Discount-code generation via Shopify API, or owner supplies a static code (recommended for MVP).

---

# Implementation detail (deep-dive)

> Implementation-ready spec. New code under `domain/shopify`, `api/shopify`,
> `api/webhook/ShopifyWebhookController`, `integration/shopify/ShopifyClient`. The webhook mirrors
> the Meta pattern (`WhatsAppWebhookController` + raw-body HMAC + 200-fast + `@Async`); the recovery
> sweep mirrors `CampaignScheduler`; the encrypted token **reuses `AccessTokenConverter`** verbatim.
> Recovery sends go through the existing approved-MARKETING-template path (§13.3). Matched 2026-06-20.
> **Not built.** Confirm Shopify's mandatory webhook list + payload fields against live Shopify docs.

## A. Endpoint contracts

### Connect flow (JWT-scoped, `admin`)
```
GET  /api/shopify/connect?shop=acme.myshopify.com   → { authorizeUrl }   // builds Shopify OAuth URL + state
GET  /api/shopify/callback?code&shop&state&hmac      → 302 to settings    // token exchange + webhook registration
GET  /api/shopify/connection                         → ShopifyConnectionResponse | 404
PUT  /api/shopify/connection                         → update recovery_template_id + delay_minutes
DELETE /api/shopify/connection                       → disconnect (uninstall webhooks, deactivate)
```
`PUT` validates `recovery_template_id` is an **APPROVED MARKETING** template in the workspace
(`ConflictException` otherwise — enforces §13.3 at config time, not just send time).

### Shopify webhooks (public, HMAC-verified — structural twin of the Meta webhook)
```
POST /webhook/shopify   — X-Shopify-Hmac-Sha256 + X-Shopify-Topic + X-Shopify-Shop-Domain headers
```
Topics handled: `checkouts/create`, `checkouts/update`, `orders/create`, `orders/paid`, and the three
mandatory GDPR topics `customers/data_request`, `customers/redact`, `shop/redact`.

## B. Runtime sequences

### B1. Abandoned cart → recovery (the ROI path)
```
Shopify          ShopifyWebhookController       ShopifyService(@Async)        CartRecoveryScheduler        send path
  │ POST /webhook/shopify (checkouts/create)                                        │                        │
  │────────────────────────▶│ verifier.isValid(rawBody, hmac)  (base64 HMAC-SHA256, Shopify secret)
  │                         │  invalid → 401                                        │                        │
  │ ◀──── 200 (fast) ───────│ processAsync(topic, shopDomain, body) ─▶│            │                        │
  │                         │            │ match connection by shop_domain          │                        │
  │                         │            │ dedup on (connection, checkout_id)        │                        │
  │                         │            │ match/create contact by phone (§13.1 — skip unsubscribed)
  │                         │            │ UPSERT abandoned_carts(status=open, abandoned_at=now)
  │                         │  (orders/create|paid for same checkout) → cart.status=recovered, recovered_at
  │                         │                                          │            │ @Scheduled(60s):
  │                         │                                          │            │ find status=open AND
  │                         │                                          │            │  abandoned_at <= now-delay
  │                         │                                          │            │ for each (own txn):
  │                         │                                          │            │   RE-CHECK status==open  ← cancel-on-purchase race
  │                         │                                          │            │   contact subscribed? (§13.1)
  │                         │                                          │            │   send recovery TEMPLATE ──▶ MetaMessageClient.sendTemplate
  │                         │                                          │            │   cart.status=message_sent, recovery_sent_at=now
```

### B2. GDPR redaction (mandatory for app review)
```
customers/redact / shop/redact / customers/data_request → ShopifyService deletes/exports the
  matching abandoned_carts + (for shop/redact) the shopify_connection; respond 200. Implement from day one.
```

## C. Concrete class signatures

### Entities (`domain/shopify/`)
```java
@Entity @Table(name="shopify_connections") @Getter @Setter @NoArgsConstructor
public class ShopifyConnection {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="shop_domain", nullable=false, length=255) private String shopDomain;
  @Convert(converter = AccessTokenConverter.class)          // reuse the Meta-token encryptor
  @Column(name="access_token", nullable=false, columnDefinition="text") private String accessToken;
  @Column(name="recovery_template_id") private UUID recoveryTemplateId;
  @Column(name="recovery_delay_minutes", nullable=false) private int recoveryDelayMinutes = 60;
  @Column(name="is_active", nullable=false) private boolean active = true;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
@Entity @Table(name="abandoned_carts") @Getter @Setter @NoArgsConstructor
public class AbandonedCart {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="connection_id", nullable=false) private UUID connectionId;
  @Column(name="contact_id") private UUID contactId;
  @Column(name="shopify_checkout_id", nullable=false, length=100) private String shopifyCheckoutId;
  @Column(name="checkout_url", columnDefinition="text") private String checkoutUrl;
  @Column(name="total_amount", precision=12, scale=2) private BigDecimal totalAmount;   // display only; money-to-send stays template vars
  @Column(length=8) private String currency;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(nullable=false, columnDefinition="cart_status")
  private CartStatus status = CartStatus.open;
  @Column(name="abandoned_at", nullable=false) private Instant abandonedAt;
  @Column(name="recovery_sent_at") private Instant recoverySentAt;
  @Column(name="recovered_at") private Instant recoveredAt;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
public enum CartStatus { open, recovered, message_sent, expired }
```

### Integration + repositories + service + scheduler
```java
public interface ShopifyClient {                          // integration/shopify, RestClient-based
  String exchangeToken(String shopDomain, String code);   // OAuth code → offline access token
  void registerWebhooks(String shopDomain, String token, String callbackUrl, List<String> topics);
  void uninstallWebhooks(String shopDomain, String token);
}
@Component public class ShopifyWebhookVerifier {          // copy of WebhookSignatureVerifier, base64 not hex
  boolean isValid(byte[] body, String hmacHeader);        // keyed by shopify.webhook-secret
}
public interface ShopifyConnectionRepository extends JpaRepository<ShopifyConnection, UUID> {
  Optional<ShopifyConnection> findByWorkspaceId(UUID ws);
  Optional<ShopifyConnection> findByShopDomain(String shopDomain);   // webhook lookup
}
public interface AbandonedCartRepository extends JpaRepository<AbandonedCart, UUID> {
  Optional<AbandonedCart> findByConnectionIdAndShopifyCheckoutId(UUID conn, String checkoutId);  // dedup/upsert
  List<AbandonedCart> findByStatusAndAbandonedAtLessThanEqual(CartStatus status, Instant cutoff); // sweep
}
@Service public class ShopifyService {
  ConnectUrlResponse beginConnect(UUID ws, String shop);
  void completeConnect(UUID ws, String shop, String code, String state);    // token + registerWebhooks
  @Async("webhookExecutor") void processAsync(String topic, String shopDomain, byte[] body);  // ingest
  ShopifyConnectionResponse connection(UUID ws);
  void updateSettings(UUID ws, UpdateConnectionRequest req);                 // validates APPROVED MARKETING template
  void disconnect(UUID ws);
}
@Component public class CartRecoveryScheduler {           // mirrors CampaignScheduler
  @Scheduled(fixedDelay=60_000) void sweepDueCarts();     // per cart, own txn; re-check status==open before send
}
```
> The recovery send re-uses the campaign/message send path: resolve the workspace WABA via
> `WaAccountService.requireAccount`, map cart fields (`{{name}}`, `{{checkout_url}}`, discount) into the
> template's body params exactly like `CampaignService.resolveBodyParams`, call
> `MetaMessageClient.sendTemplate`. No free text ever leaves this feature (§13.3).

### Controllers + DTOs
```java
@RestController @RequestMapping("/api/shopify")
public class ShopifyController { /* connect, callback(302), connection, update, disconnect */ }

@RestController @RequestMapping("/webhook/shopify")        // public; add to SecurityConfig permitAll (/webhook/** already covers)
public class ShopifyWebhookController {
  @PostMapping ResponseEntity<Void> receive(@RequestBody(required=false) byte[] body,
      @RequestHeader("X-Shopify-Hmac-Sha256") String hmac,
      @RequestHeader("X-Shopify-Topic") String topic,
      @RequestHeader("X-Shopify-Shop-Domain") String shop) {
    if (!verifier.isValid(body, hmac)) return ResponseEntity.status(UNAUTHORIZED).build();
    shopifyService.processAsync(topic, shop, body);
    return ResponseEntity.ok().build();
  }
}
public record ShopifyConnectionResponse(UUID id, String shopDomain, UUID recoveryTemplateId,
                                        int recoveryDelayMinutes, boolean active, Instant createdAt) {}
public record UpdateConnectionRequest(UUID recoveryTemplateId, @Min(5) Integer recoveryDelayMinutes) {}
```

## D. Tests
- `ShopifyWebhookVerifier`: known body+secret → valid; tampered body → 401 (base64 HMAC, not hex).
- Ingest: `checkouts/create` then `orders/paid` for the same checkout → cart ends `recovered`, no send.
- Sweep race: cart flips to `recovered` between selection and send → scheduler skips it (re-check).
- §13 guards: unsubscribed contact → no recovery; non-APPROVED template in settings → 409.
- GDPR: `shop/redact` removes the connection + carts.
