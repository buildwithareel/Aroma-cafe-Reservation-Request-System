# 🔑 Configuration Guide

Every credential, field, and payload contract needed to run the **Aroma Coffee & Tea Co. — Reservation Request System**.

> ℹ️ This workflow ships with clearly marked placeholder values (`REPLACE_WITH_...`) instead of real credentials or IDs — it contains no secrets out of the box. Configure everything below through n8n's Credentials Manager and by editing the placeholder node fields directly.

---

## 1. Credential Reference Table

| Credential | Type in n8n | Used By Node(s) | Description |
|---|---|---|---|
| Google Sheets OAuth2 | `Google Sheets OAuth2 API` | 06 - Save to Google Sheets | Grants write access to the reservation request log spreadsheet. |
| Email Credential | `SMTP` / your provider's email credential | 07 - Email Customer, 08 - Notify Restaurant Staff | Authenticates outbound HTML email sends. |

---

## 2. Google Sheets Configuration

**Used by:** `06 - Save to Google Sheets`

1. Create a spreadsheet with a header row matching:

   ```
   Request ID | Date Received | Reservation Date | Reservation Time | Guests | Guest Name | Phone | Email | Seating Preference | Special Requests | Source | Status
   ```

2. In n8n: **Credentials → Add Credential → Google Sheets OAuth2 API**, complete the OAuth consent flow.
3. Open **06 - Save to Google Sheets** and:
   - Select the credential.
   - Replace `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` with your Sheet's document ID.
   - Replace `REPLACE_WITH_SHEET_NAME_OR_GID` with your sheet/tab name.
   - Confirm the **Matching Column** is set to `Request ID` — this is what makes the append operation safely identify each unique row.

---

## 3. Email Configuration

**Used by:** `07 - Email Customer`, `08 - Notify Restaurant Staff`

1. In n8n: **Credentials → Add Credential**, choosing whichever email credential type matches your provider (SMTP, Gmail, SendGrid, etc.).
2. Open **07 - Email Customer**:
   - Select the credential.
   - Replace `REPLACE_WITH_SENDER_EMAIL@example.com` in the **From** field with your real sending address.
   - The **To** field is already dynamically set to `{{$json.email}}` (the guest's submitted email) — no change needed.
3. Open **08 - Notify Restaurant Staff**:
   - Select the same (or a different) credential.
   - Replace `REPLACE_WITH_SENDER_EMAIL@example.com` in the **From** field.
   - Replace `REPLACE_WITH_STAFF_EMAIL` in the **To** field with the restaurant's real staff inbox (this can be a shared inbox or distribution list).

> 💡 Both emails use inline HTML templates already styled to match Aroma's brand colors. You can edit the HTML directly inside each node to adjust copy, branding, or add your own logo.

---

## 4. Expected Webhook Payload

**Used by:** `01 - Reservation Webhook` → validated in `02 - Validate Request`

The webhook expects a `POST` request with a JSON body shaped like this:

```json
{
  "guestName": "Areel",
  "email": "areeldemo12@gmail.com",
  "phone": "03005765754",
  "date": "2026-09-10",
  "time": "12:00",
  "guests": 5,
  "seatingPreference": "Indoor",
  "specialRequests": "Birthday Party",
  "source": "Aroma Website"
}
```

| Field | Required | Notes |
|---|---|---|
| `guestName` | ✅ | Non-empty string. |
| `email` | ✅ | Must match a standard email format. |
| `phone` | ✅ | Non-empty string (no format enforced). |
| `date` | ✅ | Expected as `YYYY-MM-DD`. |
| `time` | ✅ | Expected as 24-hour `HH:MM`. |
| `guests` | ✅ | Must be a positive number. |
| `seatingPreference` | ✅ | Must be exactly one of: `Indoor`, `Patio`, `No Preference`. |
| `specialRequests` | ❌ | Defaults to `"None"` if omitted. |
| `source` | ❌ | Defaults to `"Aroma Reservation Demo"` if omitted. |

> ⚠️ If any required field is missing or invalid, the workflow responds with HTTP `400` and a specific, human-readable error message — it never partially processes an invalid request.

---

## 5. Webhook Endpoint Reference

| Setting | Value |
|---|---|
| Method | `POST` |
| Path | `/aroma-reservation` |
| Response Mode | `responseNode` (response is controlled explicitly by nodes 09/10/11) |

Your production webhook URL will look like:

```
https://<your-n8n-domain>/webhook/aroma-reservation
```

---

## 6. Response Contract (for front-end developers)

| Scenario | HTTP Status | Body |
|---|---|---|
| ✅ Success | `200` | `{ "success": true, "requestId": "AR-...", "status": "pending", "message": "Your reservation request has been received." }` |
| ❌ Validation error | `400` | `{ "success": false, "message": "<specific validation error(s)>" }` |
| ❌ Processing failure | `500` | `{ "success": false, "message": "We couldn't process your reservation request right now. Please try again or contact the restaurant directly." }` |

Front ends should branch their UI on `response.success` and, for `400` errors, can display `response.message` directly to the guest.

---

## 7. Node Field Reference

Quick reference for all placeholder/config fields you must fill in before going live:

| Node | Field | Example Value |
|---|---|---|
| 06 - Save to Google Sheets | Document ID | `1AbCDefGhIjkLmNoPQRstuVWXyz` |
| 06 - Save to Google Sheets | Sheet Name | `Reservations` |
| 07 - Email Customer | From | `hello@aromacoffeetea.com` |
| 08 - Notify Restaurant Staff | From | `hello@aromacoffeetea.com` |
| 08 - Notify Restaurant Staff | To | `staff@aromacoffeetea.com` |

---

## 8. Rotating & Securing Credentials

- Never commit a real Google Sheet ID, staff email, or credentials directly into a shared/public copy of this workflow JSON — keep placeholder values in any version you publish.
- Use n8n's Credentials Manager for all OAuth2/SMTP credentials so secrets stay encrypted and out of the exported JSON.
- If you rotate your Google Sheets or email credentials, simply reselect the updated credential on the relevant node — no other node needs to change.

---

## Next Steps

- 🚀 [Installation Guide](./installation.md)
- 🗺️ [Workflow Architecture](./workflow-architecture.md)
- 🧠 [Code Node Reference](./codenode.md)
- 📘 [Workflow Guide](./workflow-guide.md)