# Plan: Click-to-WhatsApp Ads (CTWA) Attribution

> Status: **planned, not built.** Post-MVP growth feature (ARCHITECTURE §12, "click-to-WhatsApp ads").
> Wins the performance-marketer D2C buyer. Design 2026-06-20.

**Why:** D2C brands run Meta ads that open WhatsApp. Without attribution, that funnel
(ad → WhatsApp → conversion) is a black hole. Capturing it lets customers measure cost-per-lead /
cost-per-purchase and feed conversion signals back to Meta to optimise delivery — a feature both Wati
and AiSensy sell. Moderate effort; reuses the inbound webhook + contact/conversation model.

## How it works
1. **Capture the referral.** When a user taps a Click-to-WhatsApp ad, Meta includes a `referral`
   object on the **first inbound message** (and a `ctwa_clid`) — `source_id` (ad id), `source_type`,
   `source_url`, headline/body, plus the `ctwa_clid`. `MetaWebhookProcessor` already parses inbound
   messages; extend it to persist these onto the conversation/contact.
2. **Attribute downstream events** — tag the contact/conversation with the ad + clid so later
   conversions (order, qualified lead, flow completion) can be rolled up by ad/campaign.
3. **Send conversions back to Meta (CAPI)** — POST conversion events with the `ctwa_clid` so Meta can
   optimise ad delivery and report cost-per-result.

## Schema — new migration (extend, don't duplicate)
```sql
ALTER TABLE conversations
  ADD COLUMN ctwa_clid TEXT,
  ADD COLUMN ad_source_id VARCHAR(100),
  ADD COLUMN ad_source_type VARCHAR(40),
  ADD COLUMN ad_referral JSONB;          -- full referral object for audit

CREATE TABLE ctwa_conversions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  conversation_id UUID NOT NULL REFERENCES conversations(id),
  ctwa_clid TEXT NOT NULL,
  event_name VARCHAR(50) NOT NULL,       -- Lead | Purchase | ...
  value NUMERIC(12,2),
  currency VARCHAR(8),
  sent_to_meta_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ctwa_conversions_ws_idx ON ctwa_conversions(workspace_id);
```
(Also store `ctwa_clid` / `ad_source_id` on the `contacts.attributes` jsonb for segmentation.)

## Non-obvious decisions
1. **The referral only arrives on the FIRST inbound** of the ad-initiated thread — capture it the
   moment a conversation is opened/updated from a `referral`-bearing message, or it's lost. Don't
   overwrite an existing clid on later messages.
2. **Conversion postback needs the Meta Conversions API for CTWA** (Dataset/Pixel + access token) —
   separate Graph endpoint from messaging. Sign and dedup events (Meta dedups on event id).
3. **Conversions are emitted from existing domain events** — order paid ([[growth-03-shopify-cart-recovery]]),
   flow completion ([[growth-01-whatsapp-flows]]), or a manual "mark qualified" in the inbox.
4. **Privacy** — `ctwa_clid` is ad-tracking data; keep it workspace-isolated (§13.8) and out of any
   shared/exported surface unless the owner opts in.

## Build order (per §11)
1. Migration (conversation columns + `ctwa_conversions`).
2. Extend `Conversation` entity; add `CtwaConversion` entity + repository.
3. Extend `MetaWebhookProcessor` to parse + persist the `referral` object on conversation open.
4. `CtwaService` — record conversions, post to Meta CAPI (new `MetaConversionsClient`).
5. Emit conversions from order-paid / flow-complete / manual-mark hooks.
6. `CtwaController` (or fold into dashboard) — attribution report (by ad: leads, conversions, value).
7. Frontend: an "Ad attribution" dashboard section + a "mark as converted" inbox action.

## Phasing
- **Phase A** — capture referral + show ad source on the conversation/contact.
- **Phase B** — attribution dashboard (leads/conversions by ad).
- **Phase C** — CAPI conversion postback for ad optimisation.

## Open questions
1. CAPI postback in MVP or just inbound attribution first (recommended: capture first, postback later).
2. Which conversion events to support initially (Lead + Purchase recommended).
3. Reuse the dashboard module ([[mvp-02-dashboard-analytics]]) for reporting vs a standalone page.

---

# Implementation detail (deep-dive)

> Implementation-ready spec. Mostly a small extension of existing inbound handling plus one new Meta
> client. New code: `Conversation` entity columns, `domain/ctwa/CtwaConversion` + repo + `CtwaService`,
> `integration/meta/MetaConversionsClient`, a hook in `MetaWebhookProcessor`, and a report endpoint
> (folded into the dashboard module, [[mvp-02-dashboard-analytics]]). Matched 2026-06-20. **Not built.**
> Confirm the `referral` object fields + the CAPI-for-CTWA endpoint against live Meta docs at build.

## A. Endpoint contracts

### `GET /api/dashboard/ctwa?from=&to=` → `CtwaReportResponse`  (JWT-scoped; folds into DashboardController)
```jsonc
{
  "byAd": [
    { "adSourceId":"23851...", "headline":"Diwali Sale",
      "conversations":120, "leads":44, "purchases":12, "value":86400.00, "currency":"INR" }
  ],
  "totals": { "conversations":120, "leads":44, "purchases":12, "value":86400.00 }
}
```

### `POST /api/conversations/{id}/mark-converted` → 200  (manual conversion from the inbox)
```jsonc
// MarkConvertedRequest
{ "eventName":"Lead", "value": null, "currency":"INR" }   // eventName: Lead | Purchase | ...
```
Records a `ctwa_conversions` row from the conversation's stored `ctwa_clid`; Phase C also posts to Meta
CAPI. `409` if the conversation has no `ctwa_clid` (not ad-initiated). Inbox-access roles only (§5).

## B. Runtime sequences

### B1. Capture on the FIRST inbound (extends existing processor)
```
Meta webhook → MetaWebhookProcessor.handleInbound (after dedup rule5, after recordInbound):
   if message.referral present AND conversation.ctwa_clid IS NULL:        ← capture-once guard
       ctwaService.attachReferral(conversation, message.referral)
         ├ conversation.ctwaClid      = referral.ctwa_clid
         ├ conversation.adSourceId    = referral.source_id
         ├ conversation.adSourceType  = referral.source_type
         ├ conversation.adReferral    = referral (full jsonb, audit)
         └ contact.attributes += { ctwa_clid, ad_source_id }   (segmentation)
   // NEVER overwrite an existing clid on later messages (first-touch wins)
```

### B2. Conversion → Meta CAPI postback (Phase C)
```
domain event (order paid [growth-03] / flow complete [growth-01] / manual mark)
   → CtwaService.recordConversion(conversationId, eventName, value, currency)
        ├ INSERT ctwa_conversions(ctwa_clid, event_name, value, …)
        └ MetaConversionsClient.sendEvent(datasetId, token, ctwaClid, eventName, value)   // /{dataset_id}/events
             → set sent_to_meta_at  (dedup: Meta dedups on a stable event_id we supply)
```

## C. Concrete signatures

### `Conversation` entity — add columns (migration `ALTER TABLE conversations …`)
```java
@Column(name="ctwa_clid", columnDefinition="text") private String ctwaClid;
@Column(name="ad_source_id", length=100) private String adSourceId;
@Column(name="ad_source_type", length=40) private String adSourceType;
@JdbcTypeCode(SqlTypes.JSON) @Column(name="ad_referral", columnDefinition="jsonb")
private Map<String,Object> adReferral;
```

### `domain/ctwa/CtwaConversion` + repo
```java
@Entity @Table(name="ctwa_conversions") @Getter @Setter @NoArgsConstructor
public class CtwaConversion {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="conversation_id", nullable=false) private UUID conversationId;
  @Column(name="ctwa_clid", nullable=false, columnDefinition="text") private String ctwaClid;
  @Column(name="event_name", nullable=false, length=50) private String eventName;
  @Column(precision=12, scale=2) private BigDecimal value;
  @Column(length=8) private String currency;
  @Column(name="sent_to_meta_at") private Instant sentToMetaAt;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
public interface CtwaConversionRepository extends JpaRepository<CtwaConversion, UUID> {
  @Query("""
     SELECT c.adSourceId AS adSourceId, COUNT(DISTINCT c.id) AS conversations
     FROM Conversation c WHERE c.workspaceId=:ws AND c.ctwaClid IS NOT NULL
       AND c.createdAt >= :from AND c.createdAt < :to GROUP BY c.adSourceId""")
  List<AdConversationCount> conversationsByAd(UUID ws, Instant from, Instant to);
  // + conversionsByAd(...) grouping ctwa_conversions by event_name per ad
  List<CtwaConversion> findBySentToMetaAtIsNull();   // postback retry (Phase C)
}
```

### `integration/meta/MetaConversionsClient` + `CtwaService`
```java
public interface MetaConversionsClient {                  // separate Graph endpoint from messaging
  void sendEvent(String datasetId, String accessToken, String ctwaClid,
                 String eventName, BigDecimal value, String currency, String eventId);  // POST /{dataset_id}/events
}
@Service public class CtwaService {
  @Transactional void attachReferral(Conversation conv, JsonNode referral);   // capture-once
  @Transactional CtwaConversion recordConversion(UUID conversationId, UUID ws, String eventName, BigDecimal value, String currency);
  @Transactional(readOnly=true) CtwaReportResponse report(UUID ws, Instant from, Instant to);
}
```

### Touch-points
- `MetaWebhookProcessor.handleInbound` → after `recordInbound`, the capture-once branch above (inject `CtwaService`).
- `CtwaService.recordConversion` is called from: `ShopifyService` order-paid (growth-03),
  `FlowResponseService` (growth-01), and the manual `mark-converted` endpoint.
- Report endpoint added to `DashboardController` (`/api/dashboard/ctwa`).

## D. Tests
- Referral captured only on the first inbound; a second `referral`-bearing message does **not** overwrite.
- `report` groups conversations + conversions by ad and sums value; workspace-isolated (§13.8 — a
  different workspace's clids never appear).
- `MetaConversionsClient` with `MockRestServiceServer`: event posts to the dataset endpoint with the
  clid; `sent_to_meta_at` set; re-running uses the same `eventId` (Meta-side dedup).
