# Plan: Attach a Flow to a Campaign (growth-01 Phase B, part 2)

> Status: **planned, not built.** Extends the built growth-01 (Phase A + Phase B follow-up). Design 2026-06-20.
> Prereq context: Phase A (`wa_flows`, `flow_responses`, `FlowService`, `FlowResponseService`,
> `MetaFlowClient`, `/api/flows`) and Phase B follow-up (`wa_flows.followup_template_id`) are **already built
> and tested** (`FlowIntegrationTest`, `MetaFlowClientImplTest`).

**Goal:** let a broadcast campaign send a flow to its whole audience, so flows can be *initiated outside the
24-hour window* (§13.2) — the flow CTA rides on the campaign's **approved template** message. This is the one
remaining Phase B item from `growth-01-whatsapp-flows.md` ("attach flows to campaigns").

## The Meta constraint that drives the whole design

A flow can only ride a template campaign message if the **template itself was authored with a FLOW button**
(component `BUTTONS` → `{ "type": "FLOW", "text": "...", "flow_id": "..." }`), and that template was
**APPROVED** by Meta with the button in place. At send time you then pass a matching **button component
parameter** carrying a per-recipient `flow_token`:

```jsonc
// POST /{phone-number-id}/messages  (type=template) — the part we don't send today
"components": [
  { "type": "body", "parameters": [ /* existing body vars */ ] },
  { "type": "button", "sub_type": "flow", "index": "0",
    "parameters": [ { "type": "action", "action": { "flow_token": "<per-recipient token>" } } ] }
]
```

Today `MetaMessageClientImpl.sendTemplate(...)` only builds the `body` component (see
`buildTemplate`). So the work is: (1) model the flow link on the campaign, (2) extend the template-send call
to append the flow button component, (3) thread a per-recipient `flow_token` through the broadcast engine so
completed responses resolve back to the right flow **and** the right campaign recipient.

> **Non-obvious:** we cannot reuse "flow_token = flow.id" the way `FlowService.send` does, because here we
> also want to attribute the completion to a campaign + campaign_contact. Encode both. See §C.

## Schema — new migration (`V7__campaign_flow.sql`)
```sql
ALTER TABLE campaigns
    ADD COLUMN flow_id UUID REFERENCES wa_flows(id),   -- null = ordinary template campaign
    ADD COLUMN flow_cta VARCHAR(40);                    -- button label echoed for display only
```
No change to `campaign_contacts`. (The `flow_token` is derived, not stored — see §C.) `flow_responses`
already has a nullable `conversation_id`; add nothing there for MVP — campaign attribution rides in the token.

## Non-obvious decisions
1. **Validate at attach time, not send time.** When a campaign sets `flow_id`, require: flow is `published`,
   flow belongs to the workspace, and the campaign's template has a `FLOW` button component referencing the
   same `meta_flow_id` and is `APPROVED`. Fail fast in `CampaignService.create/update` (409) — a half-valid
   campaign must never reach the queue.
2. **Token encodes campaign_contact.** Use `flow_token = "c:" + campaignContactId` (or `flow:<flowId>` for
   the inbox path). `FlowResponseService.resolveFlow` learns a second token scheme: if it starts with `c:`,
   load the `campaign_contact` → derive `flow_id` from its campaign and set `flow_responses.flow_id` +
   (future) a campaign linkage. Keep the existing UUID-token path for `FlowService.send`.
3. **Reuse the existing engine.** No new queue or consumer. `BroadcastMessage` already carries enough
   (`campaignId, contactId, phoneNumberId, templateId`); the consumer re-loads the campaign and sees
   `flow_id`, so the per-recipient token is computed in `CampaignService.deliver` from the
   `campaign_contact.id` it already has. `BroadcastMessage` stays unchanged.
4. **Billing unchanged.** A flow-bearing template is still one template message → existing
   `walletService.debitForMessage(...)` by template category applies as-is.
5. **§13 rules still hold.** Unsubscribed filtered (rule 1, already in `deliver`), only APPROVED templates
   (rule 3, enforced by decision 1), workspace isolation (rule 8).

## Build order (per §11)
1. Migration `V7__campaign_flow.sql` (`campaigns.flow_id`, `campaigns.flow_cta`).
2. `Campaign` entity: add `flowId`, `flowCta`. `CampaignResponse`/`CreateCampaignRequest`/`UpdateCampaignRequest` DTOs.
3. `MetaMessageClient.sendTemplate` overload (or new param) that appends the flow **button component** with a
   `flow_token`. Keep the old signature for non-flow campaigns/replies. `MetaMessageClientImpl.buildTemplate`
   gains an optional `(flowToken)` → adds the `button`/`sub_type:flow` component shown above.
4. `CampaignService`:
   - `create`/`update`: when `flowId` present, run the §1 validation (published flow + template has matching
     APPROVED FLOW button).
   - `deliver`: if `campaign.getFlowId() != null`, compute `flowToken = "c:" + cc.getId()` and call the new
     send path; else unchanged.
5. `FlowResponseService.resolveFlow`: handle the `c:<campaignContactId>` token scheme (load campaign_contact
   → flow_id); existing UUID scheme stays for inbox sends. Optionally stamp the originating campaign on the
   `flow_responses` row (add `campaign_id` column in the same migration if we want campaign-level reporting).
6. Frontend: campaign create/edit — optional "Attach flow" (published-flows dropdown) + CTA label; a
   template picker filtered to templates that *have a FLOW button*. Surface flow responses per campaign
   (reuse the existing `/api/flows/{id}/responses`, optionally filterable by campaign once §5 stamps it).

## Tests
- `MetaMessageClientImpl`: `sendTemplate` with a flow token emits the `button`/`sub_type:flow`/`flow_token`
  component; without a token the payload is byte-for-byte the current one (no regression).
- `CampaignService.create`: attaching an unpublished flow → 409; template without a FLOW button → 409.
- `CampaignService.deliver`: flow campaign computes `c:<ccId>` token and calls the flow send path; counters +
  wallet debit unchanged.
- `FlowResponseService`: a `c:<campaignContactId>` token resolves to the right flow and writes one
  `flow_responses` row; dedup on `wa_message_id` still holds (rule 5).

## Open questions
1. Add `flow_responses.campaign_id` now for campaign-level flow reporting, or defer until reporting is asked for?
   (Recommend: add the nullable column in `V7` while we're here — cheap, avoids a later migration.)
2. Template authoring UI for FLOW buttons — do we add first-class support in the template editor, or rely on
   the raw `components` JSON the editor already accepts? (Recommend: raw JSON first; first-class later.)
3. Should attaching a flow auto-set the campaign's follow-up behavior, or keep follow-up purely a per-flow
   concern (`wa_flows.followup_template_id`, already built)? (Recommend: keep them independent.)
```
