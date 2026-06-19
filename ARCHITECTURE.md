# WhatsApp Marketing Tool — Architecture & Requirements
> Complete reference document for implementation. Share this with Claude Code at the start of every session.

---

## 1. Product Overview

A WhatsApp marketing SaaS tool that competes with Wati and AiSensy. Allows businesses to:
- Send bulk broadcast campaigns via WhatsApp Business API
- Manage a shared team inbox for customer conversations
- Create and manage Meta-approved message templates
- Manage contacts, lists, and tags
- Track campaign delivery, read, and reply analytics

**Target customers:** D2C brands, coaching businesses, clinics, real estate agents — any business already using WhatsApp manually to sell or support.

**Pricing:**
- Starter: ₹999/month — 3 agents, 5,000 messages/month
- Pro: ₹2,499/month — 10 agents, unlimited messages
- Agency: ₹5,999/month — multi-workspace, white-label

---

## 2. Tech Stack

### Backend
| Layer | Technology | Notes |
|---|---|---|
| Framework | Spring Boot 3 | Java 21, virtual threads enabled |
| Security | Spring Security + JWT | Stateless, workspace-scoped tokens |
| Database | Spring Data JPA + Hibernate | Flyway for migrations |
| Queue | AWS SQS via Spring Cloud AWS 3.1 | Broadcast fan-out engine |
| Storage | AWS S3 | CSV uploads, template media |
| Realtime | SSE via SseEmitter | Live inbox updates |
| Scheduler | Spring @Scheduled | Campaign scheduling, polling |
| HTTP Client | RestClient (Spring 6) | Meta Cloud API calls |
| Email | AWS SES | Team invites, billing receipts |

### Frontend
| Layer | Technology | Notes |
|---|---|---|
| Framework | Next.js 16 | App Router, Turbopack, React 19 |
| UI | Tailwind CSS v4 + shadcn/ui | OKLCH semantic theme tokens; swappable `data-theme` presets |
| State | TanStack Query (React Query) | Server state, caching |
| Auth | httpOnly cookie with JWT | |
| Realtime | EventSource (native) | SSE connection for inbox |
| Deploy | Vercel | |

### Infrastructure
| Service | Tool | Notes |
|---|---|---|
| Database | AWS RDS PostgreSQL 15 | db.t3.micro to start |
| Queue | AWS SQS | Standard queue, not FIFO |
| Storage | AWS S3 | ap-south-1 region |
| Backend deploy | AWS ECS Fargate | Docker container |
| Frontend deploy | Vercel | Auto-deploy on push |
| Payments | Razorpay | Subscriptions |
| Error tracking | Sentry | Both frontend and backend |
| WhatsApp | Meta Cloud API | Direct, no BSP middleman |

### Local Development
| Service | Tool |
|---|---|
| Database | Docker — postgres:15-alpine |
| SQS + S3 | Docker — LocalStack |
| Email | Docker — Mailpit (localhost:8025) |
| Meta webhooks | ngrok → localhost:8080 |

---

## 3. Repository Structure

Two separate repositories:

```
whatsapp-tool-api/          ← Spring Boot backend
whatsapp-tool-web/          ← Next.js frontend
```

### Backend Package Structure
```
src/main/java/com/watools/
├── config/
│   ├── SecurityConfig.java
│   ├── AsyncConfig.java         — thread pool for @Async webhook processing
│   ├── SqsConfig.java           — LocalStack override for local profile
│   └── S3Config.java            — LocalStack override for local profile
├── domain/
│   ├── organization/
│   │   ├── Organization.java
│   │   ├── OrganizationRepository.java
│   │   └── OrganizationService.java
│   ├── workspace/
│   │   ├── Workspace.java
│   │   ├── WorkspaceMember.java
│   │   ├── WorkspaceRepository.java
│   │   ├── WorkspaceMemberRepository.java
│   │   └── WorkspaceService.java
│   ├── user/
│   │   ├── User.java
│   │   ├── UserRepository.java
│   │   └── UserService.java
│   ├── wa/                      — WhatsApp connection
│   │   ├── WaBusinessAccount.java
│   │   ├── WaPhoneNumber.java
│   │   ├── WaBusinessAccountRepository.java
│   │   ├── WaPhoneNumberRepository.java
│   │   └── WaAccountService.java
│   ├── contact/
│   │   ├── Contact.java
│   │   ├── Tag.java
│   │   ├── ContactTag.java
│   │   ├── ContactList.java
│   │   ├── ListContact.java
│   │   ├── ContactRepository.java
│   │   └── ContactService.java
│   ├── template/
│   │   ├── Template.java
│   │   ├── TemplateRepository.java
│   │   └── TemplateService.java
│   ├── campaign/
│   │   ├── Campaign.java
│   │   ├── CampaignContact.java
│   │   ├── CampaignRepository.java
│   │   ├── CampaignContactRepository.java
│   │   └── CampaignService.java
│   ├── conversation/
│   │   ├── Conversation.java
│   │   ├── ConversationNote.java
│   │   ├── ConversationRepository.java
│   │   └── ConversationService.java
│   └── message/
│       ├── Message.java
│       ├── QuickReply.java
│       ├── MessageRepository.java
│       └── MessageService.java
├── api/
│   ├── auth/
│   │   └── AuthController.java
│   ├── workspace/
│   │   └── WorkspaceController.java
│   ├── contact/
│   │   └── ContactController.java
│   ├── template/
│   │   └── TemplateController.java
│   ├── campaign/
│   │   └── CampaignController.java
│   ├── conversation/
│   │   └── ConversationController.java
│   ├── message/
│   │   └── MessageController.java
│   └── webhook/
│       └── WhatsAppWebhookController.java
├── integration/
│   └── meta/
│       ├── MetaApiClient.java       — all calls to graph.facebook.com
│       └── MetaWebhookProcessor.java — @Async handler
├── infrastructure/
│   ├── sqs/
│   │   ├── BroadcastProducer.java
│   │   └── BroadcastConsumer.java
│   ├── s3/
│   │   └── S3Service.java
│   └── sse/
│       └── SseService.java
└── WaToolsApplication.java
```

### Frontend Structure
```
app/
├── (auth)/
│   ├── login/page.tsx
│   └── signup/page.tsx
├── (dashboard)/
│   ├── layout.tsx               — sidebar + workspace switcher
│   ├── dashboard/page.tsx
│   ├── contacts/page.tsx
│   ├── templates/page.tsx
│   ├── campaigns/
│   │   ├── page.tsx
│   │   └── [id]/page.tsx        — live campaign stats
│   ├── inbox/page.tsx           — SSE-powered live inbox
│   └── settings/
│       ├── page.tsx
│       └── connect/page.tsx     — WhatsApp number connection flow
components/
├── ui/                          — shadcn/ui (auto-generated, do not edit)
├── contacts/
├── campaigns/
└── inbox/
lib/
├── api.ts                       — fetch wrapper with JWT header
├── hooks/
│   ├── useSSE.ts                — SSE connection hook for inbox
│   └── useWorkspace.ts          — current workspace context
proxy.ts                         — route gate, redirect unauthenticated to /login
                                   (Next.js 16 renamed `middleware.ts` → `proxy.ts`)
app/bff/[...path]/route.ts        — BFF proxy: forwards client calls to the API with the
                                   JWT from the httpOnly cookie (token never reaches the browser)
```

---

## 4. Database Schema

### Design Decisions
- All tables use `uuid` primary keys with `gen_random_uuid()` default
- All timestamps use `timestamptz`
- Every table (except junction tables) has `workspace_id` for multi-tenancy
- All queries MUST filter by `workspace_id` — enforced at service layer
- `attributes jsonb` on contacts for flexible custom fields
- `components jsonb` on templates — stored as Meta's native format
- `content jsonb` on messages — flexible for text/image/template types
- `workspace_id` is denormalized on `messages` for fast inbox queries

### Entity Hierarchy
```
organizations (SaaS billing entity)
    └── users (members of the org)
    └── workspaces (one per brand — like AiSensy's "Project")
            └── workspace_members (users ↔ workspaces with roles)
            └── wa_business_accounts (STRICT 1:1 with workspace)
                    └── wa_phone_numbers (1:many under the WABA)
            └── contacts, tags, lists
            └── templates (scoped to WABA level)
            └── conversations (one per contact per phone number)
                    └── messages (individual messages)
                    └── conversation_notes (internal agent notes)
            └── campaigns
                    └── campaign_contacts (one row per contact per campaign)
            └── quick_replies
```

### Key Relationship Rules
- `workspace ↔ wa_business_account` is **strictly 1:1** enforced by UNIQUE constraint
- `wa_business_account → wa_phone_numbers` is **1:many**
- `conversations` are unique per `(phone_number_id, contact_id)` — one conversation per contact per number
- `templates` belong to a WABA (not phone number) — all numbers under the WABA share templates
- `campaigns` and `conversations` always reference `phone_number_id` — messages flow through specific numbers

### Complete SQL Schema (Flyway V1__init_schema.sql)

```sql
-- EXTENSIONS
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ENUMS
CREATE TYPE plan_type AS ENUM ('starter', 'pro', 'agency');
CREATE TYPE workspace_role AS ENUM ('admin', 'agent', 'marketing');
CREATE TYPE contact_status AS ENUM ('subscribed', 'unsubscribed', 'blocked');
CREATE TYPE contact_source AS ENUM ('imported', 'inbound', 'bot', 'manual');
CREATE TYPE quality_rating AS ENUM ('GREEN', 'YELLOW', 'RED');
CREATE TYPE template_category AS ENUM ('MARKETING', 'UTILITY', 'AUTHENTICATION');
CREATE TYPE template_status AS ENUM ('DRAFT', 'PENDING', 'APPROVED', 'REJECTED', 'PAUSED');
CREATE TYPE message_direction AS ENUM ('inbound', 'outbound');
CREATE TYPE message_type AS ENUM ('text', 'template', 'image', 'document', 'audio', 'video');
CREATE TYPE message_status AS ENUM ('sent', 'delivered', 'read', 'failed');
CREATE TYPE conversation_status AS ENUM ('open', 'resolved', 'bot');
CREATE TYPE campaign_status AS ENUM ('draft', 'scheduled', 'sending', 'sent', 'failed');
CREATE TYPE campaign_contact_status AS ENUM ('pending', 'sent', 'delivered', 'read', 'failed');

-- LAYER 1: IDENTITY & BILLING
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    plan plan_type NOT NULL DEFAULT 'starter',
    plan_expires_at TIMESTAMPTZ,
    razorpay_customer_id VARCHAR(100),
    razorpay_subscription_id VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    password_hash TEXT NOT NULL,
    is_org_owner BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX users_org_email_idx ON users(organization_id, email);
CREATE INDEX users_org_idx ON users(organization_id);

-- LAYER 2: WORKSPACES
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(50) NOT NULL,
    timezone VARCHAR(50) NOT NULL DEFAULT 'Asia/Kolkata',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX workspaces_org_slug_idx ON workspaces(organization_id, slug);
CREATE INDEX workspaces_org_idx ON workspaces(organization_id);

CREATE TABLE workspace_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    user_id UUID NOT NULL REFERENCES users(id),
    role workspace_role NOT NULL DEFAULT 'agent',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX workspace_members_ws_user_idx ON workspace_members(workspace_id, user_id);
CREATE INDEX workspace_members_user_idx ON workspace_members(user_id);

-- LAYER 3: META / WHATSAPP CONNECTION
CREATE TABLE wa_business_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID UNIQUE NOT NULL REFERENCES workspaces(id),  -- STRICT 1:1
    meta_business_portfolio_id VARCHAR(100),
    waba_id VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    access_token TEXT NOT NULL,  -- store encrypted
    business_verified BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE wa_phone_numbers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    waba_id UUID NOT NULL REFERENCES wa_business_accounts(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),  -- denormalized
    phone_number VARCHAR(20) NOT NULL,
    phone_number_id VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    quality_rating quality_rating DEFAULT 'GREEN',
    messaging_limit INTEGER DEFAULT 1000,
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    is_connected BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX wa_phone_numbers_waba_idx ON wa_phone_numbers(waba_id);
CREATE INDEX wa_phone_numbers_ws_idx ON wa_phone_numbers(workspace_id);

-- LAYER 4: CONTACTS
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    phone VARCHAR(20) NOT NULL,
    name VARCHAR(100),
    email VARCHAR(255),
    status contact_status NOT NULL DEFAULT 'subscribed',
    source contact_source NOT NULL DEFAULT 'manual',
    attributes JSONB DEFAULT '{}',
    last_seen_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX contacts_ws_phone_idx ON contacts(workspace_id, phone);
CREATE INDEX contacts_ws_status_idx ON contacts(workspace_id, status);
CREATE INDEX contacts_ws_idx ON contacts(workspace_id);

CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    name VARCHAR(50) NOT NULL,
    color VARCHAR(7) DEFAULT '#3B82F6',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX tags_ws_name_idx ON tags(workspace_id, name);

CREATE TABLE contact_tags (
    contact_id UUID NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (contact_id, tag_id)
);
CREATE INDEX contact_tags_tag_idx ON contact_tags(tag_id);

CREATE TABLE lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX lists_ws_idx ON lists(workspace_id);

CREATE TABLE list_contacts (
    list_id UUID NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
    contact_id UUID NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
    PRIMARY KEY (list_id, contact_id)
);
CREATE INDEX list_contacts_contact_idx ON list_contacts(contact_id);

-- LAYER 5: MESSAGING
CREATE TABLE templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    waba_id UUID NOT NULL REFERENCES wa_business_accounts(id),
    meta_template_id VARCHAR(100),
    name VARCHAR(100) NOT NULL,
    category template_category NOT NULL,
    language VARCHAR(10) NOT NULL DEFAULT 'en_US',
    status template_status NOT NULL DEFAULT 'DRAFT',
    components JSONB NOT NULL DEFAULT '[]',
    rejection_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX templates_waba_name_idx ON templates(waba_id, name);
CREATE INDEX templates_ws_status_idx ON templates(workspace_id, status);

CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    phone_number_id UUID NOT NULL REFERENCES wa_phone_numbers(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    assigned_to UUID REFERENCES users(id),
    status conversation_status NOT NULL DEFAULT 'open',
    last_message_at TIMESTAMPTZ,
    last_customer_message_at TIMESTAMPTZ,
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX conversations_phone_contact_idx ON conversations(phone_number_id, contact_id);
CREATE INDEX conversations_ws_status_time_idx ON conversations(workspace_id, status, last_message_at DESC);
CREATE INDEX conversations_contact_idx ON conversations(contact_id);
CREATE INDEX conversations_assigned_idx ON conversations(assigned_to);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    phone_number_id UUID NOT NULL REFERENCES wa_phone_numbers(id),
    wa_message_id VARCHAR(100) UNIQUE,
    direction message_direction NOT NULL,
    type message_type NOT NULL DEFAULT 'text',
    content JSONB NOT NULL DEFAULT '{}',
    status message_status,
    sent_by UUID REFERENCES users(id),
    error_code VARCHAR(20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status_updated_at TIMESTAMPTZ
);
CREATE INDEX messages_conversation_time_idx ON messages(conversation_id, created_at DESC);
CREATE INDEX messages_wa_id_idx ON messages(wa_message_id);
CREATE INDEX messages_ws_idx ON messages(workspace_id);

CREATE TABLE conversation_notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    user_id UUID NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX conversation_notes_conv_idx ON conversation_notes(conversation_id);

-- LAYER 6: CAMPAIGNS
CREATE TABLE campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    phone_number_id UUID NOT NULL REFERENCES wa_phone_numbers(id),
    name VARCHAR(100) NOT NULL,
    template_id UUID NOT NULL REFERENCES templates(id),
    template_variables JSONB DEFAULT '{}',
    status campaign_status NOT NULL DEFAULT 'draft',
    scheduled_at TIMESTAMPTZ,
    sent_at TIMESTAMPTZ,
    total_count INTEGER DEFAULT 0,
    sent_count INTEGER DEFAULT 0,
    delivered_count INTEGER DEFAULT 0,
    read_count INTEGER DEFAULT 0,
    failed_count INTEGER DEFAULT 0,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX campaigns_ws_status_idx ON campaigns(workspace_id, status);
CREATE INDEX campaigns_scheduled_idx ON campaigns(status, scheduled_at)
    WHERE status = 'scheduled';

CREATE TABLE campaign_contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id UUID NOT NULL REFERENCES campaigns(id),
    contact_id UUID NOT NULL REFERENCES contacts(id),
    wa_message_id VARCHAR(100),
    status campaign_contact_status NOT NULL DEFAULT 'pending',
    error_code VARCHAR(20),
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    read_at TIMESTAMPTZ
);
CREATE INDEX campaign_contacts_campaign_status_idx ON campaign_contacts(campaign_id, status);
CREATE INDEX campaign_contacts_wa_id_idx ON campaign_contacts(wa_message_id);
CREATE INDEX campaign_contacts_contact_idx ON campaign_contacts(contact_id);

-- LAYER 7: UTILITY
CREATE TABLE quick_replies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    title VARCHAR(100) NOT NULL,
    content TEXT NOT NULL,
    shortcut VARCHAR(30),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX quick_replies_ws_shortcut_idx ON quick_replies(workspace_id, shortcut)
    WHERE shortcut IS NOT NULL;
CREATE INDEX quick_replies_ws_idx ON quick_replies(workspace_id);
```

---

## 5. Architecture Decisions

### Multi-tenancy
- Every query filters by `workspace_id`
- JWT contains `userId` and `workspaceId` (current active workspace)
- Service layer validates that the requested resource belongs to the JWT's `workspaceId`
- Users can belong to multiple workspaces — switch via workspace selector in UI

### Workspace ↔ WABA Relationship
- **Strictly 1:1** — one workspace owns exactly one WABA
- Enforced by `UNIQUE` constraint on `wa_business_accounts.workspace_id`
- If a customer needs two brands/WABAs, they create two workspaces
- One WABA can have multiple phone numbers (`wa_phone_numbers`)
- All messages/campaigns reference `phone_number_id`, not `waba_id`
- Templates are scoped to `waba_id` (Meta requirement — templates are per WABA)

### Broadcast Engine (SQS)
```
POST /campaigns/{id}/send
    ↓
CampaignService.initiateSend()
    ↓
Fetch all contact IDs for campaign's list/segment
    ↓
INSERT campaign_contacts rows (status=pending) in batches of 100
    ↓
Enqueue one SQS message per contact:
    { campaignId, contactId, phoneNumberId, templateId, templateVariables }
    ↓
BroadcastConsumer (@SqsListener)
    ↓
Resolve contact's variable values (name, attributes, etc.)
    ↓
POST to Meta Cloud API
    ↓
On success: UPDATE campaign_contacts SET wa_message_id=?, status='sent'
On failure: UPDATE campaign_contacts SET status='failed', error_code=?
    ↓
Atomic counter: UPDATE campaigns SET sent_count = sent_count + 1
```

**Rate limiting:** Add 12ms delay between Meta API calls in consumer = ~80 msg/sec, safe under Meta's limits.

**Crash recovery:** If consumer crashes mid-campaign, SQS redelivers unacknowledged messages. `wa_message_id` uniqueness check prevents duplicates.

### Campaign Scheduling
```
@Scheduled(fixedDelay = 60000)  — runs every 60 seconds
    ↓
SELECT * FROM campaigns 
WHERE status = 'scheduled' AND scheduled_at <= NOW()
    ↓
For each due campaign: call initiateSend() → SQS fan-out
```

### Webhook Processing
```
Meta POST /webhook
    ↓
1. Verify X-Hub-Signature-256 (HMAC-SHA256 with app secret)
2. Return 200 immediately — ALWAYS within 5 seconds
3. @Async("webhookExecutor") processAsync(payload)
    ↓
Parse phone_number_id → find workspace
    ↓
messages[] → handleInbound()
    - dedup on wa_message_id (Meta sends duplicates)
    - find or create contact
    - find or create conversation (unique per phone_number_id + contact_id)
    - update last_customer_message_at (24-hour window tracking)
    - save message
    - push to SSE clients for this workspace
    ↓
statuses[] → handleStatusUpdate()
    - update messages.status
    - if campaign message: update campaign_contacts.status
    - atomic increment on campaigns counters
```

### SSE (Server-Sent Events) for Live Inbox
- `SseService` maintains `Map<String, List<SseEmitter>>` keyed by `workspaceId`
- On new inbound message or conversation update: push to all emitters for that workspace
- Frontend uses native `EventSource` API
- Start with polling (3s interval) in week 1 — replace with SSE in week 3
- **Single instance only for MVP** — if scaling to multiple ECS tasks later, add Redis Pub/Sub

### 24-Hour Window Enforcement
```java
// Before every agent reply:
boolean withinWindow = conversation.getLastCustomerMessageAt() != null
    && Duration.between(conversation.getLastCustomerMessageAt(), Instant.now())
              .toHours() < 24;

if (!withinWindow) {
    throw new WindowExpiredException("Must use approved template after 24 hours");
}
```

### JWT Structure
```json
{
  "sub": "userId",
  "workspaceId": "current-workspace-uuid",
  "organizationId": "org-uuid",
  "role": "admin",
  "exp": 1234567890
}
```

### Security Rules
- All `/api/**` endpoints require valid JWT
- `GET /webhook` and `POST /webhook` are public (verified by Meta signature)
- `POST /auth/login` and `POST /auth/signup` are public
- Role checks in service layer:
  - `admin`: full access
  - `agent`: can read/reply conversations assigned to them or unassigned; no campaigns
  - `marketing`: can create campaigns and templates; no inbox access

---

## 6. Meta Cloud API Integration

### Base URL
```
https://graph.facebook.com/v19.0/
```

### Sending a Template Message
```
POST /{phone-number-id}/messages
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "messaging_product": "whatsapp",
  "to": "919876543210",
  "type": "template",
  "template": {
    "name": "order_confirmation",
    "language": { "code": "en_US" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Matty" },
          { "type": "text", "text": "ORD-12345" }
        ]
      }
    ]
  }
}
```

### Sending a Free-Text Reply (within 24h window)
```
POST /{phone-number-id}/messages
Authorization: Bearer {access_token}

{
  "messaging_product": "whatsapp",
  "to": "919876543210",
  "type": "text",
  "text": { "body": "Hello! How can I help?" }
}
```

### Submitting a Template for Approval
```
POST /{waba-id}/message_templates
Authorization: Bearer {access_token}

{
  "name": "order_confirmation",
  "category": "UTILITY",
  "language": "en_US",
  "components": [...]
}
```

### Webhook Verification (GET)
```
Query params: hub.mode, hub.verify_token, hub.challenge
Response: hub.challenge as plain text (not JSON)
```

### Webhook Events (POST)
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "WABA_ID",
    "changes": [{
      "value": {
        "metadata": {
          "phone_number_id": "123456"
        },
        "messages": [{ "id": "wamid.xxx", "from": "91...", "type": "text", "text": {"body": "Hi"} }],
        "statuses": [{ "id": "wamid.xxx", "status": "delivered", "recipient_id": "91..." }]
      },
      "field": "messages"
    }]
  }]
}
```

---

## 7. API Endpoints

### Auth
```
POST /api/auth/signup          — create org + user + default workspace
POST /api/auth/login           — returns JWT
POST /api/auth/refresh         — refresh JWT
POST /api/auth/logout
```

### Workspaces
```
GET    /api/workspaces                    — all workspaces for current user
POST   /api/workspaces                    — create new workspace
GET    /api/workspaces/{id}
PUT    /api/workspaces/{id}
GET    /api/workspaces/{id}/members
POST   /api/workspaces/{id}/members/invite — send email invite
DELETE /api/workspaces/{id}/members/{userId}
PUT    /api/workspaces/{id}/members/{userId}/role
```

### WhatsApp Connection
```
POST   /api/wa/connect                    — save phone_number_id + access_token
GET    /api/wa/account                    — get WABA + phone numbers for current workspace
GET    /api/wa/numbers                    — list phone numbers
POST   /api/wa/numbers/{id}/set-default
DELETE /api/wa/numbers/{id}
```

### Contacts
```
GET    /api/contacts                      — list with filter/search/pagination
POST   /api/contacts                      — create single contact
GET    /api/contacts/{id}
PUT    /api/contacts/{id}
DELETE /api/contacts/{id}
POST   /api/contacts/import               — upload CSV to S3 → async import job
GET    /api/contacts/import/{jobId}       — check import job status
POST   /api/contacts/{id}/tags
DELETE /api/contacts/{id}/tags/{tagId}
GET    /api/contacts/export               — download CSV
```

### Tags
```
GET    /api/tags
POST   /api/tags
PUT    /api/tags/{id}
DELETE /api/tags/{id}
```

### Lists
```
GET    /api/lists
POST   /api/lists
GET    /api/lists/{id}
PUT    /api/lists/{id}
DELETE /api/lists/{id}
POST   /api/lists/{id}/contacts           — add contacts to list
DELETE /api/lists/{id}/contacts/{contactId}
GET    /api/lists/{id}/contacts           — paginated contacts in list
```

### Templates
```
GET    /api/templates                     — filter by status, category
POST   /api/templates                     — create + auto-submit to Meta
GET    /api/templates/{id}
PUT    /api/templates/{id}               — only DRAFT templates
DELETE /api/templates/{id}
POST   /api/templates/{id}/submit         — submit to Meta for approval
POST   /api/templates/sync                — pull latest status from Meta
```

### Campaigns
```
GET    /api/campaigns                     — list with pagination
POST   /api/campaigns                     — create draft
GET    /api/campaigns/{id}
PUT    /api/campaigns/{id}               — only DRAFT campaigns
DELETE /api/campaigns/{id}
POST   /api/campaigns/{id}/send           — trigger immediate send
POST   /api/campaigns/{id}/schedule       — set scheduled_at
POST   /api/campaigns/{id}/cancel         — cancel scheduled campaign
GET    /api/campaigns/{id}/stats          — delivery funnel stats
GET    /api/campaigns/{id}/contacts       — per-contact status table
```

### Conversations (Inbox)
```
GET    /api/conversations                 — list, filter by status/agent/tag
GET    /api/conversations/{id}
PUT    /api/conversations/{id}/assign     — assign to agent
PUT    /api/conversations/{id}/resolve
PUT    /api/conversations/{id}/reopen
GET    /api/conversations/{id}/messages   — paginated message history
POST   /api/conversations/{id}/notes
GET    /api/conversations/{id}/notes
GET    /api/inbox/stream                  — SSE endpoint, produces text/event-stream
```

### Messages
```
POST   /api/conversations/{id}/messages   — send reply (free text or template)
GET    /api/quick-replies                 — list quick replies
POST   /api/quick-replies
PUT    /api/quick-replies/{id}
DELETE /api/quick-replies/{id}
```

### Webhook
```
GET    /webhook                           — Meta verification (public)
POST   /webhook                           — Meta event handler (public, signature-verified)
```

### Dashboard
```
GET    /api/dashboard/summary             — key metrics for date range
GET    /api/dashboard/campaigns           — recent campaigns with stats
```

---

## 8. Key Implementation Patterns

### Multi-tenant Service Pattern
```java
// Every service method follows this pattern
public Contact getContact(UUID contactId, UUID workspaceId) {
    return contactRepository
        .findByIdAndWorkspaceId(contactId, workspaceId)  // always scope by workspace
        .orElseThrow(() -> new ResourceNotFoundException("Contact not found"));
}
```

### JWT Extraction in Controllers
```java
// SecurityUtils helper
public static UUID getCurrentWorkspaceId(Authentication auth) {
    JwtAuthToken token = (JwtAuthToken) auth;
    return token.getWorkspaceId();
}

// Usage in any controller
@GetMapping("/contacts")
public Page<ContactDto> list(Authentication auth, Pageable pageable) {
    UUID workspaceId = SecurityUtils.getCurrentWorkspaceId(auth);
    return contactService.list(workspaceId, pageable);
}
```

### Atomic Campaign Counter Updates
```java
// Avoid race conditions — use atomic DB increments
@Modifying
@Query("UPDATE Campaign c SET c.deliveredCount = c.deliveredCount + 1 WHERE c.id = :id")
void incrementDelivered(@Param("id") UUID id);

@Modifying
@Query("UPDATE Campaign c SET c.readCount = c.readCount + 1 WHERE c.id = :id")
void incrementRead(@Param("id") UUID id);
```

### SSE Push Pattern
```java
@Service
public class SseService {
    private final Map<String, CopyOnWriteArrayList<SseEmitter>> emitters =
        new ConcurrentHashMap<>();

    public SseEmitter register(String workspaceId) {
        SseEmitter emitter = new SseEmitter(0L);
        emitters.computeIfAbsent(workspaceId, k -> new CopyOnWriteArrayList<>()).add(emitter);
        emitter.onCompletion(() -> remove(workspaceId, emitter));
        emitter.onTimeout(() -> remove(workspaceId, emitter));
        return emitter;
    }

    public void push(String workspaceId, Object data) {
        List<SseEmitter> list = emitters.getOrDefault(workspaceId, List.of());
        List<SseEmitter> dead = new ArrayList<>();
        for (SseEmitter emitter : list) {
            try {
                emitter.send(SseEmitter.event().data(data));
            } catch (Exception e) {
                dead.add(emitter);
            }
        }
        list.removeAll(dead);
    }
}
```

### Webhook Deduplication
```java
public void handleInbound(InboundMessage msg, WaPhoneNumber phoneNumber) {
    // Always check first — Meta sends duplicate webhook events
    if (messageRepository.existsByWaMessageId(msg.getId())) {
        return;
    }
    // ... rest of processing
}
```

### Contact Import (CSV via S3)
```java
// 1. Controller: get presigned S3 upload URL
// 2. Frontend: upload CSV directly to S3
// 3. Controller: POST /contacts/import with S3 key
// 4. Service: @Async job reads S3 file, parses CSV, bulk inserts
// 5. On duplicate phone: update name/attributes, skip insert
// 6. Return job ID for polling
```

---

## 9. Application Configuration

### application.yml
```yaml
spring:
  application:
    name: watools-api
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASS}
    hikari:
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          time_zone: UTC
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080

jwt:
  secret: ${JWT_SECRET}
  expiry-hours: ${JWT_EXPIRY_HOURS:24}

meta:
  api:
    base-url: https://graph.facebook.com/v19.0
  webhook:
    verify-token: ${META_VERIFY_TOKEN}
  app:
    secret: ${META_APP_SECRET}

aws:
  region: ${AWS_REGION:ap-south-1}
  sqs:
    broadcast-queue-url: ${SQS_BROADCAST_QUEUE_URL}
  s3:
    bucket: ${S3_BUCKET}

razorpay:
  key-id: ${RAZORPAY_KEY_ID}
  key-secret: ${RAZORPAY_KEY_SECRET}
  webhook-secret: ${RAZORPAY_WEBHOOK_SECRET}

thread-pool:
  webhook:
    core-size: 10
    max-size: 50
    queue-capacity: 500
```

### application-local.yml
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/watools
    username: watools
    password: secret
  jpa:
    show-sql: true

aws:
  endpoint: http://localhost:4566
  access-key: test
  secret-key: test
  sqs:
    broadcast-queue-url: http://localhost:4566/000000000000/broadcast-queue
  s3:
    bucket: wa-tools-local

jwt:
  secret: local_secret_minimum_32_characters_long_for_hs256
  expiry-hours: 168

mail:
  host: localhost
  port: 1025

logging:
  level:
    com.watools: DEBUG
    org.springframework.security: DEBUG
```

---

## 10. Local Development Setup

### docker-compose.yml (place in project root)
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    container_name: wa-postgres
    environment:
      POSTGRES_DB: watools
      POSTGRES_USER: watools
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U watools"]
      interval: 5s
      retries: 5

  localstack:
    image: localstack/localstack
    container_name: wa-localstack
    ports:
      - "4566:4566"
    environment:
      - SERVICES=sqs,s3
      - DEFAULT_REGION=ap-south-1
    volumes:
      - localstack-data:/var/lib/localstack
      - ./localstack-init:/etc/localstack/init/ready.d

  mailpit:
    image: axllent/mailpit
    container_name: wa-mailpit
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  postgres-data:
  localstack-data:
```

### localstack-init/init.sh
```bash
#!/bin/bash
awslocal sqs create-queue --queue-name broadcast-queue --region ap-south-1
awslocal s3 mb s3://wa-tools-local --region ap-south-1
echo "LocalStack resources created."
```

### Daily dev commands
```bash
# Start all local services
docker compose up -d

# Run Spring Boot with local profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# Run Next.js
npm run dev

# Expose webhook to Meta
ngrok http 8080
```

### Local service URLs
| Service | URL |
|---|---|
| Next.js | http://localhost:3000 |
| Spring Boot | http://localhost:8080 |
| Postgres | localhost:5432 |
| LocalStack | http://localhost:4566 |
| Mailpit (email UI) | http://localhost:8025 |

---

## 11. Feature Implementation Order

Build in this exact sequence — each depends on the previous:

```
Week 1: Project setup + Meta webhook endpoint
Week 2: Contacts module + Templates module
Week 3: Broadcast campaigns (SQS producer + consumer + status tracking)
Week 4: Team inbox (SSE + inbound webhook handler + reply endpoint)
Week 5: Dashboard + Razorpay billing + onboarding flow
Week 6: Soft launch — 10 beta customers
Week 7: Fix bugs from beta feedback only
Week 8: Convert to paid — target ₹25k MRR
```

### Build Order Within Each Feature
1. Flyway migration SQL (if new tables needed)
2. JPA Entity classes
3. Spring Data Repository interfaces
4. Service class (business logic)
5. REST Controller (thin — validate, call service, return DTO)
6. Next.js page + API calls

---

## 12. Post-MVP Features (Do Not Build in Weeks 1–8)
- Chatbot / no-code flow builder
- Shopify / WooCommerce integration
- AI reply suggestions
- Click-to-WhatsApp ads
- WhatsApp catalog and payments
- Mobile app
- Redis + WebSockets (add only when scaling beyond single ECS instance)
- Advanced analytics (engagement scoring, agent reports)
- White-labelling

---

## 13. Important Business Rules (Must Enforce in Code)

1. **Unsubscribed contacts are NEVER sent campaign messages** — filter in CampaignService before enqueueing
2. **Free-text replies only within 24 hours of last customer message** — check `last_customer_message_at`
3. **Only APPROVED templates can be used in campaigns** — validate in CampaignService
4. **workspace ↔ WABA is 1:1** — UNIQUE constraint in DB + validation in WaAccountService
5. **wa_message_id deduplication** — always check before processing any inbound message
6. **Return 200 to Meta webhook within 5 seconds** — always process async
7. **Verify X-Hub-Signature-256 on every POST /webhook** — reject 403 if invalid
8. **Contacts from different workspaces are completely isolated** — every query includes workspace_id filter
