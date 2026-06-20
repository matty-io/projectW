# Plan: Contact CSV Import (async, S3-backed)

> Status: **planned, not built.** Source of truth for schema/endpoints remains `ARCHITECTURE.md`
> (§7 Contacts, §8 Contact Import pattern); this doc captures the design agreed on 2026-06-20.

**Why this next:** the one hard **beta-blocker** of the remaining gaps — beta customers arrive with
spreadsheets and won't add contacts one at a time. Also builds the shared `S3Service` that
template-media and contact-export need later. The S3 starter is already on the classpath and
LocalStack S3 is wired; `ContactController` notes import is "deferred until the S3 infra lands."

## Flow (ARCHITECTURE §8, direct-to-S3)
1. `POST /api/contacts/import/upload-url` → presigned S3 PUT URL + object key.
2. Browser uploads the CSV **straight to S3** (no large multipart through Spring/Tomcat).
3. `POST /api/contacts/import` `{ s3Key, filename, columnMapping, targetListId? }` → creates job
   (`status=pending`), enqueues async processing, returns `jobId`.
4. `@Async("importExecutor")` streams from S3, parses, validates, **upserts in batches of 100**,
   tracks counts, writes an error-report CSV back to S3.
5. `GET /api/contacts/import/{jobId}` → poll status + counts;
   `GET /api/contacts/import/{jobId}/errors` → download failed rows.

## Non-obvious decisions
1. **Direct-to-S3 presigned upload**, not multipart-through-the-API — keeps big files off request
   threads and the Tomcat upload limit. (Multipart fallback exists if we cut scope.)
2. **Separate `importExecutor` pool** in `AsyncConfig`, NOT the shared `webhookExecutor` — a 50k-row
   import must never starve the 5-second Meta webhook SLA (rule 6).
3. **`INSERT … ON CONFLICT (workspace_id, phone) DO UPDATE`** native upsert (unique index exists).
   Fast for tens of thousands of rows AND idempotent — crash mid-file → re-run is duplicate-safe.
   New rows get `source = imported`.
4. **Never resurrect an unsubscribed contact.** On conflict, update name/attributes but DO NOT flip
   `status` back to `subscribed` (would violate §13.1 downstream). Status is insert-only.
5. **Error report CSV back to S3**, downloadable — a customer with 200 bad rows in 5,000 can fix and
   re-upload instead of suffering a silent partial import.

## Schema — new migration `contact_imports`
```sql
CREATE TYPE import_status AS ENUM ('pending','processing','completed','failed');

CREATE TABLE contact_imports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id UUID NOT NULL REFERENCES workspaces(id),
  created_by UUID NOT NULL REFERENCES users(id),
  s3_key TEXT NOT NULL,
  original_filename VARCHAR(255),
  status import_status NOT NULL DEFAULT 'pending',
  total_rows INT DEFAULT 0,
  imported_count INT DEFAULT 0,
  updated_count INT DEFAULT 0,
  skipped_count INT DEFAULT 0,
  failed_count INT DEFAULT 0,
  column_mapping JSONB DEFAULT '{}',     -- CSV header → name/email/phone/attribute key
  target_list_id UUID REFERENCES lists(id),
  error_report_s3_key TEXT,
  error_message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);
CREATE INDEX contact_imports_ws_idx ON contact_imports(workspace_id);
```
Mapping auto-detects standard headers (`name`,`phone`,`email`); unknown columns fold into
`contacts.attributes` (jsonb).

## Build order (per §11)
1. Migration `contact_imports` (+ `import_status` enum).
2. `ContactImport` entity + `ImportStatus` enum (native-enum mapping).
3. `ContactImportRepository` (`findByIdAndWorkspaceId`).
4. **`infrastructure/s3/S3Service`** — presign PUT, presign/stream GET, putObject. New shared infra.
5. `ContactImportService` — `createUploadUrl`, `startImport`, `@Async processImport`, `getStatus`.
   Reuses `ContactService` phone-normalization; batched upserts.
6. `ContactImportController` — endpoints above (or extend `ContactController`).
7. Next.js: contacts-page **Import** button → upload dialog (presign → PUT → start) → progress
   poller → result summary with "download failed rows" link.

## Dependency note
Needs a CSV parser (Apache Commons CSV recommended — tiny) — new `pom.xml` dependency, flag for approval.

## Phasing
- **Phase A — `S3Service`** (presign + stream). Shared; also unblocks export + template media.
- **Phase B — Import job** (schema → entity → service → async → controller → polling).
- **Phase C — Frontend UI + error-report download.** Cheap to add contact export
  (`GET /api/contacts/export`) here since it reuses `S3Service`.

## Open questions
1. Column mapping: auto-detect standard headers only (recommended) vs full mapping UI day one.
2. Row cap per import for MVP (e.g. 50k; later plan-based).
3. CSV library: Commons CSV (recommended) vs univocity.
4. Bundle contact export into Phase C, or import-only.

---

# Implementation detail (deep-dive)

> This section takes the plan above to implementation-ready depth: exact endpoint contracts,
> the runtime sequence, and concrete class signatures. Package layout follows the real tree —
> controllers/DTOs under `com.watools.api.contact`, entity/repo/service under
> `com.watools.domain.contact`, shared infra under `com.watools.infrastructure.s3`. Still
> **not built** — this is the spec to build against. Verify class/method names against the repo
> at build time; they were matched to the codebase on 2026-06-20.

## A. Endpoint contracts

All four require a valid JWT; `workspaceId` is resolved from the token via
`SecurityUtils.getCurrentWorkspaceId(auth)` (never trusted from the body). Roles: `admin` and
`marketing` may import; `agent` may not (enforce in the service, consistent with §5 role rules).

### 1. `POST /api/contacts/import/upload-url` → presigned PUT
Mints a short-lived S3 PUT URL the browser uploads straight to. **New endpoint** beyond the
ARCHITECTURE §7 list (the spec assumed multipart-through-API; we chose direct-to-S3).

```jsonc
// Request — CreateUploadUrlRequest
{ "filename": "contacts-jan.csv", "contentType": "text/csv" }

// 200 — UploadUrlResponse
{
  "uploadUrl": "https://<bucket>.s3.ap-south-1.amazonaws.com/imports/<wsId>/<uuid>-contacts-jan.csv?X-Amz-…",
  "s3Key": "imports/<wsId>/<uuid>-contacts-jan.csv",
  "expiresAt": "2026-06-20T10:25:00Z"
}
```
- The server, not the client, builds the key: `imports/{workspaceId}/{randomUuid}-{sanitizedFilename}`.
  The workspace prefix is the isolation boundary (§13.8) and is re-validated in step 3.
- Presign TTL ~10 min. `contentType` is pinned into the signature so the PUT must match.

### 2. `POST /api/contacts/import` → create + enqueue job
```jsonc
// Request — StartImportRequest
{
  "s3Key": "imports/<wsId>/<uuid>-contacts-jan.csv",   // from step 1
  "filename": "contacts-jan.csv",
  "columnMapping": { "Phone": "phone", "Full Name": "name", "City": "attributes.city" },
  "targetListId": "1f2e…"                                // optional
}

// 202 Accepted — ImportJobResponse (status=pending)
{
  "id": "9a7c…", "status": "pending",
  "totalRows": 0, "importedCount": 0, "updatedCount": 0, "skippedCount": 0, "failedCount": 0,
  "originalFilename": "contacts-jan.csv", "targetListId": "1f2e…",
  "hasErrorReport": false,
  "createdAt": "2026-06-20T10:16:00Z", "completedAt": null
}
```
Validation before enqueue (synchronous, fail fast with `400`):
- `s3Key` **must** start with `imports/{callerWorkspaceId}/` — rejects forging another tenant's key.
- `columnMapping` values must resolve to `phone` | `name` | `email` | `attributes.<key>`; `phone` is required.
- `targetListId` (if present) must belong to the workspace (`listRepository.findByIdAndWorkspaceId`).
- Returns `404`/`ResourceNotFoundException` if the S3 object is missing (HEAD check).

### 3. `GET /api/contacts/import/{jobId}` → poll status
Returns the same `ImportJobResponse`, now with live counts and `status` in
`pending|processing|completed|failed`. Frontend polls every ~2 s while `pending|processing`.
`404` if the job isn't in the caller's workspace (`findByIdAndWorkspaceId`).

### 4. `GET /api/contacts/import/{jobId}/errors` → error-report download
```jsonc
// 200 — ErrorReportResponse  (404 if job has no error report)
{ "downloadUrl": "https://…s3…/imports/<wsId>/<uuid>-errors.csv?X-Amz-…", "expiresAt": "…" }
```
A presigned **GET** URL (TTL ~5 min) rather than streaming bytes through the API — same rationale
as the upload. The error CSV is the original rows that failed plus a trailing `error` column.

**Error envelope** for all four reuses the app's existing handler
(`DuplicateResourceException`→409, `ResourceNotFoundException`→404, bean-validation→400).

## B. Runtime sequence

```
Browser (contacts page)        API (ContactImportController/Service)        S3            importExecutor pool
      │                                   │                                  │                    │
 1.   │ POST /import/upload-url           │                                  │                    │
      │──────────────────────────────────▶│ S3Service.presignPut(key,ttl)   │                    │
      │                                   │─────────────────────────────────▶│                    │
      │ ◀───────── 200 {uploadUrl,s3Key} ─│◀──── presigned URL ─────────────│                    │
      │                                   │                                  │                    │
 2.   │ PUT csv bytes (direct to S3)      │                                  │                    │
      │──────────────────────────────────┼─────────────────────────────────▶│  (object stored)   │
      │ ◀──────────────── 200 ────────────┼──────────────────────────────────│                    │
      │                                   │                                  │                    │
 3.   │ POST /import {s3Key,mapping,list} │                                  │                    │
      │──────────────────────────────────▶│ validate key prefix + list + HEAD object             │
      │                                   │ INSERT contact_imports (pending) │                    │
      │                                   │ runner.run(jobId, wsId) ─────────┼───────────────────▶│  @Async
      │ ◀──────── 202 {jobId,pending} ────│                                  │                    │ status=processing
      │                                   │                                  │  getObjectStream    │
 4.   │                                   │                                  │◀───────────────────│ stream+parse CSV
      │                                   │                                  │                    │ for each batch(100):
      │                                   │                                  │                    │   upsert ON CONFLICT
      │                                   │                                  │                    │   (+ list link)
      │                                   │                                  │                    │   accumulate counts
      │                                   │                                  │  putObject(errors)  │ write error CSV
      │                                   │                                  │◀───────────────────│ (if any failed)
      │                                   │                                  │                    │ status=completed
 5.   │ GET /import/{jobId}  (poll ~2s)   │                                  │                    │
      │──────────────────────────────────▶│ findByIdAndWorkspaceId → counts │                    │
      │ ◀──── 200 {processing→completed} ─│                                  │                    │
      │                                   │                                  │                    │
 6.   │ GET /import/{jobId}/errors        │ S3Service.presignGet(errKey)     │                    │
      │──────────────────────────────────▶│─────────────────────────────────▶│                    │
      │ ◀──────── 200 {downloadUrl} ──────│◀─────────────────────────────────│                    │
```

Key point at step 3→4: the controller returns **202 immediately** after persisting the `pending`
row; all S3 reading and DB writing happen on the `importExecutor` thread, never on the request
thread (and never on `webhookExecutor` — protecting the §13.6 5-second webhook SLA).

## C. Concrete class signatures

### `infrastructure/s3/S3Service` (new shared infra — Phase A)
```java
package com.watools.infrastructure.s3;

public interface S3Service {
  /** Presigned PUT for direct browser upload. The key is caller-built and tenant-prefixed. */
  PresignedUpload presignPut(String key, String contentType, Duration ttl);

  /** Presigned GET for direct browser download (error report, future export). */
  PresignedDownload presignGet(String key, Duration ttl);

  /** Streaming read for the import job — never buffer a 50k-row file fully in memory. */
  InputStream openStream(String key);

  /** Small server-generated objects (the error-report CSV). */
  void putBytes(String key, byte[] body, String contentType);

  boolean exists(String key);   // HEAD — validate before enqueueing a job

  record PresignedUpload(URL url, String key, Instant expiresAt) {}
  record PresignedDownload(URL url, Instant expiresAt) {}
}
```
Impl `S3ServiceImpl` uses the AWS SDK v2 `S3Presigner` (for presign — not exposed by Spring Cloud
AWS `S3Template`) plus `S3Client`/`S3Template` for `openStream`/`putBytes`/`exists`. Bucket from
`aws.s3.bucket`; LocalStack endpoint already wired for the `local` profile.

### `config/AsyncConfig` — add the dedicated pool
```java
@Bean(name = "importExecutor")
public Executor importExecutor() {                      // sized small — imports are heavy & rare
  ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
  e.setCorePoolSize(importCoreSize);                    // thread-pool.import.core-size:2
  e.setMaxPoolSize(importMaxSize);                      // thread-pool.import.max-size:4
  e.setQueueCapacity(importQueueCapacity);              // thread-pool.import.queue-capacity:50
  e.setThreadNamePrefix("import-");
  e.initialize();
  return e;
}
```

### `domain/contact/ContactImport` (entity) + `ImportStatus`
```java
@Entity @Table(name = "contact_imports") @Getter @Setter @NoArgsConstructor
public class ContactImport {
  @Id @GeneratedValue(strategy = GenerationType.UUID) private UUID id;
  @Column(name="workspace_id", nullable=false) private UUID workspaceId;
  @Column(name="created_by", nullable=false)   private UUID createdBy;
  @Column(name="s3_key", nullable=false)        private String s3Key;
  @Column(name="original_filename")             private String originalFilename;

  @JdbcTypeCode(SqlTypes.NAMED_ENUM)
  @Column(nullable=false, columnDefinition="import_status")
  private ImportStatus status = ImportStatus.pending;   // labels lower-case, match the SQL enum

  @Column(name="total_rows")     private int totalRows;
  @Column(name="imported_count") private int importedCount;
  @Column(name="updated_count")  private int updatedCount;
  @Column(name="skipped_count")  private int skippedCount;
  @Column(name="failed_count")   private int failedCount;

  @JdbcTypeCode(SqlTypes.JSON) @Column(name="column_mapping", columnDefinition="jsonb")
  private Map<String,String> columnMapping = new HashMap<>();

  @Column(name="target_list_id")        private UUID targetListId;
  @Column(name="error_report_s3_key")   private String errorReportS3Key;
  @Column(name="error_message")         private String errorMessage;
  @CreationTimestamp @Column(name="created_at", nullable=false, updatable=false) private Instant createdAt;
  @Column(name="completed_at")          private Instant completedAt;
}

public enum ImportStatus { pending, processing, completed, failed }   // lower-case == SQL labels
```

### `domain/contact/ContactImportRepository`
```java
public interface ContactImportRepository extends JpaRepository<ContactImport, UUID> {
  Optional<ContactImport> findByIdAndWorkspaceId(UUID id, UUID workspaceId);
}
```
> Note: job counters are written **only** by the single import thread that owns the job, so plain
> setter + `save` is correct here — this is *not* the campaign-counter case (§8) where the SQS
> consumer and webhook handler race and atomic `@Modifying` increments are mandatory.

### `domain/contact/ContactUpsertRepository` (JdbcTemplate — the batch upsert)
JPA can't express `ON CONFLICT`; use a thin JdbcTemplate component.
```java
@Repository
public class ContactUpsertRepository {
  /** Batch upsert. Returns per-row outcome so the service can tally imported vs updated vs skipped. */
  public List<UpsertOutcome> upsertBatch(UUID workspaceId, List<ParsedContactRow> rows);
  // SQL per row (batched):
  //   INSERT INTO contacts (id, workspace_id, phone, name, email, source, attributes)
  //   VALUES (gen_random_uuid(), :ws, :phone, :name, :email, 'imported', :attrs::jsonb)
  //   ON CONFLICT (workspace_id, phone) DO UPDATE
  //     SET name       = COALESCE(EXCLUDED.name, contacts.name),
  //         email      = COALESCE(EXCLUDED.email, contacts.email),
  //         attributes = contacts.attributes || EXCLUDED.attributes   -- jsonb merge, keep existing keys
  //   RETURNING (xmax = 0) AS inserted;   -- xmax=0 ⇒ INSERT, else UPDATE
  // NB: status is intentionally NOT in the UPDATE set → never resurrects an unsubscribed contact (§13.1).
  enum UpsertOutcome { INSERTED, UPDATED }
}
```

### `domain/contact/ContactImportService` (orchestration) + `ContactImportJobRunner` (async)
Split because Spring `@Async` does **not** apply to self-invocation — the async method must live on
a different bean so the call goes through the proxy.
```java
@Service
public class ContactImportService {
  UploadUrlResponse createUploadUrl(UUID workspaceId, UUID userId, CreateUploadUrlRequest req);
  ImportJobResponse  startImport   (UUID workspaceId, UUID userId, StartImportRequest req);   // validates, persists pending, calls runner
  ImportJobResponse  getStatus     (UUID jobId, UUID workspaceId);
  ErrorReportResponse errorReport  (UUID jobId, UUID workspaceId);
}

@Component
public class ContactImportJobRunner {
  @Async("importExecutor")
  void run(UUID jobId, UUID workspaceId);   // status=processing → stream → parse → batch upsert
                                            // → list-link → counts → error CSV → completed/failed
}
```
`run` reuses `ContactService` normalization rules (`trimToNull`, `normalizeEmail`, `phone.trim()`)
on each `ParsedContactRow`; rows that fail validation (blank/oversize phone, bad email) are counted
as `failed` and appended to the error CSV rather than aborting the whole import. A row whose only
change would be resurrecting an `unsubscribed` contact is counted as `skipped`.

### `api/contact/ContactImportController` (thin)
```java
@RestController @RequestMapping("/api/contacts/import")
public class ContactImportController {
  @PostMapping("/upload-url")
  UploadUrlResponse uploadUrl(Authentication auth, @Valid @RequestBody CreateUploadUrlRequest req);

  @PostMapping @ResponseStatus(HttpStatus.ACCEPTED)
  ImportJobResponse start(Authentication auth, @Valid @RequestBody StartImportRequest req);

  @GetMapping("/{jobId}")
  ImportJobResponse status(Authentication auth, @PathVariable UUID jobId);

  @GetMapping("/{jobId}/errors")
  ErrorReportResponse errors(Authentication auth, @PathVariable UUID jobId);
}
```
> Could also fold onto the existing `ContactController` (it already comments that import is
> "deferred until the S3 infra lands"). A separate controller keeps that class focused; either is fine.

### DTO records (`api/contact/dto/`)
```java
public record CreateUploadUrlRequest(@NotBlank @Size(max=255) String filename,
                                     @Size(max=100) String contentType) {}
public record UploadUrlResponse(String uploadUrl, String s3Key, Instant expiresAt) {}

public record StartImportRequest(@NotBlank String s3Key,
                                 @Size(max=255) String filename,
                                 Map<String,String> columnMapping,
                                 UUID targetListId) {}

public record ImportJobResponse(UUID id, ImportStatus status,
                                int totalRows, int importedCount, int updatedCount,
                                int skippedCount, int failedCount,
                                String originalFilename, UUID targetListId,
                                boolean hasErrorReport, Instant createdAt, Instant completedAt) {
  public static ImportJobResponse from(ContactImport j) { /* …mirror of ContactResponse.from */ }
}
public record ErrorReportResponse(String downloadUrl, Instant expiresAt) {}
```

## D. Build order for the deep-dive (refines §11 above)
1. Migration `V3__contact_imports.sql` (enum + table from the schema block above).
2. `S3Service` + `S3ServiceImpl` + `importExecutor` bean — **Phase A, independently shippable/testable.**
3. `ContactImport` entity, `ImportStatus`, `ContactImportRepository`, `ContactUpsertRepository`.
4. `ContactImportService` + `ContactImportJobRunner` (+ DTOs).
5. `ContactImportController`.
6. Frontend import dialog (presign → PUT → start → poll → summary + "download failed rows").

## E. Test plan (matches the repo's Testcontainers approach)
- `ContactUpsertRepository` against Testcontainers Postgres: insert-then-conflict returns
  `INSERTED` then `UPDATED`; **unsubscribed contact stays unsubscribed after re-import** (§13.1 guard);
  jsonb `attributes` merge keeps pre-existing keys.
- `ContactImportService`: forged `s3Key` (wrong workspace prefix) → 400; missing object → 404;
  cross-workspace `getStatus` → 404.
- `S3ServiceImpl` against LocalStack: presign round-trips a PUT then `openStream` reads it back.
- Controller `@WebMvcTest`: agent role rejected; 202 on start.
