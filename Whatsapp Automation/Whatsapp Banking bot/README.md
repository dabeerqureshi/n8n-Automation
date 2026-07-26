# ABC Bank — WhatsApp Banking Chatbot (self-hosted n8n)

A production-ready n8n workflow that lets customers check balance, view statements,
ask FAQs (Google Sheet + local Ollama AI fallback), and reach support — over WhatsApp.

## Files in this package

| File | Purpose |
|---|---|
| `whatsapp-banking-bot.json` | The n8n workflow — import directly |
| `customers.csv` | Seed data for the **Customers** Google Sheet |
| `faqs.csv` | Seed data for the **FAQs** Google Sheet |
| `sessions.csv` | Empty template for the **Sessions** Google Sheet (state store) |
| `README.md` | This guide |

---

## 1. Architecture

```
Customer (WhatsApp)
     │
     ▼
Meta WhatsApp Cloud API
     │  webhook (GET verify / POST message)
     ▼
n8n Webhook nodes
     │
     ▼
Parse payload → Get Session (Google Sheets) → Determine Route
     │
     ├─ MAIN_MENU        → send menu
     ├─ MENU_SELECTION    → ask PIN / show FAQ prompt / show support
     ├─ VALIDATE_PIN      → Get Customer → check PIN → balance/statement
     ├─ FAQ                → search FAQ sheet → fallback to Ollama (local LLM)
     └─ INVALID           → re-prompt
     │
     ▼
Update Session (Google Sheets, upsert)
     │
     ▼
Send WhatsApp reply (Graph API)
```

The workflow uses **IF-node chains** (not Switch) for routing — this keeps the
workflow portable across n8n versions since IF node schemas are more stable.

---

## 2. Prerequisites

- Self-hosted n8n (v1.4x+ recommended), reachable via a **public HTTPS URL**
  (WhatsApp requires HTTPS for webhooks — use a reverse proxy / ngrok / Cloudflare Tunnel if testing locally)
- A Google account + Google Sheets, with an n8n **Google Sheets OAuth2** credential configured
- A Meta developer account with a **WhatsApp Business Cloud API** app
- **Ollama** running and reachable from n8n (locally or in the same Docker network), with a model pulled, e.g. `ollama pull llama3.1`

---

## 3. Step-by-step setup

### 3.1 Google Sheets

1. Create one Google Sheet with **three tabs**: `Customers`, `FAQs`, `Sessions`.
2. Import the matching CSV into each tab (File → Import → Upload, "Replace current sheet").
   - `customers.csv` → `Customers` tab
   - `faqs.csv` → `FAQs` tab
   - `sessions.csv` → `Sessions` tab (leave empty — n8n writes to it automatically)
3. Copy the Sheet's **Document ID** from its URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

### 3.2 WhatsApp Cloud API (Meta)

1. In Meta for Developers, create an app → add the **WhatsApp** product.
2. Note down: **Phone Number ID**, and generate a **permanent access token**
   (System User token via Meta Business Settings — temporary tokens expire in 24h).
3. Under WhatsApp → Configuration, set the **Webhook URL** to:
   `https://<your-n8n-domain>/webhook/whatsapp-webhook`
4. Set a **Verify Token** of your choice (any string) — you'll reuse it below.
5. Subscribe the webhook to the `messages` field.

### 3.3 Ollama

Make sure Ollama is running and pull a model:
```bash
ollama pull llama3.1
ollama serve   # usually already running as a service
```
If n8n runs in Docker and Ollama runs on the host, use `http://host.docker.internal:11434`
instead of `http://localhost:11434` as `OLLAMA_URL`.

### 3.4 Import the workflow

1. In n8n: **Workflows → Import from File** → select `whatsapp-banking-bot.json`.
2. Open each **Google Sheets** node and:
   - Select your Google Sheets credential
   - Replace the placeholder Document ID (`YOUR_GOOGLE_SHEET_ID`) with your real Sheet ID
   - Reselect the sheet tab (`Customers` / `FAQs` / `Sessions`) from the dropdown — this
     refreshes the column list for your n8n version
3. Set environment variables (Settings → Environment Variables, or your `.env` / docker-compose):

```
WHATSAPP_TOKEN=your_permanent_meta_access_token
WHATSAPP_PHONE_NUMBER_ID=your_phone_number_id
WHATSAPP_VERIFY_TOKEN=whatever_you_set_in_meta
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=llama3.1
```

4. **Activate** the workflow.
5. In Meta, click "Verify and Save" on the webhook — it will hit the `Webhook - Verify`
   (GET) node and should succeed instantly.

### 3.5 Test

Send "Hi" to your WhatsApp Business number → you should get the main menu, then walk
through options 1–4.

---

## 4. Conversation flow

```
User: Hi
Bot:  Welcome to ABC Bank 👋  1) Balance 2) Statement 3) FAQs 4) Support

User: 1
Bot:  Please enter your 4-digit PIN.
User: 1234
Bot:  Hello Ali Khan 👋  Available Balance: $5,420

User: 3
Bot:  Ask me any banking question.
User: How do I block my debit card?
Bot:  (matched from FAQ sheet) Please call our helpline immediately...
```

- **3 wrong PIN attempts** → session resets, customer is told to contact support.
- **FAQ**: exact/keyword match against the `FAQs` sheet is tried first (fast, deterministic,
  no hallucination risk); only unmatched questions are sent to Ollama, with a system prompt
  that forbids inventing account details and caps replies at ~80 words.
- **Session state** (`MAIN_MENU`, `WAITING_PIN_BALANCE`, `WAITING_PIN_STATEMENT`,
  `WAITING_FAQ`) is persisted per phone number in the `Sessions` sheet so the bot knows
  how to interpret the next message.

---

## 5. Important production notes

- **PINs are stored in plaintext in this reference build**, matching your original spec.
  For real production use, do **not** store plaintext PINs in a Google Sheet — replace
  the Customers sheet with a proper database and hash/salt PINs, or better, drop PIN
  auth in favor of OTP verification.
- Google Sheets is fine for a prototype/small customer base; for real transaction volume,
  replace the Google Sheets nodes with Postgres/MySQL nodes for the Sessions table at least
  (Sheets has request-per-100-seconds rate limits).
- The `Ask Ollama` node has no timeout guard — for slow models, add a `Wait`/timeout or
  switch to a smaller/faster model to stay within WhatsApp's response expectations.
- Rotate the WhatsApp permanent token per Meta's System User policy and store it as an
  n8n credential/environment variable, never hard-coded in the workflow.

---

## 6. Known compatibility caveat

Google Sheets and IF node parameter schemas have changed across n8n versions. After
importing, open each Google Sheets node once and re-select the sheet/credential from
the dropdown (this regenerates the column mapping) — if a version mismatch causes a
node to show a red warning icon, this fixes it.
