# Plan: WhatsApp Flows (Meta native in-chat forms)

> Status: **planned, not built.** Post-MVP growth feature (relates to ARCHITECTURE §12). Picked as the
> first growth feature: highest ROI-to-effort because Meta hosts the form rendering. Design 2026-06-20.

**Why first:** native multi-screen forms inside the chat (text inputs, dropdowns, date pickers,
carousels) — booking, lead capture, order placement without leaving WhatsApp. Directly serves the
clinics / coaching / real-estate targets (appointments) and D2C (lead gen). Vendors report up to
~300% higher completion vs web forms (treat as a vendor claim). **Low backend effort** — Meta renders
the form; we send a flow message and receive responses on the webhook we already have.

## How it works (Meta Cloud API)
1. **Author a Flow asset** in Meta (`POST /{waba_id}/flows`, then upload the Flow JSON, then publish).
   Flow JSON defines screens + components. We store the returned `flow_id`.
2. **Send a flow** as an interactive message: `type=interactive`, `interactive.type=flow`, referencing
   the published `flow_id` + a CTA button. Goes out via the existing `MetaMessageClient`.
3. **Receive the response** on the existing webhook: a completed flow arrives as an inbound message
   with an `interactive.nfm_reply` (a.k.a. flow response) JSON payload of the submitted fields.
   `MetaWebhookProcessor` routes it to a new `FlowResponseService`.

## Schema — new migration
```sql
CREATE TYPE flow_status AS ENUM ('draft','published','deprecated');

CREATE TABLE wa_flows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  waba_id UUID NOT NULL REFERENCES wa_business_accounts(id),
  meta_flow_id VARCHAR(100),
  name VARCHAR(100) NOT NULL,
  category VARCHAR(40),                 -- SIGN_UP / APPOINTMENT_BOOKING / LEAD_GENERATION ...
  status flow_status NOT NULL DEFAULT 'draft',
  flow_json JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX wa_flows_waba_name_idx ON wa_flows(waba_id, name);

CREATE TABLE flow_responses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  flow_id UUID NOT NULL REFERENCES wa_flows(id),
  contact_id UUID NOT NULL REFERENCES contacts(id),
  conversation_id UUID REFERENCES conversations(id),
  response JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX flow_responses_ws_idx ON flow_responses(workspace_id);
CREATE INDEX flow_responses_flow_idx ON flow_responses(flow_id);
```

## Non-obvious decisions
1. **Flows are scoped to the WABA**, like templates — author/publish per WABA, all numbers share them.
2. **Lifecycle mirrors templates**: draft → publish in Meta → store `meta_flow_id`. A `FlowController`
   + `MetaFlowClient` (new, alongside `MetaTemplateClient`) handle create/upload/publish/list.
3. **Sending a flow is still a message** — subject to the 24h-window rule (§13.2). Inside the window
   send freely; to *initiate* outside it, the flow CTA must ride on an approved **template** message.
4. **On response, do something useful**: create/update the contact's `attributes` from submitted
   fields, optionally add to a list/tag, and (Phase B) fire a follow-up. Store raw response for audit.
5. **Dedup** the flow-response inbound on `wa_message_id` like every other webhook message (rule 5).

## Build order (per §11)
1. Migration (`wa_flows`, `flow_responses`, `flow_status` enum).
2. Entities + enums.
3. Repositories (workspace-scoped).
4. `MetaFlowClient` (integration/meta) — create/upload/publish/list flows; send flow message.
5. `FlowService` (author/publish/send) + `FlowResponseService` (handle webhook payload → contact).
6. `FlowController` — CRUD + publish + send; wire `MetaWebhookProcessor` to flow responses.
7. Frontend: a Flows builder/list page (start with Meta's prebuilt templates: "Book appointment",
   "Get a quote", "Sign up") + a "send flow" action in inbox/campaigns.

## Phasing
- **Phase A** — author/publish/send + capture responses to contact attributes.
- **Phase B** — trigger a follow-up message on completion; attach flows to campaigns.
- **Phase C** — visual flow-JSON editor (until then, use Meta's prebuilt templates).

## Open questions
1. Visual editor vs start with Meta's prebuilt Flow templates (recommended: prebuilt first).
2. Endpoint Flows (data-exchange to our backend mid-flow) in scope, or static Flows only (recommended).
3. Where to surface "send flow" — inbox composer, campaigns, or both.

---

# Implementation detail (deep-dive)

> Implementation-ready spec. New code: entity/repo/service under `domain/flow`, controller under
> `api/flow`, `MetaFlowClient` alongside the existing `MetaTemplateClient`/`MetaMessageClient` in
> `integration/meta`, and a small hook in `MetaWebhookProcessor`. Conventions match the repo
> (WABA-scoped like templates, native enums, encrypted WABA token via `WaAccountService`). Matched
> 2026-06-20. **Not built.** Confirm the exact `nfm_reply` payload shape against live Meta docs at build.

## A. Endpoint contracts

All JWT-scoped, workspace from token. Authoring is a `marketing`/`admin` concern (agents send, don't
author) — enforce in the service. Flows are WABA-scoped: the service resolves the workspace's WABA
via `WaAccountService.requireAccount(workspaceId)` (1:1, §13.4).

### `GET /api/flows` → `List<FlowResponse>`
```jsonc
[ { "id":"…", "name":"Book appointment", "category":"APPOINTMENT_BOOKING",
    "status":"published", "metaFlowId":"1234567890", "updatedAt":"…" } ]
```

### `POST /api/flows` → 201 `FlowResponse`  (creates draft in Meta, stores `meta_flow_id`)
```jsonc
// CreateFlowRequest
{ "name":"Book appointment", "category":"APPOINTMENT_BOOKING", "flowJson": { /* Meta Flow JSON */ } }
```
Calls `MetaFlowClient.createFlow` + `updateFlowJson`; persists `status=draft`. Duplicate name in the
WABA → `DuplicateResourceException` (409) via `wa_flows_waba_name_idx`.

### `PUT /api/flows/{id}` → `FlowResponse`  (only `draft`; re-uploads Flow JSON)
### `POST /api/flows/{id}/publish` → `FlowResponse`  (`MetaFlowClient.publish` → status `published`)
`409 ConflictException` if already published / not draft (mirrors template submit lifecycle).

### `POST /api/flows/{id}/send` → 200 `MessageResponse`
```jsonc
// SendFlowRequest
{ "conversationId":"…", "flowCta":"Book now", "screen":"WELCOME", "flowToken":"opt" }
```
- Resolves conversation → contact + sender number.
- **§13.2 gate** (same `withinWindow` check `ConversationService.sendReply` uses): inside the window →
  send interactive flow message directly; outside → `WindowExpiredException` (must ride an approved
  template — Phase B attaches the flow CTA to a template). Only `published` flows may be sent (409 otherwise).
- Persists an outbound `Message` (`type=interactive`) via `MessageService.saveOutbound`.

### `GET /api/flows/{id}/responses?page=` → `Page<FlowResponseDataResponse>` (submitted form data)

## B. Runtime sequence

### B1. Author → publish → send
```
marketing UI         FlowController/FlowService        MetaFlowClient            Meta Graph API
   │ POST /api/flows  │                                   │                          │
   │─────────────────▶│ requireAccount(ws) (WABA+token)   │                          │
   │                  │ createFlow(waba,token,name,cat) ──▶│ POST /{waba_id}/flows ──▶│ → flow_id
   │                  │ updateFlowJson(flowId,json) ──────▶│ POST /{flow_id}/assets ─▶│
   │                  │ save wa_flows(status=draft)        │                          │
   │ POST /{id}/publish ─────────────────────────────────▶│ POST /{flow_id}/publish ▶│
   │                  │ status=published                   │                          │
   │ POST /{id}/send {conversationId} │                    │                          │
   │─────────────────▶│ §13.2 window check                │                          │
   │                  │ sendFlowMessage(number,token,to,flowId,cta) ─▶│ POST /{number}/messages
   │                  │ saveOutbound(type=interactive)     │   (type=interactive,flow)│
```

### B2. Customer submits → webhook captures (extends existing processor)
```
Meta webhook → WhatsAppWebhookController (verify sig, 200 fast)
   → MetaWebhookProcessor.processAsync (@Async webhookExecutor)
       handleInbound(sender, message):
         dedup wa_message_id (rule 5)
         if message.interactive.type == "nfm_reply":            ← NEW branch
              flowResponseService.handle(sender, contact, message)
                ├ parse interactive.nfm_reply.response_json (submitted fields)
                ├ INSERT flow_responses (raw payload, audit)
                ├ map fields → contact.attributes (+ optional tag/list)
                └ Phase B: enqueue follow-up / trigger bot (growth-02 flow_complete trigger)
         else → existing inbound handling
```

## C. Concrete class signatures

### `integration/meta/MetaFlowClient` (interface + `MetaFlowClientImpl` using RestClient)
```java
public interface MetaFlowClient {
  String createFlow(String wabaId, String accessToken, String name, String category);   // → meta flow_id
  void   updateFlowJson(String metaFlowId, String accessToken, JsonNode flowJson);
  void   publish(String metaFlowId, String accessToken);
  void   deprecate(String metaFlowId, String accessToken);
  /** Sends an interactive flow message (POST /{phone-number-id}/messages, type=interactive). @return wamid. */
  String sendFlowMessage(String phoneNumberId, String accessToken, String toPhone,
                         String metaFlowId, String ctaText, String screen, String flowToken);
}
```

### `domain/flow/WaFlow` + `FlowResponse` (entity) + `FlowStatus`
```java
@Entity @Table(name="wa_flows") @Getter @Setter @NoArgsConstructor
public class WaFlow {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="waba_id", nullable=false) private UUID wabaId;
  @Column(name="meta_flow_id", length=100) private String metaFlowId;
  @Column(nullable=false, length=100) private String name;
  @Column(length=40) private String category;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(nullable=false, columnDefinition="flow_status")
  private FlowStatus status = FlowStatus.draft;
  @JdbcTypeCode(SqlTypes.JSON) @Column(name="flow_json", columnDefinition="jsonb")
  private Map<String,Object> flowJson = new HashMap<>();
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
  @UpdateTimestamp  @Column(name="updated_at", nullable=false) private Instant updatedAt;
}
public enum FlowStatus { draft, published, deprecated }

@Entity @Table(name="flow_responses") @Getter @Setter @NoArgsConstructor
public class FlowResponseRecord {                     // “FlowResponse” name avoided — clashes with the DTO
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="flow_id", nullable=false) private UUID flowId;
  @Column(name="contact_id", nullable=false) private UUID contactId;
  @Column(name="conversation_id") private UUID conversationId;
  @JdbcTypeCode(SqlTypes.JSON) @Column(columnDefinition="jsonb") private Map<String,Object> response = new HashMap<>();
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
```

### Repositories + services
```java
public interface WaFlowRepository extends JpaRepository<WaFlow, UUID> {
  List<WaFlow> findByWorkspaceIdOrderByUpdatedAtDesc(UUID ws);
  Optional<WaFlow> findByIdAndWorkspaceId(UUID id, UUID ws);
}
public interface FlowResponseRepository extends JpaRepository<FlowResponseRecord, UUID> {
  Page<FlowResponseRecord> findByFlowIdAndWorkspaceId(UUID flowId, UUID ws, Pageable p);
}

@Service public class FlowService {       // author/publish/send; WABA-scoped via WaAccountService
  List<FlowResponse> list(UUID ws);
  FlowResponse create(UUID ws, CreateFlowRequest req);
  FlowResponse update(UUID id, UUID ws, UpdateFlowRequest req);   // draft only
  FlowResponse publish(UUID id, UUID ws);
  MessageResponse send(UUID ws, UUID userId, UUID flowId, SendFlowRequest req);   // §13.2 gate
}
@Service public class FlowResponseService {   // webhook side
  @Transactional void handle(WaPhoneNumber sender, Contact contact, JsonNode nfmReplyMessage);
}
```

### `api/flow/FlowController` + DTOs (`api/flow/dto/`)
```java
@RestController @RequestMapping("/api/flows")
public class FlowController { /* list, create(201), update, publish, send, responses — all thin */ }

public record CreateFlowRequest(@NotBlank @Size(max=100) String name, @Size(max=40) String category,
                                @NotNull Map<String,Object> flowJson) {}
public record SendFlowRequest(@NotNull UUID conversationId, @NotBlank String flowCta,
                              String screen, String flowToken) {}
public record FlowResponse(UUID id, String name, String category, FlowStatus status,
                           String metaFlowId, Instant updatedAt) { public static FlowResponse from(WaFlow f){…} }
public record FlowResponseDataResponse(UUID id, UUID contactId, Map<String,Object> response, Instant createdAt) {}
```

## D. Touch-point in existing code
`MetaWebhookProcessor.handleInbound` — after the dedup check and before the generic
`conversationService.recordInbound`, branch on `message.path("interactive").path("type").asText()`:
if `nfm_reply`, delegate to `FlowResponseService.handle(...)`. Inject `FlowResponseService` into the
processor (one new constructor arg). Everything else unchanged.

## E. Tests
- `MetaFlowClientImpl` with `MockRestServiceServer`: create → flow_id parsed; publish posts to the
  right URL; never hits `graph.facebook.com`.
- `FlowService.send`: outside 24h window → `WindowExpiredException`; unpublished flow → 409.
- `FlowResponseService.handle`: `nfm_reply` payload maps to `contact.attributes`; duplicate
  `wa_message_id` → no second `flow_responses` row (rule 5).
