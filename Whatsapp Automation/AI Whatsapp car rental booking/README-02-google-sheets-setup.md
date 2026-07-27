# Google Sheets Setup

## 1. Create the spreadsheet

1. Go to https://sheets.new to create a new Google Sheet.
2. Name it e.g. **"City Rentals Bookings"**.
3. Create a tab named exactly **`Bookings`** (case-sensitive — the workflows reference it by this name).
4. Paste the header row + sample rows from `Bookings-sheet-dummy-data.csv` (open the CSV, copy all, paste into cell A1 of the `Bookings` tab — Google Sheets will auto-split into columns; if not, use **File → Import → Upload** and choose "Insert new sheet" or "Replace current sheet").

   Required header row (must match exactly, this is what the workflow nodes map to):
   ```
   Booking ID | Customer Name | Phone | Email | Car | Pickup Date | Return Date | Pickup Location | Drop-off Location | Driver Required | Total Price | Booking Status | Admin Approval | Admin Notes | Created At | Last Updated
   ```
5. (Optional) Add a second tab named **`Cars`** and import `Cars-Pricing-sheet-dummy-data.csv` into it if you want a visible pricing reference (the workflow currently calculates price from a table inside the Code node — you can later swap this for a live lookup against this sheet if you want prices editable without redeploying the workflow).

## 2. Data validation (recommended, not required)

For the `Booking Status` column, add a dropdown via **Data → Data validation**:
`Pending, Confirmed, Cancelled, Completed, Rejected`

For `Admin Approval`:
`No, Yes`

This prevents typos that would break the IF conditions in Workflow 2 (which match on exact strings `"Yes"`, `"No"`, `"Confirmed"`, `"Rejected"`).

## 3. Get the Sheet ID

Look at the URL:
```
https://docs.google.com/spreadsheets/d/1AbCdEfGhIjKlmNoPQRstuVWXyz/edit
                                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ this is the Sheet ID
```
Copy this ID — you'll paste it into both workflow JSON files (replacing `REPLACE_WITH_GOOGLE_SHEET_ID`) after import, in every Google Sheets node.

## 4. Create the Google Sheets OAuth2 credential in n8n

1. In n8n: **Credentials → New → Google Sheets OAuth2 API**.
2. You need a Google Cloud OAuth Client ID/Secret:
   - Go to https://console.cloud.google.com/apis/credentials
   - Create a project (or use existing).
   - Enable the **Google Sheets API** (APIs & Services → Library).
   - Create an **OAuth 2.0 Client ID** (Application type: Web application).
   - Add n8n's redirect URI (n8n shows this in the credential screen, typically `https://your-n8n-domain/rest/oauth2-credential/callback`, or for local: `http://localhost:5678/rest/oauth2-credential/callback`).
3. Paste the Client ID/Secret into the n8n credential, click **Connect my account**, and authorize with the Google account that owns the sheet.
4. Save the credential as **"Google Sheets account"** — matches the placeholder name used in the workflow JSON.

## 5. Point the workflow nodes at your sheet

After importing the workflows into n8n, open every **Google Sheets** node
(Append Booking Row, Update Row - Confirmed, Watch Bookings Sheet) and:
- Set **Document** to your actual spreadsheet (search by name, or paste the ID).
- Confirm **Sheet** is set to `Bookings`.
- Confirm the credential dropdown shows "Google Sheets account".
