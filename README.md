# ☕ Aroma Coffee & Tea Co. — Reservation Request System
 
**A full-stack reservation request pipeline: a custom-built restaurant website that captures table requests and hands them off to an n8n backend for validation, logging, and two-sided email notification.**
 
This is a portfolio demo built end-to-end: a real front-end reservation form, a webhook-driven n8n automation backend, structured request tracking in Google Sheets, and branded HTML email confirmations for both the guest and the restaurant staff.
 
<p align="left">
  <img alt="n8n" src="https://img.shields.io/badge/n8n-Workflow-EA4B71?logo=n8n&logoColor=white">
  <img alt="Webhook" src="https://img.shields.io/badge/Webhook-POST%20Trigger-000000?logo=webhook&logoColor=white">
  <img alt="JavaScript" src="https://img.shields.io/badge/JavaScript-Code%20Nodes-F7DF1E?logo=javascript&logoColor=black">
  <img alt="Google Sheets" src="https://img.shields.io/badge/Google%20Sheets-Request%20Log-34A853?logo=googlesheets&logoColor=white">
  <img alt="Email" src="https://img.shields.io/badge/HTML%20Email-Dual%20Notification-EA4335?logo=gmail&logoColor=white">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-lightgrey">
</p>
---

## 📌 Overview
 
**Aroma** is a fictional café built as a portfolio demo, with a real reservation form on its website. When a guest submits the form, it sends a `POST` request to an n8n webhook, which:
 
1. Validates every field server-side (not just in the browser).
2. Generates a unique, human-readable **Request ID**.
3. Formats the reservation into clean, display-ready values.
4. Appends the full request to a **Google Sheet** acting as the restaurant's request log.
5. Emails the **guest** a branded confirmation that their request was received (not yet confirmed).
6. Emails the **restaurant staff** the full reservation details so they can follow up and confirm availability.
7. Returns a clear JSON response back to the website so the front end can show a success or error state.

Unlike a simple "form → spreadsheet" integration, this workflow treats every reservation as a **request that must be manually confirmed** by the restaurant — mirroring how most real independent restaurants actually operate.
 
---

## ✨ Features
 
- 🧾 **Server-side validation** — every field (name, email format, phone, date, time, guest count, seating preference) is validated inside the workflow itself, not just trusted from the browser form.
- 🆔 **Human-readable Request IDs** — auto-generated IDs like `AR-20260910-3F2A` make it easy for staff to reference a specific request in conversation or email.
- 📋 **Structured request log** — every reservation, valid or not, is captured with a consistent schema (name, contact info, date/time, party size, seating preference, special requests, source, and status) in Google Sheets.
- 📧 **Dual HTML email notifications** — a branded confirmation email to the guest, and a separate staff-facing summary email, each with its own tailored template.
- 🚦 **Explicit "Pending Confirmation" status** — every new request is clearly marked as unconfirmed, both in the spreadsheet and in the guest-facing email, avoiding false expectations.
- 🔁 **Three distinct response paths** — success, validation error, and processing failure are each handled with their own dedicated response node and correct HTTP status code.
- 🌐 **Decoupled front end** — the workflow only cares about receiving a well-formed JSON payload, so any website, form builder, or app can act as the front end.
---
 
