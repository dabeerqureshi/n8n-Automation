# AI E-Commerce WhatsApp Assistant — Setup Guide

This package gives you a working WhatsApp shopping assistant that runs on your own
self-hosted n8n and your own local Ollama model. No cloud AI costs, no Google Sheets
OAuth required — product/order data lives in simple CSV files on disk.

## What's in this package

```
n8n-ecommerce-whatsapp-workflow.json   ← import this into n8n
data/
  products.csv       ← catalog (10 sample products)
  inventory.csv       ← reference stock table (not written to live, see note below)
  customers.csv        ← sample customer records
  orders.csv            ← orders start here, new orders are appended automatically
  faqs.csv                ← question/answer knowledge base
  coupons.csv               ← discount codes
  shipping.csv                ← shipping methods (reference)
  cart.csv                       ← sample data only; live carts are kept in memory, see below
  tracking.csv                     ← sample shipment tracking records
  settings.csv                       ← store name, tax rate, currency, AI model, etc.
  logs.csv                              ← every reply the bot sends gets logged here
  errors.csv                               ← reserved for your own error logging
README.md (this file)
```

---

## Part 1 — For non-technical users: what this bot actually does

Think of it as a shop assistant that lives inside WhatsApp. A customer can:

- Say **"MENU"** to see what the bot can do
- Ask in plain English — *"do you have wireless headphones?"* — and the bot will search
  the real product catalog and answer, including price and stock
- Ask questions like *"what's your return policy?"* and get an answer straight from
  your FAQ list (it will never make up a policy that isn't in the sheet)
- Add items to a cart: **CART ADD P003 1**
- View the cart: **CART**
- Apply a discount code: **COUPON WELCOME10**
- Place the order: **CHECKOUT** — this writes a real row into `orders.csv` and reduces
  stock in `products.csv`
- Track an order: **TRACK ORD001**
- Ask for a real person: **HUMAN**

Everything the bot says about products, prices, stock, and policies comes from the
CSV files — it is not allowed to invent facts. If you want to change what's for
sale, you edit `products.csv`. If you want to change an answer, you edit `faqs.csv`.
No coding needed for day-to-day store management.

To go live, you (or a technical person) need to do three things once:
1. Install n8n and Ollama on a server (see Part 2)
2. Connect a WhatsApp Business account (Meta gives you a token and phone number ID)
3. Import the workflow file and paste in your credentials

---

## Part 2 — For technical users: full setup

### Requirements

- Self-hosted **n8n** (any recent version, tested against the v1.x node set:
  Webhook v2, IF v2, Switch v3, HTTP Request v4.2, Code, Respond to Webhook,
  Email Send v2.1)
- **Ollama** running locally with a model pulled, e.g.:
  ```
  ollama pull llama3.1:8b
  ```
- A **WhatsApp Business Cloud API** app (Meta for Developers) with:
  - a permanent access token
  - a Phone Number ID
  - a webhook Verify Token you choose yourself
- n8n's Code node needs filesystem access. By default n8n blocks Node's built-in
  `fs`/`path` modules inside Code nodes. Enable them by setting this environment
  variable before starting n8n:
  ```
  NODE_FUNCTION_ALLOW_BUILTIN=fs,path
  ```
  (Docker: add `-e NODE_FUNCTION_ALLOW_BUILTIN=fs,path` to your `docker run`, or the
  equivalent `environment:` entry in docker-compose.)

### Step 1 — Place the data files

Copy the entire `data/` folder to **`/data`** on the machine n8n runs on (this is the
path the workflow's Code nodes read/write from).

- **Native install**: `sudo mkdir -p /data && sudo cp -r data/* /data/` and make sure
  the user running n8n can read *and write* that folder.
- **Docker**: mount a volume, e.g. in docker-compose.yml:
  ```yaml
  services:
    n8n:
      image: n8nio/n8n
      environment:
        - NODE_FUNCTION_ALLOW_BUILTIN=fs,path
      volumes:
        - ./data:/data
  ```

If you'd rather use a different path, open the workflow's **"Parse Message"**,
**"Load Data & Route"**, **"Handle Cart Add"**... etc. Code nodes and change the line
`const DATA_DIR = '/data';` in each (it appears once per Code node that touches CSVs).

### Step 2 — Import the workflow

1. In n8n: **Workflows → Import from File** → select `n8n-ecommerce-whatsapp-workflow.json`
2. Open the node named **"Parse Message"**. This is the single configuration point —
   edit the `CONFIG` object at the top:

   | Field | What to put |
   |---|---|
   | `whatsappToken` | Your WhatsApp Cloud API permanent access token |
   | `phoneNumberId` | Your WhatsApp Business phone number ID |
   | `ollamaUrl` | Usually `http://localhost:11434/api/chat`. If n8n runs in Docker and Ollama runs on the host, use `http://host.docker.internal:11434/api/chat` instead |
   | `ollamaModel` | The model you pulled, e.g. `llama3.1:8b` |
   | `dataDir` | Informational only — actual path is set per-node, see Step 1 |
   | `adminEmail` | Where "new order" notifications should go |
   | `rateLimitMax` / `rateLimitWindowMs` | Anti-spam limits (default: 8 messages/minute) |

3. Open the node **"Check Verify Token"** and set `VERIFY_TOKEN` to a secret string
   of your choosing — you'll enter this same string in the Meta webhook setup.

4. Open **"Email Admin New Order"** and attach your SMTP credentials (Gmail app
   password, SendGrid, or any SMTP server). If you skip this, order emails simply
   fail silently and the customer's WhatsApp confirmation is unaffected — checkout
   still works.

### Step 3 — Activate and connect the webhook

1. Save and **Activate** the workflow in n8n.
2. Copy the **Production URL** of the **"Webhook Receive (POST)"** node (n8n shows
   this in the node panel) — it will look like
   `https://your-n8n-domain/webhook/whatsapp-webhook`
3. In the Meta for Developers dashboard → WhatsApp → Configuration:
   - Callback URL: the URL from step 2
   - Verify Token: the same string you set in "Check Verify Token"
   - Subscribe to the `messages` webhook field
4. Meta will send a GET request to verify — this is handled automatically by the
   **"Webhook Verify (GET)"** branch of the workflow.

### Step 4 — Test it

Send "hi" from a WhatsApp number allowed to message your test app. You should get
the menu back within a few seconds (first Ollama call may be slower — it's loading
the model into memory).

---

## How the workflow is built (architecture notes)

```
Webhook Verify (GET)  ──▶ Check Verify Token ──▶ Respond Verify

Webhook Receive (POST) ──▶ Parse Message ──▶ IF Has Message
                                                 │true
                                                 ▼
                          Session/Dedup/RateLimit Manager
                                                 │
                                  IF Duplicate ──┴── (true → drop silently)
                                                 │false
                              IF Rate Limited ───┴── (true → "slow down" reply)
                                                 │false
                                Load Data & Route (reads all CSVs, detects intent)
                                                 │
                                          Switch Route
             ┌──────┬──────┬─────────┬──────────┬───────────┬──────────┬────────┬────────────────┬─────┬──────────┐
           menu   track  human   cart_add   cart_view   cart_clear  checkout  coupon    product_search  faq   fallback
             │      │      │        │           │            │         │        │            │          │       │
             ▼      ▼      ▼        ▼           ▼            ▼         ▼        ▼            ▼          ▼       │
         (each branch builds `replyText`, checkout also writes orders.csv/products.csv)     Ollama (grounded)  │
             └──────┴──────┴────────┴───────────┴────────────┴─────────┴────────┴────────────┴──────────┴──────┘
                                                 │
                                    Send WhatsApp Message ──▶ Log Interaction
```

### Design decisions and why they make this robust

- **Deterministic commands beat conversational slot-filling.** Cart/checkout/tracking
  use exact commands (`CART ADD P003 1`, `TRACK ORD001`) instead of asking the AI to
  parse free-form multi-turn orders. This is the single biggest source of bugs in
  most WhatsApp-bot tutorials — a model misreading a quantity or product name mid
  checkout. Here, the AI is only used for search and FAQ, where being "grounded but
  approximate" is fine; money-moving actions (cart, checkout) are 100% rule-based.
- **The AI can't hallucinate stock or prices.** Product search and FAQ prompts embed
  the *actual* CSV rows into the system prompt and instruct the model to only use
  that data. If Ollama is down or times out, a deterministic fallback message (or the
  matched FAQ answer verbatim) is sent instead — the customer never gets left with no
  reply.
- **Every external call can fail without breaking the workflow.** The Ollama HTTP
  call, the WhatsApp send call, and the admin email all have `continueRegularOutput`
  + retry configured, so one flaky call doesn't throw the whole execution into n8n's
  error state.
- **Duplicate delivery is handled.** WhatsApp sometimes re-sends the same webhook;
  message IDs are tracked in n8n's workflow static data so duplicates are dropped.
- **Rate limiting** protects against accidental loops or spam (default: 8
  messages/minute per phone number).
- **Stock is validated twice**: once when adding to cart, and again at checkout
  (re-read from disk) in case stock changed between adding and checking out.
- **Cart lives in memory (n8n static data)**, not in `cart.csv` — this avoids
  concurrent-write conflicts on every keystroke. Only the final `orders.csv` and
  `products.csv` are written to disk, and only at checkout.

### Known limitations (by design, for a tutorial/small-store scale)

- CSV files are not a real database — concurrent checkouts from many customers at
  the exact same second can race on the read-modify-write of `products.csv`. For
  production scale, swap the CSV read/write functions (they're isolated at the top
  of each Code node) for calls to Postgres/MySQL/Airtable/Google Sheets API.
- Session/cart state is stored in n8n's workflow static data, which resets if you
  deactivate/reactivate the workflow or restart n8n without persistence configured.
  For durability across restarts, move session storage into a CSV/database table too
  (the code is structured so this is a small, contained change).
- The bot answers one message at a time; it doesn't currently support images, voice
  notes, or location messages beyond acknowledging them (route → fallback).

### Extending it

- **New FAQ**: add a row to `faqs.csv` (Question / Keywords / AI Answer). No workflow
  changes needed.
- **New product**: add a row to `products.csv` with a unique Product ID and SKU.
- **New command**: add a regex branch in the **"Load Data & Route"** Code node, add a
  new rule + output to **"Switch Route"**, add a new **Handle X** Code node, and wire
  it to **"Send WhatsApp Message"**.
