# Plan: No-code Chatbot / Flow Builder

> Status: **planned, not built.** Post-MVP growth feature (ARCHITECTURE §12, "chatbot / no-code flow
> builder"). The strategic centerpiece — the category's #1 buying criterion. Design 2026-06-20.

**Why this is the big one:** every competitor leads with it (Wati ~200-step flows; AiSensy AI flow
builder). Without it the product is positioned as "just a broadcaster." It also becomes the foundation
for trigger-based automation and the AI agent ([[growth-06-ai-agent]]). Builds directly on infra we
already have: the inbound webhook (`MetaWebhookProcessor`), `MetaMessageClient`, and `MessageService`.

## Concept
A **bot** = a trigger + a directed graph of nodes executed per-conversation. Inbound message →
match a trigger → run/advance the contact's flow session → send the next node's message(s).

- **Triggers**: keyword match, "any first message" (welcome), button/quick-reply click, or a Flow
  completion ([[growth-01-whatsapp-flows]]).
- **Node types (MVP)**: send message (text/template/media), ask-a-question (capture reply into a
  contact attribute), buttons/list (branch on choice), condition (branch on attribute), delay,
  assign-to-agent / handoff, add-tag / add-to-list, end.

## Schema — new migration
```sql
CREATE TYPE bot_status AS ENUM ('draft','active','paused');
CREATE TYPE flow_session_status AS ENUM ('active','completed','handed_off','expired');

CREATE TABLE bots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  name VARCHAR(100) NOT NULL,
  status bot_status NOT NULL DEFAULT 'draft',
  trigger_type VARCHAR(30) NOT NULL,           -- keyword | welcome | button | flow_complete
  trigger_config JSONB NOT NULL DEFAULT '{}',  -- e.g. {"keywords":["price","pricing"]}
  graph JSONB NOT NULL DEFAULT '{}',           -- nodes + edges (visual builder canvas)
  priority INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bots_ws_status_idx ON bots(workspace_id, status);

CREATE TABLE flow_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  bot_id UUID NOT NULL REFERENCES bots(id),
  conversation_id UUID NOT NULL REFERENCES conversations(id),
  contact_id UUID NOT NULL REFERENCES contacts(id),
  current_node_id VARCHAR(50),
  status flow_session_status NOT NULL DEFAULT 'active',
  context JSONB NOT NULL DEFAULT '{}',          -- variables collected so far
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX flow_sessions_active_conv_idx
  ON flow_sessions(conversation_id) WHERE status = 'active';   -- one active bot per conversation
```

## Non-obvious decisions (where this gets hard)
1. **24h-window rule is the hard constraint (§13.2).** A bot reply is free text — only allowed inside
   the window. Welcome/keyword bots fire right after a customer message, so they're naturally in
   window. But a **delay node** that resumes hours later can cross the boundary → the engine must
   check the window and, if expired, only continue via an **approved template** or pause the session.
   This is the single biggest correctness trap.
2. **One active session per conversation** (partial unique index) — a new inbound advances the existing
   session, it doesn't start a second bot. Agent handoff sets `handed_off` and stops automation.
3. **Engine runs on the existing webhook path**, `@Async("webhookExecutor")` — but a single inbound
   may emit several outbound messages (multi-node hop until the next "wait for reply"); keep the Meta
   rate-limit delay (§5) and persist node position after each send for crash recovery.
4. **Delay nodes need a scheduler**, not a sleeping thread — a `@Scheduled` sweep over
   `flow_sessions` whose delay is due (same pattern as `CampaignScheduler`).
5. **Loop / runaway protection** — cap nodes-per-inbound and total session hops; `expires_at` reaps
   stale sessions. Without this a mis-built graph can spam a contact.
6. **Don't bot unsubscribed/blocked contacts** beyond what they initiate; respect §13 isolation.

## Build order (per §11)
1. Migration (`bots`, `flow_sessions`, enums).
2. Entities + enums (native-enum mapping; `graph`/`context` as jsonb).
3. Repositories (workspace-scoped; active-session lookup by conversation).
4. `BotService` (CRUD/activate) + **`FlowEngine`** (match trigger → load/advance session → execute
   nodes → send via `MetaMessageClient`) + `FlowSessionScheduler` (resume delays).
5. Hook `FlowEngine` into `MetaWebhookProcessor` *before* SSE push, after dedup.
6. `BotController` (CRUD/activate/test).
7. Frontend: a **visual builder** (React Flow canvas) — drag nodes, connect edges, configure each,
   publish. Plus a bots list + a live "test in sandbox" pane.

## Phasing
- **Phase A** — engine + keyword/welcome triggers + core nodes (message, question, buttons, condition,
  handoff, tag). JSON-config builder (no canvas yet) to validate the engine.
- **Phase B** — visual canvas builder (React Flow).
- **Phase C** — delay nodes + scheduler, Flow-completion triggers, analytics (entry/exit/drop-off).

## Open questions
1. Builder library: **React Flow** (recommended) vs a custom canvas.
2. MVP node set — confirm the list above is the right minimum.
3. AI flow generation ("describe in English") now or after the AI-agent work (recommended: after).
4. One bot per conversation vs multiple concurrent (recommended: one active at a time).

---

# Implementation detail (deep-dive)

> Implementation-ready spec. New code under `domain/bot` (entity/repo/service/engine/scheduler) and
> `api/bot` (controller/DTOs). The engine hooks into the existing `MetaWebhookProcessor` async path
> and sends via the existing `MetaMessageClient`. The hardest part is **not** CRUD — it's the engine's
> 24h-window handling, idempotency, and runaway protection. Matched to the repo 2026-06-20. **Not built.**

## A. Endpoint contracts (authoring is the easy half)

JWT-scoped, workspace from token; bot authoring is `admin`/`marketing`. The interesting contract is
the **graph JSON** shape, not the CRUD verbs.

### `GET /api/bots` → `List<BotResponse>` · `POST /api/bots` (201) · `PUT /api/bots/{id}` · `DELETE /api/bots/{id}`
```jsonc
// CreateBotRequest / UpdateBotRequest
{
  "name": "Pricing bot",
  "triggerType": "keyword",                       // keyword | welcome | button | flow_complete
  "triggerConfig": { "keywords": ["price","pricing","cost"] },
  "graph": {
    "entryNodeId": "n1",
    "nodes": [
      { "id":"n1", "type":"send_message", "config":{ "text":"Hi {{name}}! What do you need pricing for?" }, "next":"n2" },
      { "id":"n2", "type":"buttons", "config":{ "text":"Pick one", "buttons":[
            {"id":"plans","title":"Plans","next":"n3"},
            {"id":"demo","title":"Book demo","next":"n4"} ] } },
      { "id":"n3", "type":"send_message", "config":{ "text":"Our plans start at ₹999/mo." }, "next":"end" },
      { "id":"n4", "type":"assign_agent", "config":{}, "next":"end" }
    ]
  }
}
```

### `POST /api/bots/{id}/activate` → `BotResponse`  (`draft|paused` → `active`; validates the graph first)
Validation (409 `ConflictException` on failure): single reachable `entryNodeId`, every `next` resolves
or is `"end"`, no orphan nodes, button/condition branches all target valid nodes, **no unbounded cycle
without a wait/delay node** (runaway guard at author time).

### `POST /api/bots/{id}/test` → `BotTestResponse`  (dry-run the graph against a simulated inbound; no Meta send)

## B. Runtime sequence — the engine on the webhook path

```
Meta webhook → WhatsAppWebhookController (verify, 200 fast)
   → MetaWebhookProcessor.processAsync (@Async webhookExecutor)
       handleInbound: dedup (rule5) → recordInbound (existing) → THEN:
   ┌─────────────────────────── FlowEngine.onInbound(sender, contact, conversation, message) ───────────────────────────┐
   │ session = repo.findActiveByConversationId(convId)                                                                   │
   │ if session exists:                                                                                                   │
   │     node = graph.node(session.currentNodeId)                                                                         │
   │     advance: apply the customer's reply to the waiting node (capture answer / match button / eval condition)        │
   │ else:                                                                                                                │
   │     bot = botRepo.findFirstActiveMatching(ws, message)   // keyword/welcome, ordered by priority                    │
   │     if none → return (no automation); else create flow_session(active, entryNode), node = entry                     │
   │                                                                                                                      │
   │ loop (hopBudget = MAX_HOPS_PER_INBOUND):              // multi-node hop until next "wait for reply" / end           │
   │     switch node.type:                                                                                                │
   │       send_message/template/media → §13.2 WINDOW CHECK → MetaMessageClient.send… (+12ms rate delay, §5)             │
   │                                       if window expired & not template → pause session, stop                        │
   │       question/buttons/list       → send prompt, set currentNodeId, status stays active, BREAK (await reply)        │
   │       condition                   → pick branch from context, continue                                              │
   │       delay                       → set resumeAt = now+delay, BREAK (FlowSessionScheduler resumes later)            │
   │       assign_agent/handoff        → conversation.assignedTo set, session.status=handed_off, BREAK                    │
   │       add_tag/add_to_list         → mutate contact, continue                                                         │
   │       end                         → session.status=completed, BREAK                                                  │
   │     persist currentNodeId + context AFTER EACH SEND (crash recovery); decrement hopBudget; if 0 → pause + log       │
   └──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```
Delay resumption (separate path, mirrors `CampaignScheduler`):
```
FlowSessionScheduler @Scheduled(fixedDelay=60_000):
   sessions = repo.findByStatusAndResumeAtLessThanEqual(active, now)
   for each: FlowEngine.resume(session)  // re-enters the loop at currentNodeId; own transaction per session
             → on resume, RE-CHECK §13.2 window before any free-text send (the customer may have gone silent for hours)
```

## C. Concrete class signatures

### Entities (`domain/bot/`)
```java
@Entity @Table(name="bots") @Getter @Setter @NoArgsConstructor
public class Bot {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(nullable=false, length=100) private String name;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(nullable=false, columnDefinition="bot_status")
  private BotStatus status = BotStatus.draft;
  @Column(name="trigger_type", nullable=false, length=30) private String triggerType;
  @JdbcTypeCode(SqlTypes.JSON) @Column(name="trigger_config", columnDefinition="jsonb") private Map<String,Object> triggerConfig = new HashMap<>();
  @JdbcTypeCode(SqlTypes.JSON) @Column(columnDefinition="jsonb") private Map<String,Object> graph = new HashMap<>();
  @Column(nullable=false) private int priority;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
  @UpdateTimestamp  @Column(name="updated_at", nullable=false) private Instant updatedAt;
}
@Entity @Table(name="flow_sessions") @Getter @Setter @NoArgsConstructor
public class FlowSession {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="bot_id", nullable=false) private UUID botId;
  @Column(name="conversation_id", nullable=false) private UUID conversationId;
  @Column(name="contact_id", nullable=false) private UUID contactId;
  @Column(name="current_node_id", length=50) private String currentNodeId;
  @JdbcTypeCode(SqlTypes.NAMED_ENUM) @Column(nullable=false, columnDefinition="flow_session_status")
  private FlowSessionStatus status = FlowSessionStatus.active;
  @JdbcTypeCode(SqlTypes.JSON) @Column(columnDefinition="jsonb") private Map<String,Object> context = new HashMap<>();
  @Column(name="resume_at") private Instant resumeAt;       // for delay nodes (add column to migration)
  @Column(name="expires_at") private Instant expiresAt;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
  @UpdateTimestamp  @Column(name="updated_at", nullable=false) private Instant updatedAt;
}
public enum BotStatus { draft, active, paused }
public enum FlowSessionStatus { active, completed, handed_off, expired }
```
> Add `resume_at TIMESTAMPTZ` to the `flow_sessions` migration (the schema block above predates the
> delay-scheduler design). Index: `CREATE INDEX flow_sessions_resume_idx ON flow_sessions(status, resume_at) WHERE status='active';`

### Repositories
```java
public interface FlowSessionRepository extends JpaRepository<FlowSession, UUID> {
  Optional<FlowSession> findByConversationIdAndStatus(UUID conversationId, FlowSessionStatus status);   // active lookup
  List<FlowSession> findByStatusAndResumeAtLessThanEqual(FlowSessionStatus status, Instant cutoff);     // scheduler
  List<FlowSession> findByStatusAndExpiresAtLessThanEqual(FlowSessionStatus status, Instant cutoff);    // reaper
}
public interface BotRepository extends JpaRepository<Bot, UUID> {
  Optional<Bot> findByIdAndWorkspaceId(UUID id, UUID ws);
  List<Bot> findByWorkspaceIdAndStatusOrderByPriorityDesc(UUID ws, BotStatus status);   // trigger matching
}
```

### Engine + scheduler + service
```java
@Component
public class FlowEngine {                          // NOT @Transactional at class level — see note
  /** Entry from the webhook. Matches/advances a session and drives the node loop. */
  @Transactional void onInbound(WaPhoneNumber sender, Contact contact, Conversation conv, JsonNode inbound);
  /** Entry from the scheduler for a due delay node. Re-checks the §13.2 window before any free-text send. */
  @Transactional void resume(FlowSession session);
  // private NodeOutcome execute(node, session, ...) ; private boolean withinWindow(Conversation) (same calc as ConversationService)
  static final int MAX_HOPS_PER_INBOUND = 25;       // runaway guard
}
@Component
public class FlowSessionScheduler {                 // mirrors CampaignScheduler
  @Scheduled(fixedDelay=60_000) void resumeDueDelays();   // findByStatusAndResumeAtLessThanEqual → engine.resume (own txn each)
  @Scheduled(fixedDelay=300_000) void reapExpired();      // active & expires_at passed → status=expired
}
@Service
public class BotService { /* list/create/update/delete/activate(validateGraph)/test — workspace-scoped */ }
```
> **Self-invocation caveat (same as billing/import):** the scheduler and webhook processor must call
> `FlowEngine` as an injected bean so `@Transactional`/`@Async` proxies apply. The engine itself runs
> synchronously *on* the `webhookExecutor` thread (it's invoked from the already-async processor) — do
> not annotate `onInbound` with a second `@Async`.

### `api/bot/BotController` + DTOs
```java
@RestController @RequestMapping("/api/bots")
public class BotController { /* CRUD + activate + test, thin, SecurityUtils workspace scoping */ }
public record CreateBotRequest(@NotBlank String name, @NotBlank String triggerType,
                               Map<String,Object> triggerConfig, @NotNull Map<String,Object> graph,
                               Integer priority) {}
public record BotResponse(UUID id, String name, BotStatus status, String triggerType,
                          Map<String,Object> triggerConfig, Map<String,Object> graph, int priority, Instant updatedAt) {}
public record BotTestResponse(List<String> emittedMessages, String endedAtNodeId, String terminalStatus) {}
```

## D. The three correctness invariants (call these out in code review)
1. **§13.2 window before every free-text send**, and re-checked on delayed `resume` — the single
   biggest trap. Template sends are exempt; free text outside the window pauses the session.
2. **Idempotency**: an inbound is deduped (rule 5) *before* the engine runs, and node position is
   persisted after each send, so an SQS/webhook redelivery or a crash mid-flow never double-sends past
   the last committed node.
3. **Bounded execution**: `MAX_HOPS_PER_INBOUND`, the author-time cycle check, and `expires_at` reaping
   together guarantee a mis-built graph cannot spam a contact.

## E. Tests
- Engine unit tests over crafted graphs: keyword match starts a session; button reply branches; a
  cycle without a wait node trips the hop budget → paused; delay node sets `resumeAt` and emits nothing
  until the scheduler fires.
- Window: a delay that resumes >24h after the last customer message and hits a free-text node → no send,
  session paused (assert no `MetaMessageClient` call).
- One-active-session invariant: a second inbound on the same conversation advances, never creates a
  second `active` row (partial unique index also backstops).
