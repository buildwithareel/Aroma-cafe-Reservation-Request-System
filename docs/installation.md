# 🚀 Installation Guide

This guide walks you through setting up the **Aroma Coffee & Tea Co. — Reservation Request System** from a blank n8n instance to a fully working, website-connected automation.

> 📖 New to the project? Start with the [main README](../README.md) for an overview before following the steps below.

---

## 1. Prerequisites Checklist

| # | Requirement | Where to get it |
|---|---|---|
| 1 | An **n8n instance** (Cloud or self-hosted) | [n8n.io](https://n8n.io) |
| 2 | A **Google account** with Sheets access | [sheets.google.com](https://sheets.google.com) |
| 3 | An **email-sending credential** (SMTP, Gmail, etc.) supported by n8n's Email Send node | Your email provider |
| 4 | A **website or form** able to `POST` JSON to a webhook URL | Your own site, or the included Aroma demo front end |

📎 For full credential setup, see [configuration.md](./configuration.md).

---

## 2. Get the Workflow File

```bash
git clone https://github.com/<your-username>/aroma-reservation-request-system.git
cd aroma-reservation-request-system
```

The importable workflow file is:

```
aroma-reservation-workflow.json
```

---

## 3. Import the Workflow into n8n

1. Open your n8n instance.
2. Go to **Workflows → Import from File**.
3. Select `aroma-reservation-workflow.json`.
4. The full 11-node canvas loads — **01 - Reservation Webhook → 02 - Validate Request → 03 - Valid Request? → ... → response nodes**.

> ⚠️ The workflow is inactive on import, and the Google Sheets and Email nodes are pre-filled with placeholder values (e.g. `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID`). This is expected — continue to the next step.

---

## 4. Connect Credentials

Open each of the following nodes and attach the matching credential:

| Node | Credential to attach |
|---|---|
| 06 - Save to Google Sheets | Google Sheets OAuth2 |
| 07 - Email Customer | SMTP / Email credential |
| 08 - Notify Restaurant Staff | SMTP / Email credential |

Full field-by-field details: [configuration.md](./configuration.md).

---

## 5. Replace Placeholder Values

Search the workflow for the following placeholders and replace each with your real values:

| Placeholder | Found In | Replace With |
|---|---|---|
| `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` | 06 - Save to Google Sheets | Your Google Sheet's document ID |
| `REPLACE_WITH_SHEET_NAME_OR_GID` | 06 - Save to Google Sheets | Your sheet/tab name or GID |
| `REPLACE_WITH_SENDER_EMAIL@example.com` | 07 - Email Customer, 08 - Notify Restaurant Staff | The email address sending these notifications |
| `REPLACE_WITH_STAFF_EMAIL` | 08 - Notify Restaurant Staff | The restaurant staff's real inbox address |

---

## 6. Prepare Your Google Sheet

Create a Google Sheet with a header row matching the columns the workflow writes to:

```
Request ID | Date Received | Reservation Date | Reservation Time | Guests | Guest Name | Phone | Email | Seating Preference | Special Requests | Source | Status
```

Copy the Sheet's document ID from its URL (`https://docs.google.com/spreadsheets/d/<THIS_PART>/edit`) into the `06 - Save to Google Sheets` node.

---

## 7. Activate the Webhook

1. Open **01 - Reservation Webhook**.
2. Copy the **Production Webhook URL** (once the workflow is Active) or the **Test URL** for local testing.
3. Point your website's reservation form to `POST` its submission to this URL as a JSON body — see [configuration.md](./configuration.md#4-expected-webhook-payload) for the exact expected field names.

---

## 8. Activate the Workflow

1. Toggle the workflow to **Active**.
2. Submit a real test reservation through your website (or send a test `POST` request with a tool like Postman/curl).
3. Confirm:
   - A new row appears in your Google Sheet with status `Pending Confirmation`.
   - The guest receives the confirmation email.
   - The restaurant staff receives the notification email.
   - The website receives a `200` JSON response with a `requestId`.

---

## 9. Verify End-to-End

- [ ] A valid submission produces a Google Sheets row, both emails, and a `200` success response.
- [ ] An invalid submission (e.g. missing guest name) returns a `400` response with a clear error message, and **does not** write to Sheets or send any email.
- [ ] Temporarily breaking a credential (e.g. Google Sheets) confirms the workflow still returns a clean `500` response instead of hanging.
- [ ] The Request ID shown in the website's success response matches the one written to Google Sheets and included in both emails.

If anything fails, check credential mapping in [configuration.md](./configuration.md) and review the run in n8n's **Executions** tab.

---

## Next Steps

- 🔑 [Configuration Guide](./configuration.md)
- 🗺️ [Workflow Architecture](./workflow-architecture.md)
- 🧠 [Code Node Reference](./codenode.md)
- 📘 [Workflow Guide](./workflow-guide.md)