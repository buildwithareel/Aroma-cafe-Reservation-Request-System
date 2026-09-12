# 🧠 Code Node Reference

This workflow relies on three `n8n-nodes-base.code` nodes to do its heavy lifting — validation, ID generation, and data formatting. This document explains exactly what each script does, line by line where it matters, so you can safely modify the logic later.

> 📖 For where these nodes sit in the overall pipeline, see [workflow-architecture.md](./workflow-architecture.md).

---

## 1. `02 - Validate Request`

**Purpose:** Independently re-validate every field from the incoming webhook payload, regardless of what the website's own form validation already did.

```javascript
const input = $input.first().json.body || $input.first().json;

const errors = [];

const guestName = (input.guestName || '').toString().trim();
const email = (input.email || '').toString().trim();
const phone = (input.phone || '').toString().trim();
const date = (input.date || '').toString().trim();
const time = (input.time || '').toString().trim();
const guestsRaw = input.guests;
const guests = Number(guestsRaw);
let seatingPreference = (input.seatingPreference || '').toString().trim();
let specialRequests = (input.specialRequests || '').toString().trim();
let source = (input.source || '').toString().trim();

const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

if (!guestName) errors.push('Guest name is required.');
if (!email) errors.push('Email is required.');
else if (!emailRegex.test(email)) errors.push('Email format is invalid.');
if (!phone) errors.push('Phone number is required.');
if (!date) errors.push('Reservation date is required.');
if (!time) errors.push('Reservation time is required.');

if (guestsRaw === undefined || guestsRaw === null || guestsRaw === '' || isNaN(guests) || guests <= 0) {
  errors.push('Guests must be a positive number.');
}

const allowedSeating = ['Indoor', 'Patio', 'No Preference'];
if (!allowedSeating.includes(seatingPreference)) {
  errors.push('Seating preference must be Indoor, Patio, or No Preference.');
}

if (!specialRequests) specialRequests = 'None';
if (!source) source = 'Aroma Reservation Demo';

const isValid = errors.length === 0;

return [{
  json: {
    isValid,
    errorMessage: errors.join(' '),
    guestName,
    email,
    phone,
    date,
    time,
    guests,
    seatingPreference,
    specialRequests,
    source
  }
}];
```

### Walkthrough

| Step | What it does |
|---|---|
| `const input = ...` | Reads from `body` if the webhook wrapped the payload there, otherwise falls back to the raw JSON — makes the node resilient to slightly different webhook payload shapes. |
| Field extraction | Every field is coerced to a string and `.trim()`-ed, so stray whitespace never causes a false validation failure or a messy spreadsheet row. |
| `emailRegex` | A standard, pragmatic email-format check (not RFC-perfect, but catches the vast majority of real mistakes). |
| `errors.push(...)` | Each failed check appends a specific, human-readable message — these get joined together and shown directly to the guest in the `400` response. |
| `guests` validation | Explicitly checks for `undefined`/`null`/empty string **before** the `isNaN`/`<= 0` check, since `Number('')` evaluates to `0`, which would otherwise slip past a naive check. |
| `allowedSeating` | An allow-list, not just a non-empty check — protects downstream email templates and the spreadsheet from unexpected values. |
| Defaults | `specialRequests` and `source` are optional and get sensible defaults rather than validation errors. |
| Return shape | Returns a single item with `isValid` and `errorMessage` alongside every normalized field — this is what `03 - Valid Request?` branches on, and what flows forward unchanged if valid. |

### How to extend
- **Add a new required field:** add an extraction line, a validation check with `errors.push(...)`, and include it in the returned object.
- **Change the seating options:** edit the `allowedSeating` array — no other node needs to change.
- **Tighten email validation:** swap `emailRegex` for a stricter pattern, or add an MX-record lookup in a separate node before this one.

---

## 2. `04 - Generate Request ID`

**Purpose:** Create a unique, human-readable identifier for the reservation, and set its initial status.

```javascript
const item = $input.first().json;

function randomHex(len) {
  const chars = '0123456789ABCDEF';
  let out = '';
  for (let i = 0; i < len; i++) {
    out += chars[Math.floor(Math.random() * chars.length)];
  }
  return out;
}

const datePart = item.date
  ? item.date.replace(/-/g, '')
  : new Date().toISOString().slice(0, 10).replace(/-/g, '');

const requestId = `AR-${datePart}-${randomHex(4)}`;

return [{
  json: {
    ...item,
    requestId,
    status: 'Pending Confirmation'
  }
}];
```

### Walkthrough

| Step | What it does |
|---|---|
| `randomHex(len)` | Generates a short random hex string (e.g. `3F2A`) used as a uniqueness suffix. Not cryptographically secure — that's intentional, since this is a human-facing reference ID, not a security token. |
| `datePart` | Strips dashes from the reservation's `date` field (`2026-09-10` → `20260910`). Falls back to today's date if `date` is somehow missing at this point (shouldn't happen post-validation, but kept as a safety net). |
| `requestId` | Final format: `AR-<YYYYMMDD>-<4-char hex>` — e.g. `AR-20260910-3F2A`. The `AR` prefix stands for "Aroma Reservation." |
| `status` | Every new request starts as `"Pending Confirmation"` — this value flows into the spreadsheet and both emails, making the pending state explicit everywhere it's shown. |
| `...item` | Spreads all previously validated/normalized fields forward, so nothing from step `02` is lost. |

### Collision risk
With 4 hex characters, there are 65,536 possible suffixes per day. For a single restaurant's daily reservation volume, collision risk is negligible — but if you fork this for high-volume use, consider increasing `randomHex(4)` to `randomHex(6)` or incorporating a timestamp component.

### How to extend
- **Change the ID prefix:** edit the `AR-` literal in the template string.
- **Use a different ID scheme (e.g. sequential):** replace this node's logic with a call to an external counter (e.g. a dedicated Google Sheets cell, or a database sequence) instead of random hex.

---

## 3. `05 - Prepare Reservation Data`

**Purpose:** Convert machine-friendly date/time values into human-friendly display strings for the emails and spreadsheet.

```javascript
const item = $input.first().json;

const resDate = new Date(item.date + 'T00:00:00');
const dateDisplay = resDate.toLocaleDateString('en-US', {
  year: 'numeric', month: 'long', day: 'numeric'
});

const [hh, mm] = item.time.split(':').map(Number);
const timeHelper = new Date();
timeHelper.setHours(hh, mm, 0, 0);
const timeDisplay = timeHelper.toLocaleTimeString('en-US', {
  hour: 'numeric', minute: '2-digit', hour12: true
});

const dateReceived = new Date().toISOString().slice(0, 10);

return [{
  json: {
    ...item,
    dateDisplay,
    timeDisplay,
    dateReceived
  }
}];
```

### Walkthrough

| Step | What it does |
|---|---|
| `resDate` | Parses the `date` field (`YYYY-MM-DD`) into a `Date` object, anchored at local midnight (`T00:00:00`) to avoid timezone rollover shifting the displayed date by a day. |
| `dateDisplay` | Formats the date as a friendly long-form string, e.g. `September 10, 2026`. |
| `[hh, mm]` | Splits the 24-hour `time` string (`"12:00"`) into hour/minute numbers. |
| `timeHelper` | Uses today's date purely as a scratch object to apply `setHours()`, so `toLocaleTimeString` can format just the time portion. |
| `timeDisplay` | Formats the time as a friendly 12-hour string, e.g. `12:00 PM`. |
| `dateReceived` | Captures **today's** date (when the request was submitted) — distinct from the reservation date itself, and used as an audit trail in the spreadsheet. |

### How to extend
- **Change the date/time format:** adjust the options object passed to `toLocaleDateString` / `toLocaleTimeString`, or swap `'en-US'` for another locale.
- **Add a day-of-week field:** add `weekday: 'long'` to the `toLocaleDateString` options object, or compute `resDate.toLocaleDateString('en-US', { weekday: 'long' })` as a separate field.
- **Support 12-hour input time:** if you ever accept `time` in `HH:MM AM/PM` format instead of 24-hour, this parsing logic will need to change accordingly — currently it assumes strict 24-hour input.

---

## Next Steps

- 🚀 [Installation Guide](./installation.md)
- 🔑 [Configuration Guide](./configuration.md)
- 🗺️ [Workflow Architecture](./workflow-architecture.md)
- 📘 [Workflow Guide](./workflow-guide.md)