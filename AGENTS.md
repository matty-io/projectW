# WhatsApp Marketing Tool — Shared Context

A WhatsApp marketing SaaS (WhatsApp Business API broadcasts, shared team inbox,
Meta-approved templates, contacts/lists, campaign analytics). Competes with Wati / AiSensy.

> **The full specification lives in `@ARCHITECTURE.md`.** It is the source of truth for
> schema, endpoints, and tech decisions. Before implementing any module, read the relevant
> section of that file — do not work from memory.

This file is the **shared brain** for both repositories. It is inherited automatically by
Codex sessions in either sub-repo (AGENTS.md is read from the cwd and every parent dir).
Repo-specific build commands, patterns, and tooling live in each repo's own `AGENTS.md`.

## Repository map

```
projectW/
├── ARCHITECTURE.md        ← full spec (read before implementing)
├── AGENTS.md              ← this file (shared rules)
├── whatsapp-tool-api/     ← Spring Boot 3 / Java 21 backend  (current focus)
└── whatsapp-tool-web/     ← Next.js 14 frontend             (next phase)
```

## Non-negotiable business rules (ARCHITECTURE §13) — enforce in every change

1. **Unsubscribed contacts are NEVER sent campaign messages** — filter in `CampaignService` before enqueueing.
2. **Free-text replies only within 24h of the last customer message** — check `conversation.last_customer_message_at`; otherwise require an approved template.
3. **Only `APPROVED` templates may be used in campaigns** — validate in `CampaignService`.
4. **workspace ↔ WABA is strictly 1:1** — DB `UNIQUE` constraint + validation in `WaAccountService`.
5. **`wa_message_id` deduplication** — always check before processing any inbound webhook message (Meta sends duplicates).
6. **Return 200 to the Meta webhook within 5 seconds** — verify signature, then always process asynchronously.
7. **Verify `X-Hub-Signature-256` on every `POST /webhook`** — reject with 403 if invalid.
8. **Contacts (and all data) are isolated per workspace** — every query includes a `workspace_id` filter.

## Schema documentation rule

- When you change the database schema, update [`schema.dbml`](/Users/matty/Work/projectW/schema.dbml) in the same change set.

## Multi-tenancy model (applies everywhere)

- Every domain row (except junction tables) carries `workspace_id`.
- The JWT carries `userId`, `workspaceId` (current active workspace), `organizationId`, `role`.
- The **service layer** validates that every requested resource belongs to the caller's `workspaceId`.
- Users may belong to multiple workspaces and switch between them.
- Roles: `admin` (full), `agent` (assigned/unassigned conversations, no campaigns), `marketing` (campaigns + templates, no inbox).

## Per-feature build order (ARCHITECTURE §11)

For each feature, build in this exact sequence — each step depends on the previous:

1. Flyway migration SQL (only if new tables are needed)
2. JPA entity classes
3. Spring Data repository interfaces
4. Service class (business logic — this is where workspace scoping + rules live)
5. REST controller (thin: validate, call service, return DTO)
6. Next.js page + API calls (frontend phase)

## Phasing

Backend first (weeks 1–4: setup → contacts/templates → campaigns → inbox), then frontend.
Do **not** build the Post-MVP features listed in ARCHITECTURE §12 (chatbot, Shopify, AI
suggestions, Redis/WebSockets, white-label, etc.) during the MVP weeks.
