# Setup Walkthrough

This document covers the hands-on wiring required to go from "JSON imported" to "workflow activated and firing correctly." It assumes you've already read the [main README](../README.md).

## 1. Import the workflow

1. n8n editor → top-right menu → **Import from file**.
2. Pick one of the JSON files under `workflows/`.
3. The editor will open the imported workflow. Don't activate yet — the credentials and placeholders still need to be filled in.

## 2. Configure credentials

Every HTTP Request node that talks to GHL uses **Header Auth** with a GHL Private Integration Token. Create the credential once and assign it to every GHL node.

1. Create the GHL PIT: in GHL → Settings → Private Integrations → Create PIT with scopes:
   - `contacts.readonly`, `contacts.write`
   - `contacts/tags.write`
   - `conversations/messages.write`
   - `locations.readonly`
2. In n8n: Credentials → **Header Auth** → create new. Name it "GHL PIT".
   - Header name: `Authorization`
   - Header value: `Bearer pit-xxxxxxxxxxxxxxxx` (paste your PIT)
3. Open every HTTP Request node in the imported workflow. Any node that POSTs to `services.leadconnectorhq.com/...` should use this credential. The JSON has `{{CREDENTIAL_ID}}` / `{{CREDENTIAL_NAME}}` placeholders — you don't need to touch those, just pick your credential from the dropdown in the node UI.

## 3. Replace placeholders

Use your editor's find-and-replace on the JSON file **before** importing (or search-and-replace inside n8n's node parameters after importing). The full list of placeholders is in the main README table.

The bare minimum you must replace before the workflow will run:

- `{{GHL_LOCATION_ID}}`
- `{{GHL_CALENDAR_ID}}` (inbound template only)
- `{{VOICE_API_URL}}` and `{{VOICE_AGENT_PROMPT_ID}}` (outbound caller template only)
- `{{OUTBOUND_FROM_PHONE}}` (outbound caller template only)
- `{{BOOKING_URL}}`
- `{{COMPANY_NAME}}`
- `{{SENDER_NAME}}`, `{{SENDER_TITLE}}`
- `{{RECEPTIONIST_NAME}}`
- `{{DEMO_PHONE}}` (nurture template only — referenced in SMS 3 copy)

## 4. Create the required GHL tags

In GHL → Settings → Tags → Add:

- `ai_sdr_active`
- `send_demo_link`
- `not_interested`
- `appointment_booked`

Some of these are read by the workflows; some are written. If you skip one, the workflow will still run but the corresponding branch won't trigger.

## 5. Wire the webhooks

### Outbound caller (`ai-sdr-outbound-caller.json`)

Trigger node: **New Lead Webhook**. Expects a POST body with snake_case fields from GHL's `ContactTagUpdate` workflow action. Payload shape:

```json
{
  "contact_id": "abc123",
  "first_name": "Jane",
  "phone": "+15555555555",
  "email": "jane@example.com",
  "tag": "ai_sdr_active"
}
```

In GHL → Automation → Workflows → create a workflow:

- Trigger: **Contact Tag** → added → tag `ai_sdr_active`
- Action: **Webhook** → POST → URL = your n8n webhook URL (copy from the trigger node in n8n after activating)
- Payload: pass the contact fields above

### Post-call nurture (`ai-sdr-post-call-nurture.json`)

Trigger node: **Post-Call Webhook**. Expects a POST from your voice provider after a call completes. Payload shape:

```json
{
  "conversationId": "t12yi4wl5rx2orjqrgau6i23",
  "humanPhoneNumber": "+15555555555",
  "tags": ["not_interested"],
  "variables": {
    "origin": "workflow1",
    "wave": "1",
    "firstName": "Jane",
    "contactId": "abc123"
  }
}
```

The workflow gates on `variables.origin === "workflow1"` — only calls that originated from the outbound caller template enter the nurture flow. Inbound calls fall through to a no-op.

Configure your voice provider's post-call webhook URL to point at the n8n webhook URL (copy from the trigger node after activating).

### AI receptionist inbound (`ai-receptionist-inbound.json`)

Trigger node: **Post-Call Webhook**. Shares the same webhook shape as the nurture template above — the two can run on the same post-call URL if you merge them, or on separate URLs if you keep them apart.

This template reads `tags` from the post-call payload and checks for `send_demo_link`. If present, it upserts the contact in GHL (by phone) with source `{{COMPANY_NAME}} Inbound` and sends the booking link SMS.

## 6. Test before going live

1. Create a test contact in GHL with your own cell phone number (never a real prospect).
2. Add the `ai_sdr_active` tag to kick off the outbound caller. You should see the workflow fire and a call attempt land on your phone.
3. After the call completes, check that the nurture workflow fires and you receive Email 1 and SMS 1 (one minute apart).
4. For the inbound template, call your demo line and ask for a demo link. You should receive the SMS from `{{COMPANY_NAME}}` pointing at `{{BOOKING_URL}}`.
5. Verify the new contact exists in GHL with the correct tags and source.

## Troubleshooting

- **"Contact with id ... not found"** — your workflow is passing the wrong ID to GHL's `/conversations/messages` endpoint. Check that every node references `$('Post-Call Webhook').first().json.body?.variables?.contactId`, not `$json.body?.conversationId`.
- **"invalid syntax" from n8n expression parser** — n8n's expression pre-parser (tournament.js) chokes on escaped apostrophes inside single-quoted JS strings. Rewrite the body using template literals (backticks) instead.
- **SMS never arrives** — check A2P 10DLC registration status in Twilio. Without an approved campaign, US carriers silently filter A2P traffic.
- **Outbound call fails auth** — the `Call Lead` node uses its own credential for your voice provider. It's separate from the GHL PIT.
