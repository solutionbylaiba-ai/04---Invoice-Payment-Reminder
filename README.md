# 04 — Invoice / Payment Reminder

Every day, this automation checks every unpaid invoice in a Google Sheet, works out how overdue each one is, and automatically emails the client a reminder — with the tone escalating the longer the invoice stays unpaid. No phone calls, no manual tracking of who's been reminded and who hasn't.

## Demo Video

[▶ Watch the demo video](./demo/04-invoice-reminder-demo.mp4)


## Screenshots

**n8n Workflow Canvas**

<img width="1920" height="975" alt="workflow" src="https://github.com/user-attachments/assets/bb9e0951-7351-498f-8017-297a98560ac0" />


**Invoices — Google Sheet**

<img width="1920" height="1009" alt="invoice log" src="https://github.com/user-attachments/assets/0160be3d-bb0c-43b6-99f5-cdb84beed16e" />


**Soft Reminder Email**

<img width="1920" height="984" alt="soft tone notice" src="https://github.com/user-attachments/assets/74e871c5-a74f-4d37-b1c7-1f1223f4706f" />


**Firm Reminder Email**

<img width="1920" height="988" alt="firm tone notice" src="https://github.com/user-attachments/assets/e014dcb4-a2ce-45c0-9636-c0f53909e293" />


**Final Notice Email**

<img width="1920" height="988" alt="final tone notice" src="https://github.com/user-attachments/assets/1eafaa88-6ecc-43f7-930b-31f48bc44cba" />


## What This Automation Does

The workflow runs on a daily schedule. On each run it:

1. Reads every row from the "Invoices" sheet.
2. Skips any invoice already marked `Paid`.
3. Works out how many days overdue each remaining invoice is, based on today's date vs. its `dueDate`.
4. Decides — using the rules below — whether that invoice needs a reminder today, and if so, which of three stages: **soft**, **firm**, or **final**.
5. Emails the client a reminder in that stage's tone, formatted as a proper HTML email (not plain text) with the invoice details laid out clearly.
6. Writes the stage and timestamp back to the sheet, so tomorrow's run knows what's already been sent and doesn't repeat itself or skip a step.

## The Reminder Stages — Exact Criteria

The automation only ever looks at two things per invoice: **how many days overdue it is**, and **what the last reminder sent for it was** (tracked in the `lastReminderSent` column). The rule is checked in this order, and the first one that matches wins:

| Stage | Triggers when | Tone |
|---|---|---|
| **Final** | 14+ days overdue, and the last reminder sent wasn't already `final` | Urgent — asks the client to contact the business directly to settle or arrange a payment plan |
| **Firm** | 7+ days overdue, and the last reminder sent wasn't `firm` or `final` | Direct — states the invoice is overdue by a specific number of days and asks for prompt payment |
| **Soft** | 3+ days overdue, and no reminder has been sent yet | Friendly — a light nudge in case the client simply forgot |

If none of these match — the invoice isn't overdue by at least 3 days yet, or it's already received the reminder appropriate to its current stage — **no email is sent that day.**

### Why this doesn't spam the client

Because each rule also checks what was already sent, an invoice moves through the stages exactly once each, in order, no matter how many times the workflow runs:

- A fresh invoice gets **one soft** reminder around day 3, stays quiet, then gets **one firm** reminder once it crosses day 7, then **one final** notice once it crosses day 14.
- Once an invoice has reached `final`, it keeps getting checked every day, but the rule `lastReminderSent !== 'final'` is now false, so it's simply skipped for good — it will not receive endless final notices every day it remains unpaid. (At that point it's meant to be handled directly, not by automation.)
- Marking the invoice `Paid` at any point removes it from consideration immediately, regardless of how overdue it was.

## The Three Email Templates

Each stage has its own HTML email template (built in the **Determine Reminder Stage** code node) — not a plain-text message. Each one has a colored header matching its urgency (soft = brand teal, firm = amber, final = red), a clean table showing the invoice number, amount, due date, and (for firm/final) days overdue, and a short message appropriate to that stage. The `subject` and `message` fields produced here feed directly into the **Send Reminder** node, so no HTML/formatting settings need to be touched on the Gmail node itself — it treats the message as HTML by default.

There's a `BUSINESS_NAME` constant at the top of the code — update it to the client's business name before deploying, since it's used in the email sign-off.

## What You'll Need

1. **Google Sheet** named "Invoices" (or whatever tab name you use — just make sure the node's Sheet selection matches), with headers exactly: `invoiceId, clientName, clientEmail, amount, currency, dueDate, status, lastReminderSent, lastReminderAt`.
   - `status` is set to `Unpaid` by default; change it to `Paid` manually once payment comes in — reminders stop automatically after that.
   - `dueDate` format: `YYYY-MM-DD` (e.g. `2026-08-15`).
   - `lastReminderSent` and `lastReminderAt` should be left blank for new invoices — the workflow fills these in itself.
   - **Important:** make sure the header row has no leading or trailing spaces in any column name (select row 1 → **Data → Data cleanup → Trim whitespace** in Google Sheets). A stray space in a header (e.g. `"dueDate "` instead of `"dueDate"`) makes that field come through as `undefined` in the workflow and silently breaks the whole automation — it's the single most common setup mistake here.
2. **Gmail account** the reminders will be sent from.

## Import Steps

1. n8n → **Import from File** → `workflow.json`.
2. **Get All Invoices** and **Update Reminder Status** nodes → attach a Google Sheets credential, and set Document/Sheet to your actual spreadsheet and tab.
3. **Send Reminder** node → attach a Gmail credential.
4. **Determine Reminder Stage** (Code node) → this is where the day thresholds (3 / 7 / 14) and the three HTML email templates live. Change the thresholds here if a client wants a different escalation schedule, and update `BUSINESS_NAME` at the top.
5. **Update Reminder Status** node → under "Values to Update", keep only three fields: `invoiceId` (matching column), `lastReminderSent`, `lastReminderAt`. Remove any other columns the node may have auto-added (clientName, amount, etc.) — leaving them in with empty values will overwrite that data in the sheet. Because this node's direct input is the Gmail node's send result (which doesn't carry the invoice fields forward), reference the earlier node explicitly:
   - `invoiceId`: `{{ $('Determine Reminder Stage').item.json.invoiceId }}`
   - `lastReminderSent`: `{{ $('Determine Reminder Stage').item.json.stage }}`
   - `lastReminderAt`: `{{ $now.toISO() }}`

## How to Test

- Add a test row with `dueDate` 3+ days before today, `status` = `Unpaid`, and `lastReminderSent` blank — run the workflow manually and confirm a **soft** email arrives and the sheet updates.
- Change that same row's `dueDate` to 7+ days before today and re-run — it should escalate to **firm**.
- Push it to 14+ days and re-run — it should escalate to **final**.
- Add a row that's already marked `final` with an old `lastReminderAt` and confirm it does **not** receive another email (duplicate-prevention).
- Add a row marked `Paid` that's overdue, and one with a future `dueDate` — confirm both are skipped entirely.
- If a run ever returns zero reminders unexpectedly, the first thing to check is the sheet's header row for stray spaces (see note above) — this is the most common cause.

## For a New Client

Duplicate the sheet, point the three Google Sheets nodes at the new sheet ID, update `BUSINESS_NAME` in the code node, and attach the client's Gmail credential (or send from your own agency inbox on their behalf). No other changes needed — the day thresholds and escalation logic apply as-is.
