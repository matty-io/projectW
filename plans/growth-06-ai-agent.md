# Plan: AI Agent (LLM-powered auto-reply)

> Status: **planned, not built.** Post-MVP growth feature (ARCHITECTURE §12, "AI reply suggestions").
> The differentiation frontier — builds on the flow builder. Design 2026-06-20.

**Why:** the newest competitors (Gallabox, AiSensy AI Agents) moved past rule-based flows to NLP agents
trained on business data that qualify leads and answer support autonomously, with human handoff. This
is where the market is heading; doing it well is a way to leapfrog rather than catch up. It builds on
the inbox + flow engine ([[growth-02-flow-builder]]) already planned.

## Concept
Inbound message → retrieve relevant business context (RAG over the customer's FAQ/docs) → ask an LLM
for a reply → send within the 24h window, with **confidence-based handoff** to a human agent.

Two modes:
- **Copilot (Phase A)** — suggest a reply in the inbox; the agent edits/sends. Low risk, immediate value.
- **Autopilot (Phase B)** — the agent answers automatically until it's unsure or the customer asks for
  a human, then hands off (sets conversation `assigned_to` + flags it).

## Model choice
Default to **Claude Haiku 4.5** (`claude-haiku-4-5`) for per-message replies — cheap and fast at inbox
volume — and escalate hard cases to **Claude Sonnet 4.6** (`claude-sonnet-4-6`). Embeddings via a
dedicated embeddings model. **Confirm current model IDs + pricing via the `claude-api` skill at build
time** rather than hardcoding from memory.

## Schema — new migration
```sql
CREATE TABLE ai_agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID UNIQUE NOT NULL REFERENCES workspaces(id),
  name VARCHAR(100) NOT NULL,
  system_prompt TEXT,                    -- persona/instructions
  mode VARCHAR(20) NOT NULL DEFAULT 'copilot',  -- copilot | autopilot
  handoff_threshold NUMERIC(3,2) DEFAULT 0.60,
  is_active BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE knowledge_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  agent_id UUID NOT NULL REFERENCES ai_agents(id),
  type VARCHAR(20) NOT NULL,             -- faq | url | file | catalog
  title VARCHAR(200),
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- requires the pgvector extension
CREATE TABLE knowledge_chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  source_id UUID NOT NULL REFERENCES knowledge_sources(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  embedding vector(1024),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX knowledge_chunks_embedding_idx ON knowledge_chunks
  USING hnsw (embedding vector_cosine_ops);
```

## Non-obvious decisions
1. **24h-window rule still applies (§13.2)** — AI replies are free text, only allowed in window.
   Autopilot must check the window and fall back to a template / stop if expired.
2. **RAG is per-workspace and strictly isolated (§13.8)** — every vector query filters `workspace_id`.
   Never let one tenant's knowledge leak into another's retrieval.
3. **Confidence + handoff** — the model returns an answer + a confidence/"should I escalate" signal;
   below threshold or on explicit "talk to a human" → assign to an agent and stop automating. Always
   leave an escape hatch.
4. **Guardrails** — system prompt constrains the agent to business topics; never invent
   prices/policies not in the knowledge base; log every AI message (mark `messages.sent_by = NULL` +
   an `ai` flag) for auditability.
5. **Cost control** — cap tokens/context, cache embeddings, prefer Haiku; meter usage per workspace
   (ties into [[mvp-03-billing-monetization]] — AI replies could be a metered add-on).
6. **pgvector is a new extension** — add to the DB + a Flyway migration; not currently enabled.

## Build order (per §11)
1. Migration (`ai_agents`, `knowledge_sources`, `knowledge_chunks`; enable `vector`).
2. Entities + repositories (vector search via native query).
3. `integration/llm/AnthropicClient` (chat + embeddings) — see the `claude-api` skill.
4. `KnowledgeService` (ingest → chunk → embed) + `AiAgentService` (retrieve → prompt → reply +
   confidence) + handoff logic.
5. Inbox hook: copilot suggestion endpoint; autopilot path in `MetaWebhookProcessor`/`FlowEngine`.
6. `AiAgentController` (config + knowledge upload).
7. Frontend: settings/AI agent (persona, mode, knowledge upload) + inbox "suggested reply" UI.

## Phasing
- **Phase A — Copilot**: suggest replies in the inbox (RAG + Claude), agent approves. Low risk.
- **Phase B — Autopilot**: auto-answer with confidence-based handoff.
- **Phase C**: AI flow generation ("describe your bot in English") feeding [[growth-02-flow-builder]];
  lead-qualification scoring.

## Open questions
1. Start **copilot-only** (recommended) vs autopilot from the start.
2. pgvector in-DB (recommended, simplest) vs a managed vector store.
3. Meter AI as a billed add-on now or free during beta (recommended: free in beta, meter later).
4. Confirm Claude model IDs/pricing via the `claude-api` skill before implementing.

---

# Implementation detail (deep-dive)

> Implementation-ready spec. New code under `domain/ai` (entities/repos/services), `api/ai`
> (controller/DTOs), `integration/llm/AnthropicClient`. Copilot (Phase A) adds a suggest endpoint to
> the inbox; autopilot (Phase B) hooks the webhook path with the same §13.2 + handoff guards the flow
> engine uses. Matched 2026-06-20. **Not built. Confirm Claude model IDs/pricing via the `claude-api`
> skill at build time — do not hardcode from memory.**

## A. Endpoint contracts

### Copilot — `POST /api/conversations/{id}/ai-suggest` → `AiSuggestionResponse`  (inbox-access roles)
```jsonc
// 200 — suggestion the agent edits/sends (does NOT send)
{ "suggestedReply":"Yes, we ship to Pune in 3–4 days. Want me to share sizes?",
  "confidence":0.82, "shouldEscalate":false,
  "sources":[ {"sourceId":"…","title":"Shipping FAQ"} ] }
```
Runs RAG + Claude, returns text + confidence; the agent sends via the existing
`POST /api/conversations/{id}/messages` (so §13.2 still applies at send). No auto-send here.

### Agent config (admin) — CRUD + knowledge
```
GET  /api/ai/agent                          → AiAgentResponse | 404
PUT  /api/ai/agent                          → upsert (persona, mode, handoff_threshold, is_active)
POST /api/ai/agent/knowledge                → 202 add a source (faq text | url | file) → async ingest
GET  /api/ai/agent/knowledge                → List<KnowledgeSourceResponse> (with ingest status)
DELETE /api/ai/agent/knowledge/{sourceId}   → 204 (cascades chunks)
```

## B. Runtime sequences

### B1. Copilot suggestion
```
agent (inbox)        AiAgentController/AiAgentService        AnthropicClient            Postgres(pgvector)
   │ POST /ai-suggest │                                         │                          │
   │─────────────────▶│ load agent(ws) + last N messages        │                          │
   │                  │ embed(customerText) ───────────────────▶│ (embeddings model)       │
   │                  │ vector search knowledge_chunks WHERE workspace_id=:ws ─────────────▶│ ORDER BY embedding <=> :q LIMIT k
   │                  │ prompt = system_prompt + retrieved chunks + history                 │
   │                  │ chat(Haiku 4.5, prompt) ───────────────▶│ → {reply, confidence, escalate}
   │ ◀── suggestion ──│                                         │                          │
```

### B2. Autopilot (Phase B) — on the webhook path
```
MetaWebhookProcessor.handleInbound (after dedup + recordInbound):
   if agent.is_active && agent.mode==autopilot && no active flow_session (growth-02 takes precedence):
       §13.2 WINDOW CHECK — outside window → stop (cannot free-text), leave for human
       suggestion = aiAgentService.generate(conversation)
       if suggestion.confidence >= agent.handoff_threshold && !suggestion.shouldEscalate:
            MetaMessageClient.sendText(...) ; saveOutbound(sentBy=NULL, ai-flag) ; log AI message
       else:
            conversation.assignedTo = round-robin/owner ; flag for human ; STOP automating  ← escape hatch
```

## C. Concrete signatures

### `integration/llm/AnthropicClient`
```java
public interface AnthropicClient {
  /** Chat completion. modelId resolved from config — confirm via claude-api skill (Haiku 4.5 default). */
  AiReply chat(String modelId, String systemPrompt, List<ChatTurn> history, String userMessage);
  /** Embeddings for RAG ingest + query. */
  float[] embed(String modelId, String text);
  record ChatTurn(String role, String content) {}
  record AiReply(String text, double confidence, boolean escalate) {}   // confidence/escalate parsed from a structured tool/JSON reply
}
```

### Entities (`domain/ai/`) — pgvector chunk uses a native query
```java
@Entity @Table(name="ai_agents") @Getter @Setter @NoArgsConstructor
public class AiAgent {
  @Id @GeneratedValue(strategy=GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;   // UNIQUE (one agent per ws)
  @Column(nullable=false, length=100) private String name;
  @Column(name="system_prompt", columnDefinition="text") private String systemPrompt;
  @Column(nullable=false, length=20) private String mode = "copilot";       // copilot | autopilot
  @Column(name="handoff_threshold", precision=3, scale=2) private BigDecimal handoffThreshold = new BigDecimal("0.60");
  @Column(name="is_active", nullable=false) private boolean active = false;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
}
// KnowledgeSource (type/status); KnowledgeChunk(content, embedding vector(1024)) — embedding mapped as a
// native column; nearest-neighbour search goes through a @Query(nativeQuery=true) (JPA has no vector type).
```

### Repositories — vector search is native SQL
```java
public interface AiAgentRepository extends JpaRepository<AiAgent, UUID> {
  Optional<AiAgent> findByWorkspaceId(UUID ws);
}
public interface KnowledgeChunkRepository extends JpaRepository<KnowledgeChunk, UUID> {
  /** Workspace-isolated (§13.8) nearest-neighbour retrieval. :q is the query embedding (pgvector literal). */
  @Query(value = """
     SELECT * FROM knowledge_chunks
     WHERE workspace_id = :ws
     ORDER BY embedding <=> CAST(:q AS vector)
     LIMIT :k""", nativeQuery = true)
  List<KnowledgeChunk> searchNearest(@Param("ws") UUID ws, @Param("q") String queryVectorLiteral, @Param("k") int k);
}
```

### Services + controller
```java
@Service public class KnowledgeService {        // ingest pipeline
  @Async("importExecutor") void ingest(UUID sourceId);   // fetch → chunk → embed → INSERT chunks (reuse import pool)
}
@Service public class AiAgentService {
  @Transactional(readOnly=true) AiSuggestionResponse suggest(UUID conversationId, UUID ws);  // copilot
  AiReply generate(Conversation conv);                                                        // autopilot core (RAG+chat)
  AiAgentResponse upsert(UUID ws, UpsertAgentRequest req);
}
@RestController @RequestMapping("/api/ai")
public class AiAgentController { /* agent GET/PUT, knowledge add(202)/list/delete, + suggest lives on ConversationController */ }

public record AiSuggestionResponse(String suggestedReply, double confidence, boolean shouldEscalate,
                                   List<SourceRef> sources) { public record SourceRef(UUID sourceId, String title){} }
public record UpsertAgentRequest(@NotBlank String name, String systemPrompt,
                                 @Pattern(regexp="copilot|autopilot") String mode,
                                 @DecimalMin("0") @DecimalMax("1") BigDecimal handoffThreshold, boolean active) {}
```

## D. The non-negotiable guards (review checklist)
1. **§13.2 window** — autopilot free-text only inside the window; outside → hand to human, never send.
2. **Workspace isolation of RAG (§13.8)** — every `searchNearest` filters `workspace_id`; one tenant's
   knowledge can never surface in another's retrieval. This is the highest-risk leak in the whole spec.
3. **Confidence handoff + explicit "talk to a human"** → assign + stop automating (always an escape hatch).
4. **Audit** — every AI message persisted with `sent_by=NULL` + an `ai` marker; never invent
   prices/policies absent from the knowledge base (system-prompt guardrail).
5. **Flow engine precedence** — if an active `flow_session` (growth-02) owns the conversation, the AI
   agent stands down (no double-replying).
6. **Cost** — Haiku default, cap context/tokens, cache embeddings; meter per workspace for
   [[mvp-03-billing-monetization]] if billed.

## E. Tests
- `searchNearest` (Testcontainers + pgvector): seeded chunks for workspace A are never returned for a
  workspace-B query, even with identical text.
- Autopilot: confidence below threshold → conversation assigned, no Meta send; outside 24h window → no
  send regardless of confidence.
- `AnthropicClient` stubbed (never call the real API in tests); ingest produces chunk rows with embeddings.
