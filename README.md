# Telvana Workflow Templates

Ready-to-import [n8n](https://n8n.io) workflows for running an AI phone receptionist and a 5-wave outbound calling sequence on top of [GoHighLevel](https://www.gohighlevel.com) (GHL).

Import, fill in a few placeholders, wire up a webhook, and you have a working AI SDR + inbound demo-link flow.

## What's included

| Template | What it does |
|---|---|
| [`ai-receptionist-inbound.json`](workflows/ai-receptionist-inbound.json) | Receives a post-call webhook from an AI receptionist. If the caller asked for a demo link, upserts the contact into GHL and sends a branded booking-link SMS. |
| [`ai-sdr-outbound-caller.json`](workflows/ai-sdr-outbound-caller.json) | 5-wave outbound calling sequence. Triggered by a `ContactTagUpdate` webhook when a lead is tagged active in GHL. Fires a voice call through your outbound API, waits, calls again across 5 waves, then removes the active tag. |
| [`ai-sdr-post-call-nurture.json`](workflows/ai-sdr-post-call-nurture.json) | Post-call follow-up for the 5-wave sequence above. Sends a per-wave email, waits 1 minute, sends a per-wave SMS. Skips if the call ended with `not_interested` or `appointment_booked`. |

## Prerequisites

- **n8n** — Cloud or self-hosted, v1.x or later.
- **GoHighLevel (GHL)** sub-account — for CRM, contact storage, and messaging.
- **GHL Private Integration Token (PIT)** with `contacts.read/write`, `contacts/tags.write`, and `conversations/messages.write` scopes.
- **An outbound voice API** — these templates assume a JSON POST like `{ from, to, outboundPromptId, variables }` returns a call. Telvana is one option; you can plug in any provider that matches that shape.
- **Twilio A2P 10DLC registration** (for US SMS) — required by carriers. Brand + Campaign registration takes 1–5 days.

## Quick start

1. Import the JSON into your n8n instance (editor → top-right menu → Import from file).
2. Open every HTTP Request node and re-attach credentials (they're stripped for safety — look for "Authentication: Header Auth" nodes with missing credential IDs).
3. Search the workflow for `{{` and replace every placeholder with your real values. See [`docs/setup.md`](docs/setup.md) for the full list.
4. Activate the workflow(s) you want and wire the webhook URLs into GHL (for the outbound caller and nurture) and your voice provider (for the inbound template).
5. Test end-to-end before you flip traffic over. Use a test contact you control — never test against real prospects.

## Placeholder reference

| Placeholder | What to fill in |
|---|---|
| `{{GHL_LOCATION_ID}}` | Your GHL sub-account location ID (Settings → Business Profile). |
| `{{GHL_CALENDAR_ID}}` | The GHL calendar ID for your demo/booking calendar. |
| `{{CREDENTIAL_ID}}` / `{{CREDENTIAL_NAME}}` | n8n credential object — replaced on import when you re-attach credentials in the editor. |
| `{{VOICE_API_URL}}` | Your outbound voice provider's API endpoint. |
| `{{VOICE_AGENT_PROMPT_ID}}` | The ID of your outbound calling prompt / agent. |
| `{{OUTBOUND_FROM_PHONE}}` | E.164 phone number your voice provider dials from. |
| `{{BOOKING_URL}}` | The URL you want SMSs to point at for booking (e.g. `https://yourdomain.com/demo`). |
| `{{DEMO_PHONE}}` | A phone number callers can dial to experience your AI receptionist live (optional copy). |
| `{{COMPANY_NAME}}` | Your company name, used in SMS and email copy. |
| `{{SENDER_NAME}}` | The human name on outbound emails/SMS (e.g. the founder or SDR). |
| `{{SENDER_TITLE}}` | Sender's title (e.g. "CEO"). |
| `{{RECEPTIONIST_NAME}}` | The name of your AI receptionist (e.g. "Taylor"). |
| `{{AUTO_GENERATED_ON_IMPORT}}` | n8n regenerates the webhookId on import — ignore. |

See [`docs/setup.md`](docs/setup.md) for the step-by-step walkthrough including the GHL custom fields and tags the workflows expect.

## Tags and custom fields the workflows use

These templates read and write specific GHL tags. Add them to your GHL account before running:

- **`ai_sdr_active`** — set to start the 5-wave outbound sequence, removed when the sequence finishes.
- **`send_demo_link`** — set by the inbound receptionist workflow when a caller asks for a demo link.
- **`not_interested`** — set by your voice agent when the call ends with a "not interested" signal. Stops the nurture.
- **`appointment_booked`** — set when a demo is booked. Stops the nurture.

## Limitations and disclaimers

- These templates are provided as-is. Your environment will be different — expect to debug.
- The email and SMS copy is a starting point. You'll want to rewrite it in your own voice.
- The outbound voice API shape (`from`, `to`, `outboundPromptId`, `variables`) matches [Telvana](https://telvana.com). If you use a different provider, you'll need to rewrite the HTTP body in the `Call Lead` nodes.
- A2P 10DLC is required for US SMS. These templates do not handle compliance for you.

## Contributing

PRs welcome. If you find a bug in one of the templates, open an issue with your n8n version and what you saw.

## License

[MIT](LICENSE). Use them in commercial products, fork them, rebrand them, whatever helps.
