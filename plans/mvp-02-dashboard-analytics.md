# Plan: Dashboard Analytics Backend

> Status: **planned, not built.** Source of truth for schema/endpoints remains `ARCHITECTURE.md`
> (§7 Dashboard); this doc captures the design agreed on 2026-06-20.

**Why this next:** cheapest remaining win that makes the product feel finished for the Week 6 soft
launch. The frontend `dashboard-page.tsx` currently fakes its metrics from pagination totals across
4 list calls and cannot show message volume, delivery rate, read rate, or inbox load. §7 already
specifies the endpoints; they're just unbuilt.

## Endpoints (ARCHITECTURE §7)
```
GET /api/dashboard/summary?from=&to=     — aggregated metrics for a date range
GET /api/dashboard/campaigns?limit=10    — recent campaigns with stat rollups
```
Both workspace-scoped from the JWT. Default range = last 30 days in the **workspace timezone**.

## `summary` payload (all from group-by aggregates — never load rows)
- **Contacts**: total, subscribed, unsubscribed, blocked (`count … group by status`).
- **Message funnel** (outbound, in range): sent/delivered/read/failed + derived delivery & read rate.
- **Campaigns**: count by status; sent-in-range.
- **Inbox**: open conversations, unread, assigned-to-me.
- **Templates**: approved / pending (feeds the existing readiness widget).

`campaigns` already carries `sent/delivered/read/failed` counters (atomic via broadcast engine +
webhooks), so recent-campaigns is a cheap read of existing columns.

## Non-obvious decisions
1. **One aggregate endpoint replaces four list calls.** FE stops abusing pagination totals and gets
   numbers it currently can't display (delivery/read rate, message volume).
2. **Aggregate in the DB** via `@Query` count/group-by on existing repos — no entity hydration.
3. **Timezone-correct day bucketing.** Timestamps are `timestamptz` (UTC); any per-day series must
   bucket using `workspaces.timezone` or daily counts shift for IST customers.
4. **Role-scoping.** `marketing` has no inbox access (§5) — omit the inbox section for that role.
5. **Optional covering index** `messages(workspace_id, created_at)` — add only if EXPLAIN shows a
   range scan needs it; don't pre-optimize.

## Build order (per §11)
1. (No new tables.) Optional index-only migration if profiling warrants.
2. (No new entities.)
3. Repository aggregates — `@Query` count/group methods on Contact/Message/Campaign/Conversation
   repos (projection interfaces for group-by rows).
4. `DashboardService` — assembles summary, workspace-scoped, applies range + timezone.
5. `DashboardController` — two thin endpoints.
6. Frontend — replace cobbled metrics in `dashboard-page.tsx` with one `useDashboardSummary` hook;
   add a message-funnel card (optionally a daily-volume chart).
   NB: web repo `AGENTS.md` — modified Next.js, read `node_modules/next/dist/docs/` before FE edits.

## Phasing
- **Phase A** — `summary` endpoint + rewire metric cards and readiness widget.
- **Phase B** — `recent-campaigns` endpoint (replace FE client-side `.slice(0,5)`).
- **Phase C (optional)** — daily message-volume time series + chart.

## Open questions
1. Time-series chart in MVP, or headline numbers + funnel only (recommended now).
2. Date-range picker, or fixed "last 30 days" for launch (recommended).
3. Add the messages index now or wait for evidence (recommended: wait).

---

# Implementation detail (deep-dive)

> Implementation-ready spec. New code: `api/dashboard/DashboardController` +
> `domain/dashboard/DashboardService`, plus **aggregate query methods added to the existing
> repositories** (Contact/Message/Campaign/Conversation) returning Spring Data projection
> interfaces — no new entities, no row hydration. Matched to the repo on 2026-06-20. **Not built.**

## A. Endpoint contracts

Both require a JWT; workspace + role from the token. `from`/`to` are ISO dates (not date-times);
the service converts them to a UTC `Instant` half-open range `[from 00:00 tz, to+1 00:00 tz)` using
`workspaces.timezone` (default last 30 days). Role: `marketing` gets `inbox: null` (§5).

### `GET /api/dashboard/summary?from=2026-05-21&to=2026-06-20` → `DashboardSummaryResponse`
```jsonc
{
  "range": { "from":"2026-05-21", "to":"2026-06-20", "timezone":"Asia/Kolkata" },
  "contacts":  { "total":4210, "subscribed":4001, "unsubscribed":190, "blocked":19 },
  "messages":  { "sent":12044, "delivered":11600, "read":8210, "failed":444,
                 "deliveryRate":0.963, "readRate":0.682 },   // derived in Java from the counts
  "campaigns": { "draft":3, "scheduled":1, "sending":0, "sent":22, "failed":1, "sentInRange":9 },
  "inbox":     { "open":17, "unread":5, "assignedToMe":4 },   // null for marketing role
  "templates": { "approved":12, "pending":2 }
}
```
Rates are computed in the service (guard divide-by-zero → 0.0), never stored.

### `GET /api/dashboard/campaigns?limit=10` → `List<DashboardCampaignResponse>`
```jsonc
[ { "id":"…", "name":"Diwali blast", "status":"sent", "sentAt":"…",
    "totalCount":1500, "sentCount":1500, "deliveredCount":1455, "readCount":980, "failedCount":12 } ]
```
A thin projection of the existing `campaigns` counter columns (already atomic via the broadcast
engine + webhooks) ordered by `created_at desc` — effectively the existing `CampaignResponse` fields.

## B. Runtime sequence (all DB-side aggregation)

```
Browser (dashboard)              DashboardController/Service          Postgres
   │ GET /dashboard/summary?from&to │                                    │
   │────────────────────────────────▶│ resolve range → UTC Instants      │
   │                                 │   using workspaces.timezone        │
   │                                 │ contactRepo.countByStatus(ws) ─────▶│ GROUP BY status
   │                                 │ messageRepo.aggregateOutbound(ws,from,to) ─▶│ GROUP BY status (1 row)
   │                                 │ campaignRepo.countByStatus(ws) ────▶│ GROUP BY status
   │                                 │ campaignRepo.countSentInRange(…) ──▶│ COUNT
   │                                 │ if role != marketing:              │
   │                                 │   conversationRepo.inboxCounts(ws,me) ─▶│ COUNT/FILTER
   │                                 │ templateRepo.countByStatus(ws) ────▶│ GROUP BY status
   │                                 │ assemble DTO, derive rates in Java │
   │ ◀──── 200 DashboardSummary ─────│                                    │
```
Each call is a single `COUNT … GROUP BY` (≤ a handful of rows); no entities are loaded. ~6 small
aggregate queries per summary — cheap, and each already filters `workspace_id` (§13.8).

## C. Concrete signatures

### Repository additions (projection interfaces avoid hydrating entities)
```java
// domain/contact/ContactRepository (add)
@Query("SELECT c.status AS status, COUNT(c) AS count FROM Contact c WHERE c.workspaceId=:ws GROUP BY c.status")
List<StatusCount> countByStatus(@Param("ws") UUID workspaceId);
interface StatusCount { ContactStatus getStatus(); long getCount(); }   // or a shared projection

// domain/message/MessageRepository (add) — one row of outbound funnel counts in range
@Query("""
   SELECT
     SUM(CASE WHEN m.status IS NOT NULL THEN 1 ELSE 0 END) AS sent,
     SUM(CASE WHEN m.status = com.watools.domain.message.MessageStatus.delivered THEN 1 ELSE 0 END) AS delivered,
     SUM(CASE WHEN m.status = com.watools.domain.message.MessageStatus.read      THEN 1 ELSE 0 END) AS read,
     SUM(CASE WHEN m.status = com.watools.domain.message.MessageStatus.failed    THEN 1 ELSE 0 END) AS failed
   FROM Message m
   WHERE m.workspaceId=:ws AND m.direction=com.watools.domain.message.MessageDirection.outbound
     AND m.createdAt >= :from AND m.createdAt < :to""")
MessageFunnel aggregateOutbound(@Param("ws") UUID ws, @Param("from") Instant from, @Param("to") Instant to);
interface MessageFunnel { long getSent(); long getDelivered(); long getRead(); long getFailed(); }

// domain/campaign/CampaignRepository (add)
@Query("SELECT c.status AS status, COUNT(c) AS count FROM Campaign c WHERE c.workspaceId=:ws GROUP BY c.status")
List<CampaignStatusCount> countByStatus(@Param("ws") UUID ws);
long countByWorkspaceIdAndSentAtBetween(UUID ws, Instant from, Instant to);   // sentInRange

// domain/conversation/ConversationRepository (add) — counts, no rows
long countByWorkspaceIdAndStatus(UUID ws, ConversationStatus status);                 // open
long countByWorkspaceIdAndStatusAndIsReadFalse(UUID ws, ConversationStatus status);   // unread
long countByWorkspaceIdAndAssignedToAndStatus(UUID ws, UUID me, ConversationStatus status);

// domain/template/TemplateRepository (add)
@Query("SELECT t.status AS status, COUNT(t) AS count FROM Template t WHERE t.workspaceId=:ws GROUP BY t.status")
List<TemplateStatusCount> countByStatus(@Param("ws") UUID ws);
```

### `domain/dashboard/DashboardService`
```java
@Service
public class DashboardService {
  // injects the existing repos (Contact/Message/Campaign/Conversation/Template) + WorkspaceRepository (timezone)
  @Transactional(readOnly = true)
  DashboardSummaryResponse summary(UUID workspaceId, UUID userId, String role, LocalDate from, LocalDate to);
  @Transactional(readOnly = true)
  List<DashboardCampaignResponse> recentCampaigns(UUID workspaceId, int limit);

  // private Range resolveRange(UUID ws, LocalDate from, LocalDate to)  — applies workspaces.timezone,
  //   defaults to last 30 days, returns half-open [fromInstant, toInstant)
}
```

### `api/dashboard/DashboardController` (thin)
```java
@RestController @RequestMapping("/api/dashboard")
public class DashboardController {
  @GetMapping("/summary")
  DashboardSummaryResponse summary(Authentication auth,
      @RequestParam(required=false) @DateTimeFormat(iso=DATE) LocalDate from,
      @RequestParam(required=false) @DateTimeFormat(iso=DATE) LocalDate to) {
    return dashboardService.summary(getCurrentWorkspaceId(auth), getCurrentUserId(auth), getCurrentRole(auth), from, to);
  }
  @GetMapping("/campaigns")
  List<DashboardCampaignResponse> campaigns(Authentication auth, @RequestParam(defaultValue="10") int limit) {
    return dashboardService.recentCampaigns(getCurrentWorkspaceId(auth), Math.min(limit, 50));
  }
}
```

### DTO records (`api/dashboard/dto/`)
```java
public record DashboardSummaryResponse(RangeInfo range, ContactStats contacts, MessageStats messages,
                                       CampaignStats campaigns, InboxStats inbox, TemplateStats templates) {
  public record RangeInfo(LocalDate from, LocalDate to, String timezone) {}
  public record ContactStats(long total, long subscribed, long unsubscribed, long blocked) {}
  public record MessageStats(long sent, long delivered, long read, long failed, double deliveryRate, double readRate) {}
  public record CampaignStats(long draft, long scheduled, long sending, long sent, long failed, long sentInRange) {}
  public record InboxStats(long open, long unread, long assignedToMe) {}   // field is null for marketing
  public record TemplateStats(long approved, long pending) {}
}
public record DashboardCampaignResponse(UUID id, String name, CampaignStatus status, Instant sentAt,
    int totalCount, int sentCount, int deliveredCount, int readCount, int failedCount) {}
```

## D. Tests
- Service against Testcontainers seeded with mixed statuses: counts correct; **IST day-boundary** —
  a message at `2026-06-20T18:30:00Z` (00:00 IST 06-21) falls in the 06-21 bucket, not 06-20.
- `marketing` role → `inbox == null`; rates are 0.0 when `sent == 0` (no divide-by-zero).
- `@WebMvcTest`: default range applied when `from`/`to` omitted.
