# Plan: Quick Replies (canned inbox responses)

> Status: **planned, not built.** Source of truth for schema/endpoints remains `ARCHITECTURE.md`
> (§4 `quick_replies` table, §7 endpoints); this doc captures the design agreed on 2026-06-20.

**Why:** agent-productivity layer for the shared inbox — saved messages with `/shortcut` expansion so
agents answer common questions in one keystroke. Lowest urgency of the four gaps, but cheap because
the schema and API contract already exist.

## Already exists vs missing
- ✅ `quick_replies` table in `V1` (`workspace_id, title, content, shortcut`), with a partial unique
  index on `(workspace_id, shortcut) WHERE shortcut IS NOT NULL`.
- ✅ §7 endpoints specified: `GET/POST/PUT/DELETE /api/quick-replies`.
- ❌ No `QuickReply` entity, repository, service, controller, or inbox UI.

**No migration needed** — pure CRUD + frontend. The backend `new-domain-module` skill scaffolds this.

## Endpoints (ARCHITECTURE §7)
```
GET    /api/quick-replies
POST   /api/quick-replies
PUT    /api/quick-replies/{id}
DELETE /api/quick-replies/{id}
```
All workspace-scoped (`findByIdAndWorkspaceId`, `findAllByWorkspaceId`).

## Non-obvious decisions
1. **Insertion, not a bypass.** Selecting a quick reply only populates the composer text; the send
   still goes through `POST /api/conversations/{id}/messages`, which enforces the **24-hour-window
   rule** (§13.2). A canned message is free text — still blocked outside the window. Quick replies
   must not become a backdoor around that rule.
2. **Shortcut optional, unique per workspace.** The partial unique index enforces it; service catches
   the conflict → `DuplicateResourceException` (409). Empty shortcut → `NULL`, not `""`.
3. **Workspace-shared, role-aware.** `marketing` has no inbox (§5); quick replies are for
   `admin`/`agent`. Admin manages; agents use. Per-agent private replies are post-MVP.
4. **Variable substitution is Phase C.** `{{name}}` expansion from contact attributes adds parsing +
   a lookup at insert time — keep MVP a literal text insert.

## Build order (per §11)
1. (No migration — table exists in V1.)
2. `QuickReply` entity (plain columns, no native enum).
3. `QuickReplyRepository`.
4. `QuickReplyService` — CRUD, workspace-scoped, shortcut-conflict handling.
5. `QuickReplyController` — the four endpoints.
6. Frontend: a settings/quick-replies management page + inbox composer integration (picker button +
   `/shortcut` autocomplete that inserts `content`).

## Phasing
- **Phase A** — CRUD backend + management UI.
- **Phase B** — inbox composer integration (`/` shortcut expansion + picker).
- **Phase C (optional)** — `{{variable}}` substitution from contact attributes.

---

# Implementation detail (deep-dive)

> Implementation-ready spec. Conventions mirror the existing codebase: controller under
> `com.watools.api.quickreply`, entity/repo/service under `com.watools.domain.quickreply`, DTO
> records with `from(entity)` factories, workspace-scoped service with a `require(id, ws)` helper.
> Matched to the repo on 2026-06-20. Still **not built**. This is the cleanest fit for the
> `new-domain-module` skill.

## A. Endpoint contracts

All require a JWT; `workspaceId` from `SecurityUtils.getCurrentWorkspaceId(auth)`. The inbox is
off-limits to `marketing` (§5) — reuse the same `requireInboxAccess(auth)` guard
`ConversationController` uses (throws `ForbiddenException` → 403). `agent` and `admin` may use them.

### `GET /api/quick-replies` → `List<QuickReplyResponse>`
```jsonc
[ { "id":"…", "title":"Greeting", "content":"Hi {{name}}, how can I help?",
    "shortcut":"hi", "createdAt":"2026-06-20T…" } ]
```
Returns all for the workspace, ordered by `title`. Not paginated (a workspace has tens, not thousands).

### `POST /api/quick-replies` → 201 `QuickReplyResponse`
```jsonc
// CreateQuickReplyRequest
{ "title":"Greeting", "content":"Hi, how can I help?", "shortcut":"hi" }   // shortcut optional
```
- `shortcut` blank/`""` → stored as `NULL` (the partial unique index only covers non-null).
- Duplicate shortcut in the workspace → `DuplicateResourceException` (409), caught from the unique
  index violation, exactly like `ContactService.create`'s phone check.

### `PUT /api/quick-replies/{id}` → `QuickReplyResponse`
`UpdateQuickReplyRequest(title, content, shortcut)` — partial update (non-null fields applied),
following the `UpdateContactRequest` pattern. `404` if not in workspace; `409` on shortcut clash.

### `DELETE /api/quick-replies/{id}` → 204
`findByIdAndWorkspaceId` then delete; `404` if not found in workspace.

## B. Runtime sequence (the §13.2 guarantee)

The key correctness property: a quick reply is **text insertion in the composer**, never a send path.
The send still goes through the existing reply endpoint, which enforces the 24h window.

```
Agent (inbox composer)                 API                                  Meta
   │ GET /api/quick-replies             │                                    │
   │───────────────────────────────────▶│ QuickReplyService.list(ws)         │
   │ ◀──────── [replies] ───────────────│                                    │
   │ types "/hi" → picks reply          │  (purely client-side text insert)  │
   │ composer now holds reply.content   │                                    │
   │ clicks Send                        │                                    │
   │ POST /api/conversations/{id}/messages (existing endpoint)               │
   │───────────────────────────────────▶│ ConversationService.sendReply(…)   │
   │                                    │  ── 24h window check (§13.2) ──     │
   │                                    │   within window? → MetaMessageClient.sendText ─▶│
   │                                    │   expired? → WindowExpiredException (require template)
   │ ◀── MessageResponse / 4xx ─────────│                                    │
```
There is **no** "send quick reply" endpoint — adding one would be the §13.2 backdoor the plan warns
against. The picker only writes to the textarea.

## C. Concrete class signatures

### `domain/quickreply/QuickReply` (entity — no native enum, plain columns)
```java
@Entity @Table(name = "quick_replies") @Getter @Setter @NoArgsConstructor
public class QuickReply {
  @Id @GeneratedValue(strategy = GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(nullable=false, length=100) private String title;
  @Column(nullable=false, columnDefinition="text") private String content;
  @Column(length=30) private String shortcut;          // nullable — NULL when unset
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;

  public QuickReply(UUID workspaceId, String title, String content) {
    this.workspaceId = workspaceId; this.title = title; this.content = content;
  }
}
```

### `domain/quickreply/QuickReplyRepository`
```java
public interface QuickReplyRepository extends JpaRepository<QuickReply, UUID> {
  List<QuickReply> findByWorkspaceIdOrderByTitleAsc(UUID workspaceId);
  Optional<QuickReply> findByIdAndWorkspaceId(UUID id, UUID workspaceId);
  boolean existsByWorkspaceIdAndShortcut(UUID workspaceId, String shortcut);   // pre-check before insert
}
```

### `domain/quickreply/QuickReplyService`
```java
@Service
public class QuickReplyService {
  @Transactional(readOnly = true) List<QuickReplyResponse> list(UUID workspaceId);
  @Transactional QuickReplyResponse create(UUID workspaceId, CreateQuickReplyRequest req);
  @Transactional QuickReplyResponse update(UUID id, UUID workspaceId, UpdateQuickReplyRequest req);
  @Transactional void delete(UUID id, UUID workspaceId);
  // helpers: require(id, ws) → ResourceNotFoundException; normalizeShortcut(s) → trimToNull (NULL not "")
  // create/update: if shortcut != null && existsByWorkspaceIdAndShortcut → DuplicateResourceException
}
```

### `api/quickreply/QuickReplyController` (thin)
```java
@RestController @RequestMapping("/api/quick-replies")
public class QuickReplyController {
  @GetMapping List<QuickReplyResponse> list(Authentication auth);
  @PostMapping @ResponseStatus(HttpStatus.CREATED)
  QuickReplyResponse create(Authentication auth, @Valid @RequestBody CreateQuickReplyRequest req);
  @PutMapping("/{id}")
  QuickReplyResponse update(Authentication auth, @PathVariable UUID id, @Valid @RequestBody UpdateQuickReplyRequest req);
  @DeleteMapping("/{id}") @ResponseStatus(HttpStatus.NO_CONTENT)
  void delete(Authentication auth, @PathVariable UUID id);
  // each first calls requireInboxAccess(auth)  (marketing → 403)
}
```

### DTO records (`api/quickreply/dto/`)
```java
public record CreateQuickReplyRequest(@NotBlank @Size(max=100) String title,
                                      @NotBlank String content,
                                      @Size(max=30) String shortcut) {}
public record UpdateQuickReplyRequest(@Size(max=100) String title, String content,
                                      @Size(max=30) String shortcut) {}
public record QuickReplyResponse(UUID id, String title, String content, String shortcut, Instant createdAt) {
  public static QuickReplyResponse from(QuickReply q) {
    return new QuickReplyResponse(q.getId(), q.getTitle(), q.getContent(), q.getShortcut(), q.getCreatedAt());
  }
}
```

## D. Tests
- `@WebMvcTest`: `marketing` role → 403 on every route; create returns 201.
- Service against Testcontainers: duplicate shortcut → 409; blank shortcut stored as `NULL` and two
  replies with blank shortcut coexist (partial index allows multiple NULLs); cross-workspace
  `get`/`delete` → 404.
