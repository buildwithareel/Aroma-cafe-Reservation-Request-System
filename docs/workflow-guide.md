# 📘 Workflow Guide

A practical, day-to-day guide to **using, testing, monitoring, and extending** the Aroma Coffee & Tea Co. reservation workflow — written for whoever operates this system after it's live, not just the person who set it up.

> 📖 For setup, see [installation.md](./installation.md). For internals, see [workflow-architecture.md](./workflow-architecture.md) and [codenode.md](./codenode.md).

---

## 1. What This Workflow Actually Does (Plain-English Summary)

A guest fills out the reservation form on the Aroma website and hits **Request Reservation**. Behind the scenes:

1. The request is checked for missing/invalid fields.
2. If invalid → the guest sees an error message immediately, nothing is saved or emailed.
3. If valid → a Request ID is generated, the reservation is saved to a Google Sheet as `Pending Confirmation`, and two emails go out: one to the guest ("we got your request"), one to staff ("please review and confirm").
4. The website shows a success message with the Request ID.

**Important:** this system does **not** confirm the reservation automatically. It only captures and routes the request — a human still has to check availability and follow up with the guest.

---

## 2. Daily Operations Checklist (For Restaurant Staff)

- [ ] Check the staff inbox for new **"☕ New Reservation Request"** emails.
- [ ] Open the Google Sheet and find the matching row by **Request ID**.
- [ ] Confirm real availability for the requested date/time/party size.
- [ ] Contact the guest (phone or email) to confirm, decline, or offer an alternative time.
- [ ] Manually update the **Status** column in the Google Sheet (e.g. `Confirmed`, `Declined`, `Rescheduled`) — this workflow does not update status automatically after the initial write.

> 💡 Since status updates are currently manual, consider using consistent status values (`Pending Confirmation`, `Confirmed`, `Declined`, `Rescheduled`, `No Response`) so the sheet stays easy to filter and report on.

---

## 3. Testing the Workflow

### 3.1 Test a valid submission
Send a `POST` request to your webhook URL with a complete, valid payload (see [configuration.md](./configuration.md#4-expected-webhook-payload) for the exact shape). Confirm:
- A `200` response with a `requestId`.
- A new row in Google Sheets with status `Pending Confirmation`.
- The guest confirmation email arrives.
- The staff notification email arrives.

### 3.2 Test a validation failure
Submit a payload missing a required field (e.g. omit `guestName`), or with an invalid value (e.g. `seatingPreference: "Window"` instead of an allowed value). Confirm:
- A `400` response with a specific error message.
- **No** row is added to Google Sheets.
- **No** emails are sent.

### 3.3 Test a processing failure
Temporarily break one credential (e.g. disconnect the Google Sheets credential) and submit a valid payload. Confirm:
- A `500` response is returned — the request does not hang or silently fail.
- No partial state is left in a confusing spot (e.g. an email sent without a matching Sheets row) — reconnect the credential afterward and re-test to confirm normal behavior resumes.

### 3.4 Example test payload

```json
{
  "guestName": "Test Guest",
  "email": "test@example.com",
  "phone": "0000000000",
  "date": "2026-12-01",
  "time": "18:30",
  "guests": 2,
  "seatingPreference": "No Preference",
  "specialRequests": "This is a test submission",
  "source": "Manual Test"
}
```

> 💡 Use a clearly fake guest name or a `specialRequests` note like `"This is a test submission"` so staff can easily ignore test rows in the spreadsheet.

---

## 4. Monitoring & Troubleshooting

| Symptom | Likely Cause | Where to Check |
|---|---|---|
| Guest says they submitted but nothing arrived | Webhook URL on the website doesn't match the workflow's actual URL, or the workflow is inactive | n8n **Executions** tab — is the run even appearing? |
| Row appears in Sheets but no emails sent | Email credential expired/misconfigured | Open `07`/`08` node execution data in the failed run |
| Emails sent but no Sheets row | Sheets credential/permissions issue, or wrong Document ID/Sheet Name | Open `06` node execution data in the failed run |
| Guest gets a `400` for what looks like a valid submission | A field the website sends doesn't match what `02 - Validate Request` expects (e.g. wrong date format) | Compare the actual payload (visible in the failed execution's input data) against [configuration.md](./configuration.md#4-expected-webhook-payload) |
| Website shows a generic error for everything | Front end isn't distinguishing `400` vs `500` vs `200` responses | Review the front end's fetch/AJAX error handling logic |

**General debugging steps:**
1. Open n8n's **Executions** tab and find the relevant run (filter by date/time of the guest's complaint).
2. Click into the run and inspect each node's input/output data — this shows you exactly which node received bad data or failed.
3. For webhook payload mismatches specifically, compare the raw input on `01 - Reservation Webhook` against the expected shape in [configuration.md](./configuration.md).

---

## 5. Common Customizations

### 5.1 Change the restaurant's branding in emails
Edit the inline HTML/CSS directly inside `07 - Email Customer` and `08 - Notify Restaurant Staff`. Colors, fonts, and copy are all self-contained in each node — no shared template file to hunt down.

### 5.2 Add a new form field (e.g. "Occasion")
1. Add the field to the website form.
2. In `02 - Validate Request`, extract and (optionally) validate the new field.
3. Add it to the returned object in `02`, and it will automatically flow through `04` and `05` unchanged (thanks to the `...item` spread pattern used in each Code node).
4. Add a matching column to the Google Sheet and map it in `06 - Save to Google Sheets`.
5. Add it to the HTML templates in `07`/`08` if guests/staff should see it.

### 5.3 Change what counts as a valid seating preference
Edit the `allowedSeating` array inside `02 - Validate Request` — see [codenode.md](./codenode.md#2-04---generate-request-id) for the exact line.

### 5.4 Add a staff confirmation step
This is the most requested extension (see [README.md's Future Improvements](../README.md#-future-improvements)). At a high level:
1. Add a unique confirm/decline link or button to the staff email, encoding the `requestId`.
2. Build a second small webhook workflow that receives that click, updates the matching Google Sheets row's `Status` column, and triggers a follow-up email to the guest.
3. Keep this as a **separate** workflow rather than bolting it onto this one, to keep each workflow focused and easy to debug.

---

## 6. Operational Notes

- **This workflow does not deduplicate submissions.** A guest who submits the form twice will generate two separate Request IDs and two sets of emails. If this becomes a problem, add a duplicate-check step (see Future Improvements in the main README).
- **Request IDs are not sequential** — they're date-prefixed with a random suffix, so don't rely on them for counting daily volume. Use the `Date Received` column in Google Sheets for that instead.
- **The spreadsheet, not the emails, is the source of truth.** If a guest disputes what was agreed, the Google Sheets row (once staff update its Status) is the authoritative record — the emails are notifications, not a booking ledger.

---

## Next Steps

- 🚀 [Installation Guide](./installation.md)
- 🔑 [Configuration Guide](./configuration.md)
- 🗺️ [Workflow Architecture](./workflow-architecture.md)
- 🧠 [Code Node Reference](./codenode.md)