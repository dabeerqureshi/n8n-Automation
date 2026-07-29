# WhatsApp Real Estate Assistant — Local n8n + Local Ollama

A self-hosted WhatsApp bot that answers property questions, searches listings, answers FAQs, and books site visits — using **local n8n**, **local Ollama**, and **local CSV files** instead of Google Sheets. No cloud AI, no Google account required.

## Files in this package

| File | Purpose |
|---|---|
| `whatsapp-real-estate-bot.json` | The n8n workflow — import this into n8n |
| `Properties.csv` | Property listings database (10 sample rows included) |
| `Appointments.csv` | Site-visit bookings ledger (starts empty, header only) |
| `FAQs.csv` | Question/answer pairs the bot can draw on |
| `README.md` | This guide |

---

## 1. How it works (architecture)

```
Customer (WhatsApp)
      │
      ▼
n8n Webhook  ──►  Parse message  ──►  Load customer memory (JSON file per phone number)
      │
      ▼
Ollama (local LLM) — understands intent + extracts details
      │
      ▼
Decision logic (Code node) ──► Properties.csv  /  FAQs.csv
      │
      ▼
Ollama (local LLM) — writes a natural reply
      │
      ▼
Send WhatsApp reply
      │
      ▼
If customer wants a visit and all details are collected:
   → append row to Appointments.csv
   → email customer + admin (SMTP)
   → send WhatsApp confirmation
      │
      ▼
Save updated conversation memory
```

Every customer's conversation state is stored as a small JSON file (`memory/<phone>.json`) on disk, so the bot remembers context between messages without any external database.

---

## 2. Prerequisites

- **n8n** running locally (Docker or `npm install n8n -g`)
- **Ollama** running locally with a model pulled, e.g.:
  ```bash
  ollama pull qwen3:4b
  # or
  ollama pull gemma3:4b
  ```
- A **Meta WhatsApp Business Cloud API** app (WhatsApp still requires Meta's API — this is the one component that cannot be fully local, since WhatsApp itself is a hosted service). You need:
  - A temporary or permanent access token
  - A Phone Number ID
  - A webhook verify token (you choose this string yourself)
- An SMTP account for sending emails (Gmail SMTP with an app password works fine, or any local SMTP relay like `msmtp`/`mailhog` for testing)
- **n8n must be reachable from the internet** for Meta to deliver webhooks — use a tunnel like `ngrok` or `cloudflared` if running purely on localhost.

---

## 3. Folder setup (on the machine running n8n)

Create this structure and place the CSV files inside it:

```
/data/whatsapp-bot/
├── Properties.csv
├── Appointments.csv
├── FAQs.csv
└── memory/          ← leave empty, the bot creates files here automatically
```

If running n8n in Docker, mount this folder as a volume, e.g. in `docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - WHATSAPP_TOKEN=your_meta_token
      - WHATSAPP_PHONE_NUMBER_ID=your_phone_number_id
      - WHATSAPP_VERIFY_TOKEN=choose_a_secret_string
      - OLLAMA_URL=http://host.docker.internal:11434
      - OLLAMA_MODEL=qwen3:4b
      - DATA_DIR=/data/whatsapp-bot
      - ADMIN_EMAIL=agent@yourcompany.com
      - SMTP_FROM_EMAIL=bot@yourcompany.com
    volumes:
      - ./data:/data/whatsapp-bot
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

If Ollama runs on the host machine (not in Docker) and n8n runs in Docker, use `http://host.docker.internal:11434` for `OLLAMA_URL`. If both run directly on the same machine (no Docker), use `http://localhost:11434`.

> **Important:** n8n's Code nodes are sandboxed and cannot read/write files directly. All file access in this workflow goes through n8n's built-in **Read/Write File** and **Spreadsheet File** nodes instead, which is what keeps this 100% local without extra dependencies.

---

## 4. Environment variables required

Set these wherever you run n8n (`.env` file, Docker Compose, or your OS environment):

| Variable | Example | Description |
|---|---|---|
| `WHATSAPP_TOKEN` | `EAAG...` | Meta Cloud API access token |
| `WHATSAPP_PHONE_NUMBER_ID` | `123456789012345` | Your WhatsApp Business phone number ID |
| `WHATSAPP_VERIFY_TOKEN` | `myS3cretToken` | Any string you choose; used during webhook verification |
| `OLLAMA_URL` | `http://localhost:11434` | Base URL of your local Ollama server |
| `OLLAMA_MODEL` | `qwen3:4b` | Model name as shown by `ollama list` |
| `DATA_DIR` | `/data/whatsapp-bot` | Folder holding the CSVs and `memory/` subfolder |
| `ADMIN_EMAIL` | `agent@yourcompany.com` | Where new-appointment notifications go |
| `SMTP_FROM_EMAIL` | `bot@yourcompany.com` | "From" address for outgoing emails |

---

## 5. Import and configure the workflow

1. Open n8n → **Workflows → Import from File** → select `whatsapp-real-estate-bot.json`.
2. Open the two **Email** nodes ("Email Customer", "Email Admin") and attach your SMTP credential (**Credentials → New → SMTP**) — this is the one piece n8n requires as a saved credential rather than an env var.
3. Double-check the **Read/Write File** and **HTTP Request** nodes point to the right paths/URLs — they're all driven by the environment variables above, so normally nothing to edit here.
4. **Activate** the workflow.
5. Copy the production webhook URL from the **"WhatsApp Incoming (POST)"** node (something like `https://your-domain/webhook/whatsapp-webhook`).

---

## 6. Connect Meta WhatsApp Cloud API

1. In [Meta Developer Console](https://developers.facebook.com/) → your app → WhatsApp → Configuration.
2. Set **Callback URL** to your n8n webhook URL (same path used for both GET verification and POST messages).
3. Set **Verify Token** to the same value you put in `WHATSAPP_VERIFY_TOKEN`.
4. Subscribe to the `messages` webhook field.
5. Send a test WhatsApp message to your business number — the flow should trigger end-to-end.

---

## 7. Customer journey this workflow implements

1. Customer messages "Hi" → bot greets and asks what they're looking for.
2. Customer describes requirements (e.g. "1 kanal plot in DHA Phase 2") → Ollama extracts filters → bot searches `Properties.csv` → returns up to 5 matches, written in natural language.
3. Customer asks a general question (transfer charges, foreigners buying, etc.) → bot matches it against `FAQs.csv` and answers in its own words.
4. Customer says they want a visit → bot asks for name, phone, email, preferred date/time (only for whatever is still missing — it remembers what's already been given).
5. Once all details are collected → bot:
   - Generates an Appointment ID (`APT-00001`, `APT-00002`, …)
   - Appends the row to `Appointments.csv`
   - Emails the customer and the admin
   - Sends a WhatsApp confirmation message
6. Conversation memory resets to "chatting" so the customer can keep chatting or start a new search.

---

## 8. Testing without a live WhatsApp number

You can test the "brain" of the bot without WhatsApp by POSTing a fake payload directly to the incoming webhook:

```bash
curl -X POST https://your-domain/webhook/whatsapp-webhook \
  -H "Content-Type: application/json" \
  -d '{
    "entry": [{
      "changes": [{
        "value": {
          "contacts": [{"profile": {"name": "Test User"}}],
          "messages": [{
            "from": "923001234567",
            "id": "wamid.TEST123",
            "timestamp": "1234567890",
            "type": "text",
            "text": {"body": "I need a 1 kanal plot in DHA Phase 2"}
          }]
        }
      }]
    }]
  }'
```

Check the n8n execution log to see the full trace, and check `Properties.csv`/`Appointments.csv` for changes.

---

## 9. Notes, limits & things to verify on your setup

- Node parameter names for **Read/Write File** and **Spreadsheet File** vary slightly between n8n versions. If import shows a "missing parameter" warning on any of those nodes, open the node, and n8n will usually auto-map it — just confirm the file path field once.
- The Ollama "understand message" call asks the model to return JSON (`format: "json"`). Smaller models occasionally return malformed JSON; the workflow already falls back to a safe default (`intent: "other"`) in that case rather than failing.
- WhatsApp itself is Meta's hosted service — there's no way to run WhatsApp messaging fully offline. Everything else (AI, database, memory, logic) is 100% local.
- This is a working starting point sized for moderate traffic (single-CSV lookups). For high volume, swap `Properties.csv`/`Appointments.csv` for SQLite via n8n's SQLite node — the rest of the logic stays the same.
- Sample `Properties.csv` includes 10 placeholder listings and `FAQs.csv` includes 8 sample Q&As — replace with your real data.
