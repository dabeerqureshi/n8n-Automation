# WhatsApp Business Cloud API Setup (Meta)

This workflow uses Meta's official **WhatsApp Business Cloud API** (not a
third-party wrapper). It's free to set up for development/testing.

## 1. Create a Meta App

1. Go to https://developers.facebook.com/apps and create a new app.
2. App type: **Business**.
3. Add the **WhatsApp** product to the app.
4. Meta gives you a **test phone number** and a **temporary access token**
   automatically (valid 24 hours — fine for setup, you'll need a permanent
   token before going live, step 4 below).

## 2. Note your credentials

From **WhatsApp → API Setup** in the Meta App dashboard, copy:
- **Phone number ID** → this replaces `REPLACE_WITH_PHONE_NUMBER_ID` in both workflow JSON files (in every HTTP Request node that calls `graph.facebook.com/.../messages`).
- **Temporary access token** (for testing) or your permanent token (step 4).
- **WhatsApp Business Account ID** (not directly used by the workflow, but useful for reference).

## 3. Add a test recipient (development phase)

While using a test number, Meta requires you to add recipient phone numbers
to an allow-list: **WhatsApp → API Setup → "To" field → Manage phone number
list**, add the number(s) you'll test with, they'll receive a verification
code via WhatsApp.

## 4. Get a permanent access token (for production)

1. Create a **System User** in Meta Business Settings (Business Settings → Users → System Users).
2. Assign it the WhatsApp app with `whatsapp_business_messaging` and `whatsapp_business_management` permissions.
3. Generate a token for that system user with **no expiry** (or long-lived).
4. This is the token you'll use in n8n (step 6 below) — treat it like a password.

## 5. Configure the webhook (connects Meta → your n8n instance)

Your n8n instance must be reachable from the internet on HTTPS (Meta
requires HTTPS with a valid certificate — a self-signed cert will not work).

Options for local n8n:
- Use a tunnel: `ngrok http 5678` (or Cloudflare Tunnel) to expose local n8n publicly, then use the generated `https://xxxx.ngrok.app` URL.
- Or deploy n8n on a small VPS with a real domain + Let's Encrypt cert for production use.

Steps:
1. In n8n, **activate** Workflow 1 (`1 - WhatsApp AI Booking Intake`) and open the **WhatsApp Webhook** node to copy its **Production URL**. It will look like:
   `https://your-domain-or-tunnel/webhook/whatsapp-webhook`
2. In the Meta App dashboard: **WhatsApp → Configuration → Webhook → Edit**.
3. **Callback URL**: paste the URL from step 1.
4. **Verify Token**: choose any string, e.g. `mySecretVerifyToken123`, and enter it here.
5. In the workflow JSON, open the **IF Token Matches** node and set the same value in place of `REPLACE_WITH_YOUR_VERIFY_TOKEN`. Save.
6. Click **Verify and Save** in Meta. Meta will send a GET request with `hub.challenge`; your n8n workflow answers it via the "Respond Challenge OK" node. If it fails, check the tunnel is up and the workflow is active.
7. Under **Webhook fields**, subscribe to **`messages`**.

## 6. Create the WhatsApp credential in n8n

The workflow calls the Graph API directly via HTTP Request nodes using a
**Header Auth** credential (simplest approach):

1. In n8n: **Credentials → New → Header Auth**.
2. Name: `Authorization`
3. Value: `Bearer YOUR_ACCESS_TOKEN` (the permanent token from step 4, or the temporary one for testing).
4. Save as **"WhatsApp Cloud API Token"** — matches the placeholder name used in the workflow JSON.
5. Open every HTTP Request node that sends WhatsApp messages ("Send WhatsApp Reply", "WhatsApp - Confirm Customer", "WhatsApp - Reject Customer") and confirm this credential is selected, and replace `REPLACE_WITH_PHONE_NUMBER_ID` in the URL with your real phone number ID.

## 7. Test end-to-end

Send a WhatsApp message to your test business number, e.g. "Hi, I want to
book a car". You should see an execution appear in n8n, and get a reply
from the AI agent asking for the missing details.

## Production notes

- Outside a 24-hour customer-initiated session window, WhatsApp requires
  pre-approved **message templates** for business-initiated messages. The
  confirmation/rejection messages in Workflow 2 are sent as free-form text,
  which only works if the customer messaged within the last 24 hours, or if
  you convert those messages into approved templates (**WhatsApp →
  Message Templates** in the Meta dashboard) for reliability at scale.
- Rate limits and messaging tiers apply based on your Business verification
  status — see https://developers.facebook.com/docs/whatsapp/cloud-api for
  current limits.
