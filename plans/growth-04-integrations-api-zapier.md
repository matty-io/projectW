# Plan: Public API + Outbound Webhooks + Zapier/Make

> Status: **planned, not built.** Post-MVP growth feature (relates to ARCHITECTURE §12 ecosystem).
> Cheap reach: unlocks the long-tail of integrations without building each connector. Design 2026-06-20.

**Why:** leaders ship native Shopify/HubSpot/Zoho connectors **plus** Zapier/Make + webhooks + a public
REST API. "Does it connect to my stack?" is now a qualifying question. A public API + outbound webhooks
+ a Zapier/Make app covers thousands of tools for a fraction of the effort of native connectors.

## Three parts
1. **Public REST API** — the existing `/api/**` surface, authenticated by **API keys** instead of JWT.
   Scoped to a workspace; rate-limited per key.
2. **Outbound webhooks** — workspace owners subscribe to events (`message.received`,
   `message.status`, `contact.created`, `campaign.completed`, `flow.response`); we POST signed
   payloads to their URL with retries.
3. **Zapier/Make app** — thin wrapper over the above (triggers = our webhooks; actions = send message /
   add contact / start campaign).

## Schema — new migration
```sql
CREATE TABLE api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  name VARCHAR(100) NOT NULL,
  key_hash TEXT NOT NULL,                 -- store hash only; show raw once on creation
  key_prefix VARCHAR(12) NOT NULL,        -- for display/identification
  last_used_at TIMESTAMPTZ,
  created_by UUID NOT NULL REFERENCES users(id),
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX api_keys_prefix_idx ON api_keys(key_prefix);

CREATE TABLE webhook_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  target_url TEXT NOT NULL,
  secret TEXT NOT NULL,                   -- for HMAC signing of deliveries
  events TEXT[] NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
  event_type VARCHAR(50) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending|delivered|failed
  attempts INT NOT NULL DEFAULT 0,
  last_attempt_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX webhook_deliveries_retry_idx ON webhook_deliveries(status, created_at);
```

## Non-obvious decisions
1. **API keys are a second auth path on the same controllers** — add an `ApiKeyAuthenticationFilter`
   that resolves a key → a synthetic workspace-scoped `Authentication` (same `workspaceId` the JWT
   carries), so every existing service stays workspace-isolated with no changes. Store **hash only**.
2. **Outbound delivery reuses the SQS/async pattern** — enqueue a delivery, a consumer/scheduler POSTs
   with **exponential-backoff retries**, signs with HMAC (`X-WaTools-Signature`), records attempts.
   Never deliver inline on the request thread.
3. **Rate-limit per key** (token bucket) to protect the API; surface limits in headers.
4. **Webhook events are emitted from existing domain events** — hook emission into `MessageService`,
   `CampaignService`, `MetaWebhookProcessor`, etc. (one small publish call each).
5. **Zapier/Make app is built/hosted on their side** — our job is stable triggers (webhooks) +
   idempotent actions; version the API (`/api/v1/...`) before publishing.

## Build order (per §11)
1. Migration (`api_keys`, `webhook_subscriptions`, `webhook_deliveries`).
2. Entities + repositories.
3. `ApiKeyService` (issue/revoke/verify) + `WebhookService` (subscribe + emit) + delivery worker.
4. `ApiKeyAuthenticationFilter` wired into `SecurityConfig` alongside the JWT filter.
5. `ApiKeyController` + `WebhookSubscriptionController`; publish OpenAPI docs.
6. Frontend: settings/developers → API keys (create/reveal-once/revoke), webhook endpoints, event log.
7. Build + submit the Zapier/Make app.

## Phasing
- **Phase A** — API keys + a versioned public subset of endpoints + OpenAPI docs.
- **Phase B** — outbound webhooks with signed retried delivery + event log.
- **Phase C** — Zapier + Make apps.

## Open questions
1. Expose the **full** API or a curated v1 subset first (recommended: curated subset).
2. Rate-limit strategy (per-key token bucket recommended) and default limits.
3. Zapier first, Make second, or both together.

---

# Implementation detail (deep-dive)

> Implementation-ready spec. The central trick: an `ApiKeyAuthenticationFilter` that mints the **same
> `JwtAuthToken`** the JWT path produces, so every existing workspace-scoped service works unchanged
> under API-key auth. New code under `domain/integration`, `api/integration`, plus the filter in
> `config/security`. Outbound delivery reuses the SQS/async + scheduler pattern. Matched 2026-06-20.
> **Not built.**

## A. API-key auth — the key design

Key format issued to the user **once**: `wat_<keyPrefix>_<secret>` (e.g. `wat_a1b2c3d4_9f…32chars`).
We store only `sha256(secret)` + the `keyPrefix` (indexed). On each request:
```
ApiKeyAuthenticationFilter (OncePerRequestFilter, ordered BEFORE JwtAuthenticationFilter):
  header "Authorization: Bearer wat_<prefix>_<secret>"  OR  "X-Api-Key: wat_<prefix>_<secret>"
  → parse prefix → apiKeyRepo.findByKeyPrefixAndRevokedAtIsNull(prefix)
  → constant-time compare sha256(secret) to stored key_hash
  → build JwtPrincipal(userId=createdBy, workspaceId=key.workspaceId, organizationId=…, role="admin")
  → SecurityContext.setAuthentication(new JwtAuthToken(principal))
  → async-update last_used_at
  (no/invalid key → leave context empty; JWT filter runs next; else 401 as today)
```
Because the produced `Authentication` is identical in shape to the JWT one, `SecurityUtils
.getCurrentWorkspaceId(auth)` and every service stay isolated (§13.8) with **zero service changes**.
Public API routes live under a versioned prefix `/api/v1/**` (the curated subset).

## B. Endpoint contracts (management — JWT-scoped, `admin`)

```
GET    /api/api-keys                     → List<ApiKeyResponse>      (prefix + name + lastUsedAt; never the secret)
POST   /api/api-keys                     → ApiKeyCreatedResponse     (the ONLY time rawKey is returned)
DELETE /api/api-keys/{id}                → 204                       (sets revoked_at)

GET    /api/webhook-subscriptions        → List<WebhookSubscriptionResponse>
POST   /api/webhook-subscriptions        → 201 (targetUrl + events[]; returns signing secret once)
DELETE /api/webhook-subscriptions/{id}   → 204
GET    /api/webhook-subscriptions/{id}/deliveries → Page<WebhookDeliveryResponse>   (event log)
```
```jsonc
// ApiKeyCreatedResponse — rawKey shown once, never recoverable
{ "id":"…", "name":"Zapier prod", "keyPrefix":"a1b2c3d4", "rawKey":"wat_a1b2c3d4_9f…", "createdAt":"…" }
```
Public surface (key-auth, `/api/v1`): a curated subset — e.g. `POST /api/v1/messages` (send),
`POST /api/v1/contacts`, `GET /api/v1/contacts`, `POST /api/v1/campaigns/{id}/send` — each delegating
to the same services the JWT controllers use.

## C. Outbound delivery sequence

```
Domain event (e.g. inbound saved)        WebhookService            delivery worker (SQS or @Scheduled)      subscriber URL
  MessageService/CampaignService/         │                          │                                        │
   MetaWebhookProcessor emits ──────────▶│ for each active subscription matching event:                       │
                                          │   INSERT webhook_deliveries(status=pending, payload)               │
                                          │   enqueue delivery id ──▶│ POST payload                            │
                                          │                          │  headers: X-WaTools-Signature = HMAC(secret, body)
                                          │                          │  ─────────────────────────────────────▶│
                                          │                          │  2xx → status=delivered                 │
                                          │                          │  else → attempts++, exp-backoff retry   │
                                          │                          │   (schedule next; give up after N)      │
```
Emission is **one small publish call** per domain event (fire-and-forget into `WebhookService.emit`);
delivery never runs on the request thread.

## D. Concrete class signatures

### `config/security/ApiKeyAuthenticationFilter`
```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
  // injected: ApiKeyService
  @Override protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
    if (SecurityContextHolder.getContext().getAuthentication() == null) {
      apiKeyService.resolve(extractKey(req))                       // Optional<JwtPrincipal>
          .ifPresent(p -> SecurityContextHolder.getContext().setAuthentication(new JwtAuthToken(p)));
    }
    chain.doFilter(req, res);
  }
}
// SecurityConfig: .addFilterBefore(apiKeyAuthenticationFilter, JwtAuthenticationFilter.class)
//   and permit /api/v1/** to be authenticated (still anyRequest().authenticated())
```

### Entities + repositories (`domain/integration/`)
```java
@Entity @Table(name="api_keys") @Getter @Setter @NoArgsConstructor
public class ApiKey {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(nullable=false, length=100) private String name;
  @Column(name="key_hash", nullable=false, columnDefinition="text") private String keyHash;   // sha256(secret)
  @Column(name="key_prefix", nullable=false, length=12) private String keyPrefix;
  @Column(name="last_used_at") private Instant lastUsedAt;
  @Column(name="created_by", nullable=false) private UUID createdBy;
  @Column(name="revoked_at") private Instant revokedAt;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
@Entity @Table(name="webhook_subscriptions") @Getter @Setter @NoArgsConstructor
public class WebhookSubscription {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="target_url", nullable=false, columnDefinition="text") private String targetUrl;
  @Column(nullable=false, columnDefinition="text") private String secret;
  @JdbcTypeCode(SqlTypes.ARRAY) @Column(columnDefinition="text[]") private String[] events;
  @Column(name="is_active", nullable=false) private boolean active = true;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
// + WebhookDelivery entity (status as plain VARCHAR or a delivery_status enum)
public interface ApiKeyRepository extends JpaRepository<ApiKey, UUID> {
  Optional<ApiKey> findByKeyPrefixAndRevokedAtIsNull(String keyPrefix);
  List<ApiKey> findByWorkspaceIdOrderByCreatedAtDesc(UUID ws);
}
public interface WebhookSubscriptionRepository extends JpaRepository<WebhookSubscription, UUID> {
  List<WebhookSubscription> findByWorkspaceIdAndActiveTrue(UUID ws);
}
public interface WebhookDeliveryRepository extends JpaRepository<WebhookDelivery, UUID> {
  List<WebhookDelivery> findByStatusOrderByCreatedAtAsc(String status, Pageable p);   // retry worker
}
```

### Services
```java
@Service public class ApiKeyService {
  ApiKeyCreatedResponse issue(UUID ws, UUID userId, String name);   // generate, hash, store; return rawKey ONCE
  Optional<JwtPrincipal> resolve(String rawKey);                    // prefix lookup + constant-time hash compare
  void revoke(UUID id, UUID ws);
  List<ApiKeyResponse> list(UUID ws);
}
@Service public class WebhookService {
  WebhookSubscriptionResponse subscribe(UUID ws, SubscribeWebhookRequest req);
  void unsubscribe(UUID id, UUID ws);
  void emit(UUID ws, String eventType, Object payload);            // fan-out to matching subscriptions → deliveries
}
@Component public class WebhookDeliveryWorker {                     // SQS consumer or @Scheduled sweep
  void deliver(UUID deliveryId);                                    // POST + HMAC sign + record attempt
  @Scheduled(fixedDelay=30_000) void retryFailed();                 // exponential backoff, cap attempts
}
```

### Controllers + DTOs
```java
@RestController @RequestMapping("/api/api-keys")
public class ApiKeyController { /* list / create(201) / revoke(204) — JWT, admin */ }
@RestController @RequestMapping("/api/webhook-subscriptions")
public class WebhookSubscriptionController { /* list / create(201) / delete(204) / deliveries */ }

public record ApiKeyResponse(UUID id, String name, String keyPrefix, Instant lastUsedAt, Instant createdAt) {}
public record ApiKeyCreatedResponse(UUID id, String name, String keyPrefix, String rawKey, Instant createdAt) {}
public record SubscribeWebhookRequest(@NotBlank @URL String targetUrl, @NotEmpty List<String> events) {}
public record WebhookSubscriptionResponse(UUID id, String targetUrl, List<String> events, boolean active, Instant createdAt) {}
```

## E. Event emission touch-points (one line each)
- `MessageService.saveInbound` → `webhookService.emit(ws, "message.received", …)`
- `MessageService.applyStatus` / `CampaignService.applyDeliveryStatus` → `"message.status"`
- `ContactService.create`/`findOrCreateByPhone` → `"contact.created"`
- `CampaignService.maybeComplete` (on → `sent`) → `"campaign.completed"`
- `FlowResponseService.handle` (growth-01) → `"flow.response"`

## F. Tests
- `ApiKeyAuthenticationFilter`: valid key → request runs scoped to the key's workspace; revoked key →
  401; key for workspace A cannot read workspace B's contacts (§13.8 holds with no service change).
- `ApiKeyService.resolve`: constant-time compare; wrong secret with right prefix → empty.
- Delivery worker: 5xx target → retried with backoff, `attempts` increments, signature verifies on the
  receiver side; gives up after the cap.
