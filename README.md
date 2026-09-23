# Invoice-Extractor-Automation-n8n-
An AI-powered invoice processing system built in n8n. It watches an inbox for
incoming invoices (in any common format), extracts the data with an LLM,
validates the numbers, logs everything to a spreadsheet, tracks payment
status, and alerts on two channels — with zero manual data entry.
What it does
Core pipeline (event-driven, triggered per email):
	1.	Watches Gmail for incoming emails with attachments (PDF, image, Word,
Excel, or CSV)
	2.	Handles multiple attachments per email, splitting each into its own
item so every file is processed individually
	3.	Converts non-PDF files (Word/Excel/CSV) to PDF automatically via the
CloudConvert API, using an upload → poll → download flow with a retry loop
	4.	Extracts invoice data using an LLM (Claude and Gemini are both
supported) — vendor, invoice number, date, line items, total, and category
	5.	Validates the math — sums the line items and flags a mismatch if they
don’t match the stated total
	6.	Checks for duplicates against a Google Sheet invoice log
	7.	Logs clean invoices to Google Sheets with a “Pending” payment status
	8.	Sends alerts on two channels (Telegram + Email) for three outcomes:
duplicate detected, total mismatch, or successfully logged
	9.	Unsupported file types are routed to a fallback alert instead of
failing silently
Companion scheduled workflows (time-driven):
	•	Payment reminders — runs daily, checks the sheet for any invoice still
marked “Pending” after 7+ days, and sends a reminder
	•	Monthly summary — runs on the 1st of each month, reports total invoice
count, total spend, and top spending category
Tech stack
	•	n8n — workflow orchestration (Cloud or self-hosted)
	•	Claude / Gemini API — invoice data extraction (vision + document
analysis)
	•	CloudConvert API — Word/Excel/CSV → PDF conversion
	•	Google Sheets — invoice log / lightweight database
	•	Gmail — trigger, attachment source, and outbound email alerts
	•	Telegram Bot API — real-time alerting

    Architecture
Gmail Trigger
    │
    ▼
Get Message (download attachments)
    │
    ▼
Split Attachments (one item per file)
    │
    ▼
Switch (route by file type)
    ├── PDF / Image ──────────────┐
    └── Word/Excel/CSV            │
          │                       │
          ▼                       │
    CloudConvert                  │
    (Create Job → Upload →        │
     Poll until finished →        │
     Download PDF)                │
          │                       │
          └───────────┬───────────┘
                       ▼
              AI Analyze Document
              (Claude / Gemini)
                       │
                       ▼
              Parse + Validate
              (sum line items vs. stated total)
                       │
                       ▼
           Google Sheets: check for duplicate
                       │
         ┌─────────────┼─────────────┐
    Duplicate?     Mismatch?      All clear
         │              │              │
         ▼              ▼              ▼
   Telegram + Email  Telegram + Email  Log to Sheets
   Duplicate Alert   Mismatch Alert    (Status: Pending)
                                             │
                                             ▼
                                    Telegram + Email
                                    Logged Successfully

     Payment Reminder (seperate workflow)
  Schedule Trigger (daily) → Read Sheet → Filter Pending 7+ days → Telegram Alert

     Monthly Summary
 Schedule Trigger (monthly) → Read Sheet → Calculate Totals → Telegram + Email

    Setup
1.	Import all three workflow JSON files into your n8n instance
	2.	Connect credentials for: Gmail, Google Sheets, Telegram Bot, and your AI
provider of choice (Anthropic and/or Google AI Studio)
	3.	Add a CloudConvert API key for file conversion (free tier: 25
conversions/day)
	4.	Create a Google Sheet with columns: Date | Vendor | Invoice Number | Category | Total | Status | Notes
	5.	Update the placeholder IDs in each workflow (Sheet ID, Telegram Chat ID,
email address, credential references)
	6.	Activate all three workflows

        Status
Fully built and tested end-to-end, including duplicate detection, mismatch
detection, multi-attachment handling, multi-format file conversion, and
dual-channel alerting. Built and debugged from scratch — including OAuth
scopes, binary data handling, S3-style multipart uploads, async polling
loops, and cross-provider AI response parsing (Claude vs. Gemini response
shapes).

       Possible future additions 
•	Vendor-based anomaly detection (flag invoices unusually high vs. a
vendor’s history)
•	Auto-reply to the sender confirming receipt
•	A simple dashboard (e.g. Looker Studio) on top of the sheet
•	Retry/backoff limit on the CloudConvert polling loop


