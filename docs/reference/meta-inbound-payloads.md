# Meta WhatsApp Business Platform — Inbound Webhook Payload Reference

Source of truth for every webhook event we receive from the Meta Cloud API on `POST /webhook`.
Cleaned up and expanded from `meta inbound reference.txt`.

> **Provenance.** Payloads marked **[source]** are taken verbatim (cleaned) from the raw reference
> file. Payloads marked **[canonical]** are filled in from Meta's published Cloud API schema because
> the raw file only names the subtype (e.g. "image", "location") without giving its body. Treat
> **[canonical]** shapes as the documented Meta contract, not as captured traffic.

## How to read this document

- **Envelope** — every webhook shares the same outer envelope (below); per-event JSON in each
  section shows only the `value` object inside `entry[].changes[].value` unless stated otherwise.
- **Trigger** — the condition under which Meta emits the event.
- **Optional / nullable** — the field tables flag which fields may be absent or `null`. Meta **adds
  fields over time**, so any consumer must ignore unknown fields (never fail deserialization on them).
- **Dedup & ordering** — inbound messages can be **redelivered** (dedup on `messages[].id`), and
  status callbacks can arrive **out of order** (`read` before `delivered`); never regress state.

### Shared envelope

Every webhook (messages, statuses, account, template, flows) is wrapped in:

```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "<WHATSAPP_BUSINESS_ACCOUNT_ID>",
      "time": 1739321024,
      "changes": [
        { "value": { "...": "..." }, "field": "<WEBHOOK_FIELD>" }
      ]
    }
  ]
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `object` | no | Always `whatsapp_business_account`. Reject anything else. |
| `entry[]` | no | Usually one element, but treat as a list. |
| `entry[].id` | no | The WABA id the event belongs to. |
| `entry[].time` | yes | Webhook trigger time (epoch seconds). Absent on some `messages` payloads. |
| `entry[].changes[]` | no | One change per event. |
| `changes[].field` | no | Selects the `value` schema. The values we act on use `field: "messages"`. |
| `changes[].value` | no | Schema varies entirely by `field`. |

---

# Category A — Inbound messages (`field: "messages"`, `value.messages[]`)

When a customer sends a message, Meta delivers it under `field: "messages"` with a `messages` array.
The `value` wrapper is identical across subtypes; only the per-message object varies by `type`.

### A.0 — The `messages` value wrapper **[source]**

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "102290129340398",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": { "display_phone_number": "15550783881", "phone_number_id": "106540352242922" },
        "contacts": [{ "profile": { "name": "Sheena Nelson" }, "wa_id": "16505551234" }],
        "messages": [{
          "from": "16505551234",
          "id": "wamid.HBgLMTY1MDM4Nzk0MzkVAgASGBQzQTRBNjU5OUFFRTAzODEwMTQ0RgA=",
          "timestamp": "1749416383",
          "type": "text",
          "text": { "body": "Does it come in another color?" }
        }]
      },
      "field": "messages"
    }]
  }]
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `value.messaging_product` | no | Always `whatsapp`. |
| `value.metadata.display_phone_number` | no | The business number that received the message. |
| `value.metadata.phone_number_id` | no | **Our routing key** — resolve to a `wa_phone_numbers` row. |
| `value.contacts[]` | yes | Sender profile. May be omitted on some events. |
| `value.contacts[].profile.name` | yes | Display name; user-controlled, may be absent. |
| `value.contacts[].wa_id` | yes | Sender's WhatsApp id (usually equals `messages[].from`). |
| `value.messages[]` | no\* | Present on inbound; absent on status-only payloads (see Category B). |
| `value.errors[]` | yes | Account/system-level errors (see Category F). |

### Shared per-message fields

Every object in `messages[]` carries these, plus a subtype block named after `type`:

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `from` | no | Sender wa_id (E.164 without `+`). |
| `id` | no | `wamid....` — **dedup key** (rule 5). |
| `timestamp` | no | Epoch seconds, as a string. |
| `type` | no | Discriminator: `text`, `image`, `audio`, `video`, `document`, `sticker`, `location`, `contacts`, `interactive`, `button`, `reaction`, `order`, `system`, `unsupported`. |
| `context` | yes | Present when the message is a reply, a forward, or comes from a click-to-WA ad. |
| `errors[]` | yes | Present only when `type` is `unsupported`. |

#### `context` object (reply / forward / referral)

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `context.from` | yes | wa_id of the message being replied to. |
| `context.id` | yes | `wamid` of the message being replied to. |
| `context.forwarded` | yes | `true` if forwarded. |
| `context.frequently_forwarded` | yes | `true` if forwarded many times. |
| `context.referral` | yes | Click-to-WhatsApp ad referral (source_url, source_type, source_id, headline, body, media_type, image_url, video_url, ctwa_clid). |

---

## A.1 — Text **[source]**

**When:** the customer sends a plain text message. **Trigger:** any non-media text body.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDM4Nzk0MzkVAgASGBQzQTRBNjU5OUFFRTAzODEwMTQ0RgA=",
  "timestamp": "1749416383",
  "type": "text",
  "text": { "body": "Does it come in another color?" }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `text.body` | no | The message text. |

---

## A.2 — Image **[canonical]**

**When:** the customer sends a photo. **Trigger:** image attachment. Media bytes are **not** in the
payload — fetch via the media `id`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416400",
  "type": "image",
  "image": {
    "id": "1310463202926571",
    "mime_type": "image/jpeg",
    "sha256": "G3sf7...=",
    "caption": "Is this the right one?"
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `image.id` | no | Media id — fetch bytes via `GET /{media-id}`. |
| `image.mime_type` | no | e.g. `image/jpeg`, `image/png`. |
| `image.sha256` | yes | Integrity hash. |
| `image.caption` | yes | Present only if the user added a caption. |

---

## A.3 — Audio & voice **[canonical]**

**When:** the customer sends an audio file or a recorded voice note. **Trigger:** `type: "audio"`;
voice notes set `voice: true`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416410",
  "type": "audio",
  "audio": {
    "id": "971571215375675",
    "mime_type": "audio/ogg; codecs=opus",
    "sha256": "X9...=",
    "voice": true
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `audio.id` | no | Media id. |
| `audio.mime_type` | no | `audio/ogg; codecs=opus` for voice notes. |
| `audio.sha256` | yes | Integrity hash. |
| `audio.voice` | yes | `true` for a recorded voice note, otherwise absent/false. |

> The legacy `type: "voice"` label is treated the same as `audio` with `voice: true`.

---

## A.4 — Video **[canonical]**

**When:** the customer sends a video. **Trigger:** video attachment.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416420",
  "type": "video",
  "video": {
    "id": "1166846690780726",
    "mime_type": "video/mp4",
    "sha256": "Q1...=",
    "caption": "Demo clip"
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `video.id` | no | Media id. |
| `video.mime_type` | no | e.g. `video/mp4`. |
| `video.sha256` | yes | Integrity hash. |
| `video.caption` | yes | Optional caption. |

---

## A.5 — Document **[canonical]**

**When:** the customer sends a file. **Trigger:** document attachment.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416430",
  "type": "document",
  "document": {
    "id": "971571215375675",
    "mime_type": "application/pdf",
    "sha256": "Z2...=",
    "filename": "invoice-2026-06.pdf",
    "caption": "Latest invoice"
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `document.id` | no | Media id. |
| `document.mime_type` | no | e.g. `application/pdf`. |
| `document.sha256` | yes | Integrity hash. |
| `document.filename` | yes | Original file name. |
| `document.caption` | yes | Optional caption. |

---

## A.6 — Sticker **[canonical]**

**When:** the customer sends a sticker. **Trigger:** `type: "sticker"`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416440",
  "type": "sticker",
  "sticker": {
    "id": "335017183975037",
    "mime_type": "image/webp",
    "sha256": "A1...=",
    "animated": false
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `sticker.id` | no | Media id. |
| `sticker.mime_type` | no | Always `image/webp`. |
| `sticker.sha256` | yes | Integrity hash. |
| `sticker.animated` | yes | `true` for animated stickers. |

---

## A.7 — Location **[canonical]**

**When:** the customer shares a location. **Trigger:** `type: "location"`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416450",
  "type": "location",
  "location": {
    "latitude": 12.9716,
    "longitude": 77.5946,
    "name": "Cubbon Park",
    "address": "Kasturba Road, Bengaluru"
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `location.latitude` | no | Decimal degrees. |
| `location.longitude` | no | Decimal degrees. |
| `location.name` | yes | Place name (only for named/POI shares). |
| `location.address` | yes | Street address (only for named/POI shares). |

---

## A.8 — Contacts **[canonical]**

**When:** the customer shares one or more contact cards. **Trigger:** `type: "contacts"`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416460",
  "type": "contacts",
  "contacts": [{
    "name": { "first_name": "Asha", "formatted_name": "Asha Rao", "last_name": "Rao" },
    "phones": [{ "phone": "+919812345678", "type": "MOBILE", "wa_id": "919812345678" }],
    "emails": [{ "email": "asha@example.com", "type": "WORK" }]
  }]
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `contacts[].name.formatted_name` | no | Required per card. |
| `contacts[].name.first_name` / `last_name` | yes | Optional name parts. |
| `contacts[].phones[]` | yes | Each: `phone`, `type` (CELL/MOBILE/WORK/HOME), `wa_id` (if on WhatsApp). |
| `contacts[].emails[]` | yes | Each: `email`, `type`. |
| `contacts[].org` / `urls` / `addresses` / `birthday` | yes | Rarely populated; ignore if absent. |

---

## A.9 — Interactive replies **[source: nfm_reply / canonical: button_reply, list_reply]**

**When:** the customer taps a reply button, picks a list option, or completes a Flow.
**Trigger:** `type: "interactive"`; the nested `interactive.type` discriminates.

### A.9a — `button_reply` (interactive reply button) **[canonical]**

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416470",
  "type": "interactive",
  "interactive": {
    "type": "button_reply",
    "button_reply": { "id": "track-order", "title": "Track my order" }
  }
}
```

### A.9b — `list_reply` (interactive list) **[canonical]**

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416480",
  "type": "interactive",
  "interactive": {
    "type": "list_reply",
    "list_reply": { "id": "SKU-42", "title": "Blue / Large", "description": "In stock" }
  }
}
```

### A.9c — `nfm_reply` (Flow submission) **[source]**

```json
{
  "context": { "from": "16315558151", "id": "gBGGEiRVVgBPAgm7FUgc73noXjo" },
  "from": "<USER_ACCOUNT_NUMBER>",
  "id": "<MESSAGE_ID>",
  "type": "interactive",
  "interactive": {
    "type": "nfm_reply",
    "nfm_reply": {
      "name": "flow",
      "body": "Sent",
      "response_json": "{\"flow_token\": \"<FLOW_TOKEN>\", \"optional_param1\": \"<value1>\"}"
    }
  },
  "timestamp": "<MESSAGE_SEND_TIMESTAMP>"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `interactive.type` | no | `button_reply` \| `list_reply` \| `nfm_reply`. |
| `interactive.button_reply.id` / `.title` | (when button) | Button payload id and label. |
| `interactive.list_reply.id` / `.title` | (when list) | Selected row id and title. |
| `interactive.list_reply.description` | yes | Optional row description. |
| `interactive.nfm_reply.name` | (when flow) | Usually `flow`. |
| `interactive.nfm_reply.body` | yes | Usually `Sent`. |
| `interactive.nfm_reply.response_json` | (when flow) | **Stringified JSON** — parse to read `flow_token` + Flow answers. |

---

## A.10 — Button (template quick-reply postback) **[canonical]**

**When:** the customer taps a **quick-reply button on a template** we sent. Distinct from
A.9 interactive buttons. **Trigger:** `type: "button"`. Usually carries `context` linking the
original template message.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416490",
  "type": "button",
  "context": { "from": "15550783881", "id": "wamid.HBgL...template..." },
  "button": { "text": "Stop promotions", "payload": "UNSUBSCRIBE" }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `button.text` | no | Visible button label. |
| `button.payload` | no | Developer-defined payload — drives opt-out / routing logic. |
| `context.id` | yes | `wamid` of the template message that rendered the button. |

---

## A.11 — Reaction **[canonical]**

**When:** the customer reacts to (or removes a reaction from) a message. **Trigger:**
`type: "reaction"`. An **empty** `emoji` means the reaction was removed.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416500",
  "type": "reaction",
  "reaction": { "message_id": "wamid.HBgL...original...", "emoji": "👍" }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `reaction.message_id` | no | `wamid` of the message being reacted to. |
| `reaction.emoji` | yes | The emoji; **empty string when the reaction is removed**. |

---

## A.12 — Order (cart / catalog) **[canonical]**

**When:** the customer sends a cart from the business catalog. **Trigger:** `type: "order"`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416510",
  "type": "order",
  "order": {
    "catalog_id": "1392984211309544",
    "text": "Please ship today",
    "product_items": [
      { "product_retailer_id": "SKU-42", "quantity": "2", "item_price": "499.00", "currency": "INR" }
    ]
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `order.catalog_id` | no | Catalog the items came from. |
| `order.text` | yes | Optional note from the customer. |
| `order.product_items[]` | no | Each: `product_retailer_id`, `quantity`, `item_price`, `currency`. |

---

## A.13 — System **[canonical]**

**When:** a system event in the conversation — most commonly the customer changed their phone number.
**Trigger:** `type: "system"`.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416520",
  "type": "system",
  "system": {
    "body": "User A changed from 16315551111 to 16505551234",
    "wa_id": "16505551234",
    "type": "user_changed_number"
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `system.body` | no | Human-readable description. |
| `system.type` | no | e.g. `user_changed_number`, `customer_identity_changed`. |
| `system.wa_id` | yes | The new wa_id (on number changes) — update the contact. |

---

## A.14 — Unsupported (inbound error) **[source note: `type: unsupported`]**

**When:** the customer sent a message type we can't process, or Meta couldn't render it. **Trigger:**
`type: "unsupported"`. The `messages[].errors[]` array carries the reason.

```json
{
  "from": "16505551234",
  "id": "wamid.HBgLMTY1MDUx...",
  "timestamp": "1749416530",
  "type": "unsupported",
  "errors": [{
    "code": 131051,
    "title": "Message type unknown",
    "message": "Message type is currently not supported.",
    "error_data": { "details": "Message type is not currently supported." }
  }]
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `errors[].code` | no | Meta error code. |
| `errors[].title` | no | Short error title. |
| `errors[].message` | yes | Longer description. |
| `errors[].error_data.details` | yes | Extra detail. |

---

# Category B — Status callbacks (`field: "messages"`, `value.statuses[]`)

Delivery receipts for **outbound** business messages. Same `field: "messages"`, but the `value`
carries a `statuses` array instead of `messages`. Up to three receipts per message
(`sent` → `delivered` → `read`), plus `failed` on error. **They can arrive out of order.**

### B.0 — The `statuses` value wrapper **[source]**

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "102290129340398",
    "changes": [{
      "value": {
        "messaging_product": "whatsapp",
        "metadata": { "display_phone_number": "15550783881", "phone_number_id": "106540352242922" },
        "statuses": [{
          "id": "wamid.HBgLMTY1MDM4Nzk0MzkVAgARGBI3MTE5MjVBOTE3MDk5QUVFM0YA",
          "status": "delivered",
          "timestamp": "1750263773",
          "recipient_id": "16505551234",
          "conversation": { "id": "6ceb9d929c9bdc4f90e967a32f8639b4", "origin": { "type": "service" } },
          "pricing": { "billable": true, "pricing_model": "CBP", "category": "service" }
        }]
      },
      "field": "messages"
    }]
  }]
}
```

### Shared status fields

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `statuses[].id` | no | `wamid` of the **outbound** message — join key to `messages.wa_message_id` / `campaign_contacts.wa_message_id`. |
| `statuses[].status` | no | `sent` \| `delivered` \| `read` \| `failed`. |
| `statuses[].timestamp` | no | Epoch seconds (string). |
| `statuses[].recipient_id` | no | Customer wa_id. |
| `statuses[].conversation` | yes | Present on `sent`/`delivered`/`read`; absent on most `failed`. |
| `statuses[].conversation.id` | yes | Meta conversation id. |
| `statuses[].conversation.origin.type` | yes | `marketing` \| `utility` \| `authentication` \| `service` \| `referral_conversion`. |
| `statuses[].conversation.expiration_timestamp` | yes | When the conversation window closes (on `sent`). |
| `statuses[].pricing` | yes | `billable`, `pricing_model` (`CBP`/`PMP`), `category`. |
| `statuses[].errors[]` | yes | **Present only on `failed`** — see B.4. |

## B.1 — sent **[canonical]**

**When:** Meta accepted the message for delivery. First receipt; carries the conversation window.

```json
{
  "id": "wamid.HBgL...",
  "status": "sent",
  "timestamp": "1750263770",
  "recipient_id": "16505551234",
  "conversation": {
    "id": "6ceb9d929c9bdc4f90e967a32f8639b4",
    "expiration_timestamp": "1750350170",
    "origin": { "type": "marketing" }
  },
  "pricing": { "billable": true, "pricing_model": "CBP", "category": "marketing" }
}
```

## B.2 — delivered **[source]**

**When:** the message reached the customer's device. See B.0 for the full payload.

## B.3 — read **[canonical]**

**When:** the customer opened the message. May arrive before `delivered` — never regress status.

```json
{
  "id": "wamid.HBgL...",
  "status": "read",
  "timestamp": "1750263800",
  "recipient_id": "16505551234"
}
```

## B.4 — failed **[canonical]**

**When:** the message could not be delivered. **Trigger:** any send failure (re-engagement window
closed, invalid number, user block, rate limit, etc.). Carries `errors[]`.

```json
{
  "id": "wamid.HBgL...",
  "status": "failed",
  "timestamp": "1750263780",
  "recipient_id": "16505551234",
  "errors": [{
    "code": 131049,
    "title": "This message was not delivered to maintain healthy ecosystem engagement.",
    "message": "This message was not delivered to maintain healthy ecosystem engagement.",
    "error_data": { "details": "Free-form messages can only be sent within 24h of the last user message." },
    "href": "https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes/"
  }]
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `errors[].code` | no | Persist to `messages.error_code` / `campaign_contacts.error_code`. |
| `errors[].title` | no | Short reason. |
| `errors[].message` | yes | Longer reason. |
| `errors[].error_data.details` | yes | Extra detail. |
| `errors[].href` | yes | Link to Meta's error-code docs. |

---

# Category C — Account & WABA management webhooks

These do not bear messages; they update WABA/phone-number standing. Out of scope for the inbound
message/status processors, documented here for completeness.

## C.1 — `account_review_update` **[source]**

**When:** a WABA has been reviewed against policy. **Trigger:** review decision changes.

```json
{ "value": { "decision": "APPROVED" }, "field": "account_review_update" }
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `decision` | no | `APPROVED` \| `REJECTED` \| `PENDING` \| `DEFERRED`. |

## C.2 — `account_update` **[source]**

**When:** WABA-level changes — verification, partner sharing, policy violations, offboarding,
reconnection, deletion, pricing tier. **Trigger:** `value.event` drives which sub-object appears.

`event` values: `ACCOUNT_DELETED`, `ACCOUNT_RESTRICTION`, `ACCOUNT_VIOLATION`, `AD_ACCOUNT_LINKED`,
`AUTH_INTL_PRICE_ELIGIBILITY_UPDATE`, `BUSINESS_PRIMARY_LOCATION_COUNTRY_UPDATE`, `DISABLED_UPDATE`,
`MM_LITE_TERMS_SIGNED`, `PARTNER_ADDED`, `PARTNER_APP_INSTALLED`, `PARTNER_APP_UNINSTALLED`,
`PARTNER_CLIENT_CERTIFICATION_STATUS_UPDATE`, `PARTNER_REMOVED`, `VOLUME_BASED_PRICING_TIER_UPDATE`,
`ACCOUNT_OFFBOARDED`, `ACCOUNT_RECONNECTED`.

```json
{
  "value": {
    "event": "ACCOUNT_RESTRICTION",
    "restriction_info": [
      { "restriction_type": "RESTRICTED_BIZ_INITIATED_MESSAGING", "expiration": 1641330498 },
      { "restriction_type": "RESTRICTED_ADD_PHONE_NUMBER_ACTION", "expiration": 1641330498 }
    ]
  },
  "field": "account_update"
}
```

```json
{
  "value": {
    "event": "PARTNER_ADDED",
    "waba_info": {
      "waba_id": "980198427658004",
      "owner_business_id": "2329417887457253",
      "solution_id": "1715120619246906",
      "solution_partner_business_ids": ["2949482758682047", "520744086200222"]
    }
  },
  "field": "account_update"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `event` | no | Drives which sub-object is present. |
| `country` | yes | `BUSINESS_PRIMARY_LOCATION_COUNTRY_UPDATE` only. |
| `waba_info` | yes | `AD_ACCOUNT_LINKED` / `PARTNER_*` events. |
| `violation_info.violation_type` | yes | `ACCOUNT_VIOLATION` only. |
| `auth_international_rate_eligibility` | yes | `AUTH_INTL_PRICE_ELIGIBILITY_UPDATE` only. |
| `ban_info` | yes | `DISABLED_UPDATE` only (`waba_ban_state`, `waba_ban_date`). |
| `volume_tier_info` | yes | `VOLUME_BASED_PRICING_TIER_UPDATE` only. |
| `disconnection_info` | yes | `PARTNER_REMOVED` only (`reason`, `initiated_by`). |
| `partner_client_certification_info` | yes | `PARTNER_CLIENT_CERTIFICATION_STATUS_UPDATE` only. |
| `restriction_info[]` | yes | `ACCOUNT_RESTRICTION` only (`restriction_type`, `expiration`, `remediation`). |

Other events carry only `{ "event": "<EVENT>" }` with their one relevant sub-object, e.g.
`ACCOUNT_DELETED`, `ACCOUNT_OFFBOARDED`, `ACCOUNT_RECONNECTED`.

## C.3 — `business_capability_update` **[source]**

**When:** messaging/phone-number limits change for the WABA or portfolio.

```json
{
  "value": { "max_daily_conversations_per_business": 2000, "max_phone_numbers_per_waba": 25 },
  "field": "business_capability_update"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `max_daily_conversation_per_phone` | yes | **Removed Feb 2026** — use `max_daily_conversations_per_business`. |
| `max_daily_conversations_per_business` | yes | Tier: `TIER_250`/`TIER_2K`/`TIER_10K`/`TIER_100K`/`TIER_UNLIMITED`. |
| `max_phone_numbers_per_business` | yes | Only if `per_phone == 250`. |
| `max_phone_numbers_per_waba` | yes | Only if `per_phone != 250`. |

## C.4 — `phone_number_name_update` **[source]**

**When:** a business phone-number display-name verification resolves.

```json
{
  "value": {
    "display_phone_number": "15550783881",
    "decision": "APPROVED",
    "requested_verified_name": "Lucky Shrub",
    "rejection_reason": null
  },
  "field": "phone_number_name_update"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `display_phone_number` | no | Number whose name was reviewed. |
| `decision` | no | `APPROVED` \| `DEFERRED` \| `PENDING` \| `REJECTED`. |
| `requested_verified_name` | no | Submitted display name. |
| `rejection_reason` | yes | `NAME_*` codes, `UNKNOWN`, or `null` when accepted. |

## C.5 — `phone_number_quality_update` **[source]**

**When:** a phone number's throughput level / messaging limit changes.

```json
{
  "value": {
    "display_phone_number": "15550783881",
    "event": "THROUGHPUT_UPGRADE",
    "current_limit": "TIER_UNLIMITED"
  },
  "field": "phone_number_quality_update"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `display_phone_number` | no | The number. |
| `event` | no | `ONBOARDING` \| `THROUGHPUT_UPGRADE`. |
| `old_limit` | yes | Messaging-limit changes only (**removed Feb 2026**). |
| `current_limit` | yes | **Removed Feb 2026** — use `max_daily_conversations_per_business`. |
| `max_daily_conversations_per_business` | yes | Tier: `TIER_50`…`TIER_UNLIMITED`/`TIER_NOT_SET`. |

---

# Category D — Template management webhooks

## D.1 — `message_template_components_update` **[source]**

**When:** a template's components are edited.

```json
{
  "value": {
    "message_template_id": 1315502779341834,
    "message_template_name": "order_confirmation",
    "message_template_language": "en_US",
    "message_template_title": "Your order is confirmed!",
    "message_template_element": "Thank you for your order, {{1}}! Your order number is {{2}}...",
    "message_template_footer": "Lucky Shrub: the Succulent Specialists!",
    "message_template_buttons": [
      { "message_template_button_type": "PHONE_NUMBER", "message_template_button_text": "Phone support", "message_template_button_phone_number": "+15550783881" },
      { "message_template_button_type": "URL", "message_template_button_text": "Email support", "message_template_button_url": "https://www.luckyshrub.com/support" }
    ]
  },
  "field": "message_template_components_update"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `message_template_id` / `_name` / `_language` | no | Identify the template. |
| `message_template_element` | no | Body text. |
| `message_template_title` | yes | Present only with a text header. |
| `message_template_footer` | yes | Present only with a footer. |
| `message_template_buttons[]` | yes | Present only with url/phone buttons; `button_url` / `button_phone_number` per button type. |

Button types: `CATALOG`, `COPY_CODE`, `EXTENSION`, `FLOW`, `MPM`, `ORDER_DETAILS`, `OTP`,
`PHONE_NUMBER`, `POSTBACK`, `REMINDER`, `SEND_LOCATION`, `SPM`, `QUICK_REPLY`, `URL`, `VOICE_CALL`.

## D.2 — `message_template_status_update` **[source]**

**When:** an existing template changes status (approved, rejected, disabled, archived, etc.).

```json
{
  "value": {
    "event": "APPROVED",
    "message_template_id": 1689556908129832,
    "message_template_name": "order_confirmation",
    "message_template_language": "en-US",
    "reason": "NONE",
    "message_template_category": "UTILITY"
  },
  "field": "message_template_status_update"
}
```

```json
{
  "value": {
    "event": "REJECTED",
    "message_template_id": 1689556908129835,
    "message_template_name": "abandoned_cart",
    "message_template_language": "en",
    "reason": "INVALID_FORMAT",
    "message_template_category": "MARKETING",
    "rejection_info": {
      "reason": "Your template has parameters placed next to each other (like {{1}}{{2}}) without text or punctuation between them.",
      "recommendation": "Separate parameters with descriptive text and ensure each parameter is clearly contextualized."
    }
  },
  "field": "message_template_status_update"
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `event` | no | `APPROVED`, `REJECTED`, `ARCHIVED`, `UNARCHIVED`, `DELETED`, `DISABLED`, `FLAGGED`, `IN_APPEAL`, `LIMIT_EXCEEDED`, `LOCKED`, `PAUSED`, `PENDING`, `REINSTATED`, `PENDING_DELETION`. |
| `message_template_id` / `_name` / `_language` | no | Identify the template. |
| `reason` | yes | `ABUSIVE_CONTENT`, `INCORRECT_CATEGORY`, `INVALID_FORMAT`, `NONE`, `PROMOTIONAL`, `SCAM`, `TAG_CONTENT_MISMATCH`, or `null`. |
| `message_template_category` | yes | `MARKETING` \| `UTILITY` \| `AUTHENTICATION`. |
| `disable_info.disable_date` | yes | When disabled. |
| `other_info` | yes | `title` + `description` when locked/unlocked. |
| `rejection_info` | yes | `reason` + `recommendation` when `INVALID_FORMAT`. |

## D.3 — `template_category_update` **[source]**

**When:** a template's category is about to change automatically, or has changed.

```json
{
  "field": "template_category_update",
  "value": {
    "message_template_id": 1234567890,
    "message_template_name": "promo_blast",
    "message_template_language": "en_US",
    "previous_category": "UTILITY",
    "new_category": "MARKETING"
  }
}
```

| Field | Optional/nullable | Notes |
| --- | --- | --- |
| `message_template_id` / `_name` / `_language` | no | Identify the template. |
| `correct_category` | yes | Impending-change notifications only. |
| `category_update_timestamp` | yes | Impending-change notifications only. |
| `previous_category` | yes | Completed-change notifications only. |
| `new_category` | no | Current/target category. |

---

# Category E — Flows webhooks (`field: "flows"`)

Flow performance/lifecycle alerts (the Flow **response** message arrives via the `messages`
webhook — see A.9c). **Trigger:** `value.event`.

`event` values: `FLOW_STATUS_CHANGE`, `CLIENT_ERROR_RATE`, `ENDPOINT_ERROR_RATE`,
`ENDPOINT_LATENCY`, `ENDPOINT_AVAILABILITY`, `FLOW_VERSION_EXPIRY_WARNING`.
Status values: `DRAFT` \| `PUBLISHED` \| `DEPRECATED` \| `BLOCKED` \| `THROTTLED`.

## E.1 — `FLOW_STATUS_CHANGE` **[source]**

```json
{
  "value": {
    "event": "FLOW_STATUS_CHANGE",
    "message": "Flow Webhook 3 changed status from DRAFT to PUBLISHED",
    "flow_id": "6627390910605886",
    "old_status": "DRAFT",
    "new_status": "PUBLISHED"
  },
  "field": "flows"
}
```

## E.2 — `CLIENT_ERROR_RATE` (and `ENDPOINT_ERROR_RATE`) **[source]**

Thresholds 5/10/50%; client window 60 min, endpoint window 30 min.

```json
{
  "value": {
    "event": "CLIENT_ERROR_RATE",
    "message": "The flow client request error rate has reached the 5% threshold...",
    "flow_id": "691244242662581",
    "error_rate": 14.28,
    "threshold": 10,
    "alert_state": "ACTIVATED",
    "errors": [
      { "error_type": "INVALID_SCREEN_TRANSITION", "error_rate": 66.66, "error_count": 2 },
      { "error_type": "PUBLIC_KEY_MISSING", "error_rate": 33.33, "error_count": 1 }
    ]
  },
  "field": "flows"
}
```

`ENDPOINT_ERROR_RATE` is the same shape with `"event": "ENDPOINT_ERROR_RATE"`.

## E.3 — `ENDPOINT_LATENCY` **[source]**

p90 thresholds 1s/5s/7s, 30-min window.

```json
{
  "value": {
    "event": "ENDPOINT_LATENCY",
    "message": "Flow endpoint latency has reached the p90 threshold...",
    "flow_id": "691244242662581",
    "p90_latency": 8000,
    "p50_latency": 500,
    "requests_count": 34,
    "threshold": 7000,
    "alert_state": "ACTIVATED"
  },
  "field": "flows"
}
```

## E.4 — `ENDPOINT_AVAILABILITY` **[source]**

Below 90% threshold, 10-min window.

```json
{
  "value": {
    "event": "ENDPOINT_AVAILABILITY",
    "message": "The flow endpoint availability has breached the 90% threshold...",
    "flow_id": "12345678",
    "alert_state": "ACTIVATED",
    "availability": 75,
    "threshold": 90
  },
  "field": "flows"
}
```

## E.5 — `FLOW_VERSION_EXPIRY_WARNING` **[source]**

```json
{
  "value": {
    "event": "FLOW_VERSION_EXPIRY_WARNING",
    "warning": "Your current Flow version will freeze in 21 days...",
    "flow_id": "6627390910605886"
  },
  "field": "flows"
}
```

| Field (Category E) | Optional/nullable | Notes |
| --- | --- | --- |
| `event` | no | Selects the alert schema. |
| `flow_id` | no | The Flow the alert is about. |
| `message` / `warning` | yes | Human-readable text. |
| `alert_state` | yes | `ACTIVATED` \| `DEACTIVATED`. |
| `error_rate` / `threshold` / `errors[]` | yes | Error-rate alerts. |
| `p90_latency` / `p50_latency` / `requests_count` | yes | Latency alerts. |
| `availability` | yes | Availability alerts. |
| `old_status` / `new_status` | yes | Status-change alerts. |

---

# Category F — Errors (cross-cutting)

Errors surface in **three** places; consumers must read all three:

| Location | Shape | Meaning |
| --- | --- | --- |
| `value.errors[]` | `{code, title, message, error_data}` | System/app/account-level error. |
| `value.messages[].errors[]` | same | Inbound message error (`type: "unsupported"`) — see A.14. |
| `value.statuses[].errors[]` | same (+`href`) | Outbound send failure (`status: "failed"`) — see B.4. |
</invoke>
