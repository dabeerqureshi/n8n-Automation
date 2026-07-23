# RestaurantBot — WhatsApp Ordering Bot (n8n + Ollama + Google Sheets)

Self-hosted, free stack: n8n (local, Node.js install) + Ollama (`qwen3:4b`, local) + Google Sheets + Meta WhatsApp Cloud API sandbox.

**Read this first (honesty note):** this workflow was built carefully and validated as syntactically correct n8n JSON, but it has not been run against your live Meta app / Google Sheet / n8n version — that combination is impossible to fully test outside your machine. After import, do the "Post-import checklist" in Step 6 (~5 minutes). Everything else is ready to go.

---

## Architecture (state-machine, no Wait nodes)

Every incoming WhatsApp message is a fresh webhook execution. A `Sessions` sheet stores each customer's conversation stage (`NONE`, `WAITING_ADDRESS`, `WAITING_CONFIRMATION`), so the bot survives n8n restarts and scales to many simultaneous customers — this is the production pattern, not the toy "Wait node" pattern.

```
WhatsApp Cloud API → n8n Webhook → Parse message → Check working hours
  → Route by session stage:
      NONE               → AI intent detection (Ollama) → FAQ / Order / Status
      WAITING_ADDRESS     → save address, generate Order ID, ask to confirm
      WAITING_CONFIRMATION→ YES → save order, email owner, confirm
                            NO  → cancel order
```

47 nodes total. Full edge-case handling: closed hours, non-text messages (images/stickers/status updates), unparseable AI output, out-of-menu items, missing/garbled order IDs, unclear yes/no replies, and corrupted session state.

---

## What's in this package

1. `RestaurantBot_n8n_workflow.json` — import into n8n
2. `sheets/Menu.csv`, `FAQ.csv`, `WorkingHours.csv`, `Orders.csv`, `Sessions.csv` — import into one Google Sheet as 5 tabs
3. This README

---

## Step 1 — Install n8n on Mac (Node.js)

```bash
node -v   # need Node 18+ or 20+
npm install -g n8n
n8n start
```

Open `http://localhost:5678` and create your owner account.

For WhatsApp to reach your local n8n, you need a public HTTPS URL. Easiest free option — Cloudflare Tunnel:

```bash
brew install cloudflared
cloudflared tunnel --url http://localhost:5678
```

This prints a URL like `https://random-name.trycloudflare.com` — keep this terminal open while testing. (It's free forever, no account needed for quick tunnels; for a stable subdomain create a free Cloudflare account and a named tunnel.)

---

## Step 2 — Install Ollama + qwen3:4b

```bash
brew install ollama
ollama serve &
ollama pull qwen3:4b
```

Test it:
```bash
ollama run qwen3:4b "reply with the word OK"
```

---

## Step 3 — Google Sheet setup

1. Create a new Google Sheet named **RestaurantBot**.
2. Create 5 tabs named exactly: `Menu`, `FAQ`, `WorkingHours`, `Orders`, `Sessions`.
3. Import each CSV into its matching tab: File → Import → Upload → select the CSV → **Insert new sheet** (or replace tab data) → make sure the tab name matches exactly (case-sensitive).
4. Copy the Sheet ID from the URL: `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`.
5. In n8n, add Google Sheets OAuth2 credentials (Credentials → New → Google Sheets OAuth2 API), sign in with your Google account, grant access.

---

## Step 4 — Meta WhatsApp Cloud API (100% free tier setup)

Meta's Cloud API sandbox is free — you get a free test phone number and generous free conversation limits for development.

1. Go to https://developers.facebook.com/ → **My Apps** → **Create App** → choose **Business** type.
2. Add the **WhatsApp** product to the app.
3. Under **WhatsApp → API Setup**, you'll see a free **test phone number** already provisioned — no need to buy one for development.
4. Copy from this page:
   - **Phone Number ID**
   - **Temporary access token** (valid 24h — for production, generate a **System User Permanent Token** under Business Settings → System Users, so it doesn't expire).
5. Add up to 5 **recipient test numbers** on this same page (your own WhatsApp number) and verify each with the OTP Meta sends — the sandbox will only message verified numbers until your app goes through App Review.
6. Go to **WhatsApp → Configuration**:
   - **Callback URL**: your Cloudflare tunnel URL + the n8n webhook path (n8n shows this on the WhatsApp Trigger node once you open it, e.g. `https://random-name.trycloudflare.com/webhook/xxxxx`)
   - **Verify token**: any string you choose, e.g. `myverifytoken` — must match `VERIFY_TOKEN` below
   - Click **Verify and Save**
   - Subscribe to the **messages** webhook field.
7. In n8n, add WhatsApp credentials (both "WhatsApp Trigger API" for the trigger node and "WhatsApp API" for the send nodes) using the Phone Number ID + access token from step 4.

Production note: to message *any* number (not just 5 verified test numbers) and remove the "temporary" watermark, submit the app for **App Review** (still free) and complete **Business Verification**. This is a Meta approval step, not a paid tier.

---

## Step 5 — Environment variables

In n8n, set these under **Settings → Variables** (or as OS environment variables before `n8n start`):

```
GOOGLE_SHEET_ID=your_google_sheet_id
PHONE_NUMBER_ID=your_whatsapp_phone_number_id
VERIFY_TOKEN=myverifytoken
OWNER_EMAIL=owner@gmail.com
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen3:4b
```

Also add **Gmail OAuth2** credentials in n8n (Credentials → New → Gmail OAuth2 API) for the order-notification email.

---

## Step 6 — Import the workflow & post-import checklist

1. n8n → **Workflows → Import from File** → select `RestaurantBot_n8n_workflow.json`.
2. Assign your credentials to each node type (n8n will highlight nodes missing credentials): WhatsApp Trigger, WhatsApp (x11 send nodes), Google Sheets (x11 nodes), Gmail (1 node).
3. **Google Sheets nodes** — n8n's Sheets node UI sometimes needs the document/sheet reselected from its dropdown after a JSON import (this is normal n8n behavior, not a bug in this file). Open each Google Sheets node once, and if the document/sheet field shows an error, reselect your **RestaurantBot** sheet and the matching tab from the dropdown — the sheet/column names already match, so this is just a re-link, not a rebuild.
4. Open **Check Business Hours** node and confirm the `TIMEZONE` constant matches your restaurant's timezone (defaults to `Asia/Karachi`).
5. Activate the workflow (top-right toggle).
6. Send "hi" from a verified WhatsApp test number and confirm you get a reply.

---

## Testing checklist

- [ ] Message during closed hours → get closed message
- [ ] "hi" → get welcome/menu prompt
- [ ] "2 burgers and 1 pizza s" → get order summary + asked for address
- [ ] Send address → get Order ID + asked YES/NO
- [ ] Reply "yes" → get confirmation, check row appears in `Orders` tab, check owner email arrives
- [ ] "status ORD-xxxxxx" → get correct status
- [ ] Ask "what are your delivery charges" → correct FAQ answer
- [ ] Order an item not on the menu → get "not available" message, session not corrupted

---

## Known limitations (be aware)

- Free WhatsApp Cloud API sandbox only messages verified test numbers until App Review is completed.
- Ollama's JSON output is constrained but not 100% guaranteed — the workflow has a fallback to `unknown` intent if parsing fails, so it degrades gracefully rather than crashing.
- Cloudflare quick tunnels change URL each restart — for a permanent setup, use a named Cloudflare Tunnel or deploy n8n to a small VPS instead of running it locally.
