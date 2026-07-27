# n8n Import & Configuration

Assumes you already have n8n running locally (`npx n8n`, Docker, or desktop
install) and completed the Ollama, Google Sheets, and WhatsApp steps first.

## 1. Import the workflows

1. Open n8n → **Workflows → Import from File**.
2. Import `1-WhatsApp-AI-Booking-Intake.json`.
3. Import `2-Admin-Approval-Handler.json`.

## 2. Attach credentials

Each workflow references credentials by name only (n8n strips secrets from
exported JSON). After import, click into each node below and pick/create
the matching credential:

**Workflow 1:**
| Node | Credential type | Name to use |
|---|---|---|
| Ollama Chat Model | Ollama API | Ollama (local) |
| Send WhatsApp Reply | Header Auth | WhatsApp Cloud API Token |
| Append Booking Row | Google Sheets OAuth2 API | Google Sheets account |
| Notify Admin - New Booking | SMTP | SMTP Account |

**Workflow 2:**
| Node | Credential type | Name to use |
|---|---|---|
| Watch Bookings Sheet | Google Sheets OAuth2 API | Google Sheets account |
| Update Row - Confirmed | Google Sheets OAuth2 API | Google Sheets account |
| WhatsApp - Confirm Customer / WhatsApp - Reject Customer | Header Auth | WhatsApp Cloud API Token |
| Email Customer/Admin nodes (4x) | SMTP | SMTP Account |

## 3. Replace placeholder values

Search each workflow for these strings and replace them:

- `REPLACE_WITH_GOOGLE_SHEET_ID` → your actual Google Sheet ID (README-02)
- `REPLACE_WITH_PHONE_NUMBER_ID` → your WhatsApp phone number ID (README-03)
- `REPLACE_WITH_YOUR_VERIFY_TOKEN` → the verify token you set in Meta (README-03)
- `bookings@cityrentals.example` / `admin@cityrentals.example` → your real sender/admin email addresses

## 4. SMTP credential (for email nodes)

**Credentials → New → SMTP**. Use your email provider's SMTP details (Gmail
app password, SendGrid, Mailgun, Amazon SES, your own mail server, etc.).
Save as **"SMTP Account"**.

## 5. Activate both workflows

Toggle **Active** (top right) on both workflows. Workflow 1 needs to be
active for the webhook URL to be live; Workflow 2's polling trigger needs
to be active to check the sheet every minute.

## 6. Test the full flow

1. Message your WhatsApp test number → confirm the AI Agent replies and
   asks follow-up questions.
2. Finish providing all details → confirm a row appears in the `Bookings`
   sheet with `Booking Status = Pending`, `Admin Approval = No`, and the
   admin receives an email.
3. In the sheet, manually change `Admin Approval` to `Yes` for that row →
   within ~1 minute, confirm:
   - `Booking Status` auto-updates to `Confirmed`
   - Customer receives a WhatsApp confirmation
   - Customer and admin both receive confirmation emails
4. Test rejection: on a different `Pending` row, set `Booking Status` to
   `Rejected` (keep `Admin Approval` = `No`) → confirm the customer gets a
   polite WhatsApp + email notice, and the admin gets a "customer informed"
   email.

## Troubleshooting

- **Webhook verification fails in Meta**: check the n8n workflow is Active,
  the tunnel/domain is reachable over HTTPS, and the verify token matches
  exactly on both sides.
- **AI Agent returns malformed JSON / breaks IF nodes**: see the reliability
  notes in `README-01-ollama-setup.md`.
- **Google Sheets Trigger doesn't fire**: the `rowUpdate` polling trigger
  only detects changes since its last poll — give it a minute, and make
  sure you're editing the `Bookings` tab specifically (not a different tab).
- **Node shows a version-mismatch warning after import**: open the node,
  n8n will prompt to update it to your installed node version — accept it,
  the parameters map over automatically.
