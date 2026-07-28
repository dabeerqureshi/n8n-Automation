# ABC Hotel WhatsApp Booking Bot — Setup Guide

Local n8n + local Ollama + Google Sheets + Gmail + WhatsApp Business Cloud API.

**Please read "Honest limitations" at the bottom before you start.** I built and validated this
workflow's JSON structure and logic on paper, but I don't have a live WhatsApp Business
account, Google account, or your local Ollama instance to test against — so treat this as a
strong, working starting point that you'll verify end-to-end in your own n8n editor, not a
guarantee of zero bugs.

---

## 1. What's included

| File | Purpose |
|---|---|
| `hotel_booking_workflow.json` | The n8n workflow — import this into n8n |
| `sheet1_Rooms.csv` | Rooms sheet template |
| `sheet2_Bookings.csv` | Bookings sheet template (headers only) |
| `sheet3_FAQs.csv` | FAQs sheet template |
| `sheet4_Sessions.csv` | Conversation-memory sheet (headers only, required for the bot to remember context between messages) |
| `README.md` | This guide |

---

## 2. Architecture (what actually happens)

```
Meta WhatsApp Cloud API → Webhook (n8n) → Extract message
   → Load Rooms + FAQs + Session (Google Sheets)
   → State Router (Code node — deterministic conversation state machine)
   → (if needed) Ollama — for free-text date/guest extraction and small talk
   → Finalize reply + booking actions
   → Save session → (if booking confirmed) Append booking, mark rooms Booked,
     email customer + manager → Send WhatsApp reply
```

The conversation state (what step each customer is on) is stored per-phone-number in the
**Sessions** sheet as a small JSON blob. This is what lets the bot "remember" a booking in
progress across separate WhatsApp messages. Ollama is used only where free-text understanding
is genuinely needed (extracting dates/guests, or general small talk/FAQ phrasing) — the actual
booking flow, validation, and Google Sheets writes are handled deterministically in n8n so it
doesn't misfire mid-booking.

---

## 3. Prerequisites

- **n8n** running locally (Docker or `npm install n8n -g`), version 1.7x or newer recommended.
- **Ollama** installed locally with a model pulled, e.g.:
  ```
  ollama pull llama3.1
  ```
- A **Google Cloud** project with the Google Sheets API and Gmail API enabled, and OAuth2
  credentials (Client ID/Secret) for n8n's Google Sheets and Gmail nodes.
- A **Meta developer app** with WhatsApp Business Cloud API access, a test/production phone
  number, and a permanent access token.
- A way for Meta to reach your local n8n webhook — see step 6 (ngrok/Cloudflare Tunnel), since
  Meta cannot call `localhost` directly.

---

## 4. Step 1 — Google Sheet

1. Create one new Google Sheet, e.g. "ABC Hotel Bot".
2. Create four tabs named **exactly**: `Rooms`, `Bookings`, `FAQs`, `Sessions`.
3. Import each CSV into the matching tab (File → Import → Upload → Insert new sheet, or paste
   the CSV rows directly), keeping the header row exactly as given — the workflow's column
   mappings depend on these exact header names.
4. Copy the Sheet's ID from its URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

---

## 5. Step 2 — Import the workflow into n8n

1. Open n8n → **Workflows → Import from File** → select `hotel_booking_workflow.json`.
2. Every Google Sheets node currently has a placeholder document ID
   (`REPLACE_WITH_ROOMS_SPREADSHEET_ID`). Open each of the 7 Google Sheets nodes (`Get Rooms`,
   `Get FAQs`, `Get Session`, `Save Session`, `Append Booking`, `Update Room Status`) and:
   - Select/reconnect your Google Sheets credential.
   - Paste your Sheet ID into "Document" (or pick it from the dropdown once connected).
   - Confirm the Sheet tab name in "Sheet" matches (`Rooms`, `FAQs`, `Sessions`, `Bookings`).
   - For nodes with a column mapper ("Save Session", "Append Booking", "Update Room Status"),
     n8n needs to fetch your sheet's real columns the first time — open the node once, let it
     load, and confirm the mapped fields line up (they should auto-match by header name).
3. Open both **Gmail** nodes and reconnect your Gmail OAuth2 credential.
4. This is normal, expected n8n behavior after any workflow import — credentials and
   sheet/column references are always account-specific and can't be exported.

---

## 6. Step 3 — Environment variables

Set these where n8n runs (`.env` file, `docker-compose.yml` environment block, or your shell
before `n8n start`):

```
WHATSAPP_VERIFY_TOKEN=choose-any-random-string
WHATSAPP_TOKEN=your-meta-permanent-access-token
WHATSAPP_PHONE_NUMBER_ID=your-meta-phone-number-id
MANAGER_EMAIL=manager@yourhotel.com
HOTEL_NAME=ABC Hotel
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=llama3.1
```

**If n8n runs in Docker**, `localhost` inside the container is the container itself, not your
host machine — Ollama running on your host won't be reachable. Use:
```
OLLAMA_URL=http://host.docker.internal:11434
```
(on Linux you may instead need `--add-host=host.docker.internal:host-gateway` on the n8n
container, or use the host's LAN IP).

n8n must be allowed to read environment variables in expressions — this is the default for
self-hosted n8n (`N8N_BLOCK_ENV_ACCESS_IN_NODE` is unset/false).

---

## 7. Step 4 — Expose your webhook to Meta

Meta needs a public HTTPS URL. Locally, use a tunnel:

```
ngrok http 5678
```

Your webhook URLs will be:
- Verify/incoming: `https://<your-ngrok-domain>/webhook/whatsapp-hotel`

---

## 8. Step 5 — Configure the Meta webhook

1. In Meta App Dashboard → WhatsApp → Configuration:
   - Callback URL: `https://<your-ngrok-domain>/webhook/whatsapp-hotel`
   - Verify Token: the same string you set as `WHATSAPP_VERIFY_TOKEN`.
2. Click **Verify and Save** — this triggers the `Webhook - Verify` (GET) node, which checks the
   token and returns Meta's `hub.challenge`.
3. Subscribe to the `messages` webhook field.

---

## 9. Step 6 — Activate and test

1. Activate the workflow in n8n (toggle top-right).
2. From a WhatsApp-registered test number, message your bot's number with "Hi".
3. Expected: welcome message → you provide guests/dates → bot lists available rooms → you pick
   room ID(s) → bot asks name, CNIC, phone, email, address one at a time → summary → reply
   `YES` → booking is saved, rooms marked Booked, two emails sent, WhatsApp confirmation sent.
4. Watch n8n's **Executions** log for each step — this is the fastest way to catch and fix any
   field-mapping issue.

---

## 10. Conversation states reference

`START → ASK_DATES → SHOWING_ROOMS → ASK_NAME → ASK_CNIC → ASK_PHONE → ASK_EMAIL → ASK_ADDRESS → CONFIRMING → DONE`

FAQs are matched at `START`/`DONE` by simple keyword match against the `FAQs` sheet, before
anything else — so a guest can ask "what time is checkout" any time they're not mid-booking.

---

## 11. Honest limitations (please read)

- I generated and validated the workflow's **JSON structure and internal logic** (all 27 nodes
  connect correctly, IF-branch true/false paths were checked programmatically). I could not run
  it against live Meta, Google, or your local Ollama endpoints, so I can't certify "zero bugs" —
  no one honestly can without a live test pass.
- Google Sheets node field names (`filtersUI`, column mapper) were built against n8n's current
  public documentation and source, but the column-mapper UI in your specific n8n version may
  need you to re-select fields once, which is normal for any imported workflow.
- The state machine is intentionally rule-based (not fully AI-driven) for reliability — Ollama
  only extracts free-text info, it never decides Google Sheets writes. This trades a bit of
  conversational flexibility for predictability, which matters more for something that touches
  bookings and money.
- Test with a WhatsApp test number first, and check the n8n Executions panel after each message
  before going live.

If something breaks after import, the most common causes are: wrong Sheet tab name, Gmail/Sheets
credential not reconnected, or `OLLAMA_URL` unreachable from where n8n runs — check those three
first.
