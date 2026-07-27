# City Rentals — WhatsApp AI Booking System

This package contains a working n8n automation for a WhatsApp-based car rental
booking flow, using a **local Ollama model** for the AI and **Google Sheets**
as the database.

## Files

| File | Purpose |
|---|---|
| `1-WhatsApp-AI-Booking-Intake.json` | n8n workflow: receives WhatsApp messages, AI collects booking details, saves to Sheets, emails admin |
| `2-Admin-Approval-Handler.json` | n8n workflow: watches the sheet, confirms or rejects bookings, notifies customer |
| `Bookings-sheet-dummy-data.csv` | Sample rows for the main `Bookings` sheet |
| `Cars-Pricing-sheet-dummy-data.csv` | Sample rows for an optional `Cars` pricing reference sheet |
| `README-01-ollama-setup.md` | Install/run Ollama locally and pull a model |
| `README-02-google-sheets-setup.md` | Create the Google Sheet and OAuth credential |
| `README-03-whatsapp-business-api-setup.md` | Register with Meta, get tokens, configure webhook |
| `README-04-n8n-import-and-config.md` | Import workflows, wire up credentials, go live |

## Read the READMEs in this order

1. `README-01-ollama-setup.md`
2. `README-02-google-sheets-setup.md`
3. `README-03-whatsapp-business-api-setup.md`
4. `README-04-n8n-import-and-config.md`

## Important — before you go live

These workflows use **placeholder values** you must replace after import:

- `REPLACE_WITH_GOOGLE_SHEET_ID` (in both workflow JSON files, Google Sheets nodes)
- `REPLACE_WITH_PHONE_NUMBER_ID` (in HTTP Request nodes calling the WhatsApp Cloud API)
- `REPLACE_WITH_YOUR_VERIFY_TOKEN` (in the "IF Token Matches" node, Workflow 1)
- `bookings@cityrentals.example` / `admin@cityrentals.example` (email nodes — set your real sender/admin addresses)
- Credential placeholders: `Ollama (local)`, `WhatsApp Cloud API Token`, `Google Sheets account`, `SMTP Account` — these are just labels; you create the actual credentials in n8n (steps in README-04).

## Honesty note on "100% working"

The workflow logic, node wiring, and expressions in these JSON files are complete and internally consistent for a current n8n version (2024–2025 node schema, LangChain Agent nodes). What I can't guarantee sight-unseen is:
- Your exact n8n version may label a couple of node parameters slightly differently (e.g. Google Sheets node UI changes between minor versions) — if n8n flags a node as "unrecognized version" on import, open it once and n8n will offer to update it.
- Meta's WhatsApp Cloud API requires a verified Business/App and (for anything beyond a temporary test token) approved message templates for the very first message you send a customer outside a 24-hour session window — see README-03.
- Local Ollama needs a model actually pulled and enough RAM for it — see README-01.

Everything else (data flow, Sheets columns, approval logic, rejection logic) is production-ready as designed.
