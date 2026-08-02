# FAST-NUCES Islamabad — WhatsApp Admission Bot

A fully local admission assistant for **FAST-NUCES Islamabad**, built with:

- **WhatsApp Cloud API** — student-facing channel
- **n8n (local/self-hosted)** — workflow engine
- **Ollama (local LLM)** — intent classification + natural language reply generation
- **Google Sheets** — knowledge base (9 sheets, editable by non-technical staff)

---

## 1. What's in this package

| File | Purpose |
|---|---|
| `n8n_whatsapp_admission_bot.json` | The complete n8n workflow, ready to import |
| `sheets/1_General_Info.csv` → `sheets/9_Contacts.csv` | Dummy knowledge-base data for each Google Sheet tab |
| `README.md` | This guide |

---

## 2. Prerequisites

- n8n running locally (Docker or `npm install -g n8n`), version **1.6x or newer** (uses Switch v3 / Set v3.4 / HTTP Request v4.2 node schemas).
- Ollama running locally, with a model pulled, e.g.:
  ```bash
  ollama pull llama3.1
  ollama serve
  ```
  Verify it works: `curl http://localhost:11434/api/tags`
- A Meta developer account with a WhatsApp Cloud API test/production number.
- A Google account with the Google Sheets API enabled and OAuth2 credentials.
- If your n8n instance and WhatsApp/Meta need to talk to each other, your n8n webhook URL must be **publicly reachable** (use `ngrok`, `cloudflared`, or a VPS reverse proxy — WhatsApp cannot call `localhost` directly).

---

## 3. Step 1 — Create the Google Sheet

1. Create a new Google Sheet named e.g. **FAST-NUCES Admission Bot KB**.
2. Create **9 tabs**, named exactly as below (names are case-sensitive and referenced in the workflow):
   - `General_Info`
   - `FAQs`
   - `Degree_Programs`
   - `University_Policies`
   - `Timetable`
   - `Scholarships`
   - `Faculty`
   - `Academic_Calendar`
   - `Contacts`
3. Import the matching CSV from the `sheets/` folder into each tab (**File → Import → Upload**, choose "Replace current sheet", for each tab).
4. Copy the **Spreadsheet ID** from the sheet's URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

---

## 4. Step 2 — Import the workflow into n8n

1. Open n8n → **Workflows → Import from File** → select `n8n_whatsapp_admission_bot.json`.
2. The workflow has **28 nodes** in two branches:
   - **GET branch** — handles Meta's webhook verification handshake.
   - **POST branch** — handles incoming WhatsApp messages end-to-end.

### 4.1 Configure the `Config` and `Config - Verify` nodes

Open each **Set** node named `Config` and `Config - Verify` and replace the placeholder values:

| Field | Where to get it |
|---|---|
| `verify_token` (Config - Verify) | Any string you choose — you'll enter the same value in Meta's dashboard |
| `whatsapp_token` | Meta App Dashboard → WhatsApp → API Setup → Temporary/Permanent access token |
| `whatsapp_phone_number_id` | Meta App Dashboard → WhatsApp → API Setup → Phone number ID |
| `whatsapp_api_version` | Defaults to `v20.0` — update if Meta deprecates it |
| `ollama_url` | Defaults to `http://localhost:11434` — change if Ollama runs on another host/port |
| `ollama_model` | Defaults to `llama3.1` — must match a model you've pulled with `ollama pull` |
| `google_sheet_id` | The Spreadsheet ID from Step 3.4 |

> Keeping all config in one Set node means you never have to hunt through 28 nodes to change a token later.

### 4.2 Connect Google Sheets credentials

Every `GSheet - *` node (9 total) currently has a placeholder credential. For **each** of them:
1. Open the node → Credential dropdown → **Create New Credential** (first time only) → sign in with the Google account that owns the sheet → OAuth consent.
2. Reuse the same credential for the remaining 8 nodes.
3. Confirm `Document` → select the KB spreadsheet from the dropdown (this overrides the placeholder ID and is safer than editing the expression directly, though the `google_sheet_id` in `Config` also works if left as an ID expression).

### 4.3 Activate the workflow

Toggle **Active** in the top-right of the n8n canvas. Note the **Production Webhook URL** for the `Webhook - Incoming (POST)` node (also shared by the GET node, same path `/whatsapp-webhook`).

---

## 5. Step 3 — Connect WhatsApp Cloud API to n8n

1. In Meta App Dashboard → WhatsApp → **Configuration**:
   - **Callback URL**: your public n8n webhook URL, e.g. `https://your-domain-or-ngrok.io/webhook/whatsapp-webhook`
   - **Verify Token**: the exact same string you put in `Config - Verify → verify_token`
2. Click **Verify and Save**. Meta sends a GET request; the workflow's GET branch checks the token and echoes back `hub.challenge`. A green checkmark confirms success.
3. Under **Webhook fields**, subscribe to `messages`.
4. Send a test WhatsApp message to your business number — it should trigger the POST branch.

---

## 6. How the workflow works (POST branch)

```
Webhook (POST) → Config → Extract WhatsApp Message
   → IF Has Text Message
        NO  → Respond 200 (ignored, e.g. delivery/read status callbacks)
        YES → Ollama: Classify Intent (returns JSON: intent + entities)
              → Parse Intent (safe JSON parse, defaults to "fallback" on error)
              → Switch: Route Intent
                   ├── general      → GSheet: General_Info
                   ├── faq          → GSheet: FAQs (filtered by keyword)
                   ├── degree       → GSheet: Degree_Programs (filtered by degree)
                   ├── policy       → GSheet: University_Policies
                   ├── timetable    → GSheet: Timetable (filtered by department)
                   ├── scholarship  → GSheet: Scholarships
                   ├── faculty      → GSheet: Faculty (filtered by department)
                   ├── calendar     → GSheet: Academic_Calendar
                   ├── contacts     → GSheet: Contacts
                   └── fallback     → (skips sheet lookup)
              → Build Context (formats sheet rows into plain text, or "NO_DATA_FOUND")
              → Ollama: Format Reply (turns raw data into a natural WhatsApp reply)
              → Parse Reply (falls back to a safe apology message if Ollama fails)
              → Send WhatsApp Message (Graph API)
              → Respond 200 (success)
```

### Built-in fault tolerance
- If **Ollama is unreachable** during intent classification, `Parse Intent` catches the failure and routes to `fallback` instead of breaking the workflow.
- If **Ollama is unreachable** during reply generation, `Parse Reply` substitutes a safe default message pointing the student to the Admissions Office.
- If **no matching row** is found in a Google Sheet, `Build Context` sets `NO_DATA_FOUND`, and the reply-generation prompt is instructed to acknowledge that gracefully instead of hallucinating.
- WhatsApp **status callbacks** (message delivered/read receipts, which have no `messages` array) are detected and safely ignored instead of crashing the workflow.

---

## 7. Testing checklist before going live

- [ ] `curl http://localhost:11434/api/generate -d '{"model":"llama3.1","prompt":"hi","stream":false}'` returns a response
- [ ] Webhook GET verification succeeds in Meta dashboard (green check)
- [ ] Send "What programs do you offer?" → expect `general`/`degree` style answer
- [ ] Send "What is the admission fee?" → expect FAQ answer from sheet
- [ ] Send "Can I apply with 2.6 CGPA in BS Computer Science?" → expect degree-eligibility answer
- [ ] Send "Monday timetable for CS semester 5" → expect timetable rows
- [ ] Send a random/off-topic message → expect graceful fallback reply, not an error
- [ ] Turn off Ollama and send a message → confirm the bot still replies with the safe fallback text instead of hanging or erroring

---

## 8. Known limitations (be aware of these)

- **Response latency**: the workflow calls Ollama twice (classify, then generate) synchronously before replying, and WhatsApp does not require an immediate ack on the message webhook itself — but if your model/host is slow (>15–20s), consider a smaller/quantized model, or splitting this into a "quick ack + async send" pattern with a queue (e.g. n8n sub-workflow triggered via Redis/Postgres) for higher scale.
- **Google Sheets filter node parameters** (`filtersUI` / lookup fields) can differ slightly between n8n versions. If a `GSheet - *` node doesn't return the expected rows after import, open it once in the n8n UI — the node will show any parameter that needs to be re-picked from a dropdown (this is normal for Google Sheets nodes after JSON import, since sheet/column pickers are account-specific and can't be pre-filled from outside your n8n instance).
- **Ollama JSON reliability**: smaller/older local models occasionally return malformed JSON for the intent-classification step. The `Parse Intent` node already guards against this (falls back to `fallback` intent), but if you see this often, switch to a stronger local model (e.g. `llama3.1:8b` or larger) or add few-shot examples to the prompt.
- This package has been validated for **JSON correctness and internal node wiring** (all 28 nodes connected, no dangling references). It has **not** been executed against a live n8n/Ollama/WhatsApp/Google stack from this environment, since that requires your own credentials and a publicly reachable webhook. Run through the Section 7 checklist after import to confirm end-to-end behavior in your environment.

---

## 9. Extending the bot

- Add a new intent: add a row of dummy conditions to `Switch - Route Intent`, add a new Google Sheets tab + CSV, add a branch, wire it into `Build Context`, and update the intent list inside the `Ollama - Classify Intent` prompt.
- Add multi-language support (English/Urdu): adjust the `Ollama - Format Reply` prompt to detect and mirror the student's language.
- Log every conversation: add a Google Sheets "Append Row" node after `Parse Reply` writing to a `Conversation_Log` tab (from, text, intent, reply, timestamp).
