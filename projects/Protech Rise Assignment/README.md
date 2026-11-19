# Aurelia Properties: AI-Powered Invoice Processing (Make.com)

A Make.com scenario that watches a Gmail inbox for invoice emails, reads each invoice with AI, and logs the details in a Google Sheet. It handles both PDF attachments and invoices written in the email body.

## How it works

1. **Gmail "Watch emails"** searches subjects for: invoice, billing, bill, payment due, statement, amount due. Runs every 15 minutes.
2. **Duplicate check:** the Gmail Message ID is looked up in the sheet; the run stops for emails already logged.
3. **Router**
   - **PDF attached:** list attachment → upload to Google Drive (link kept) → PDF.co upload and PDF-to-text (OCR on) → fetch text via HTTP → Gemini.
   - **No attachment:** convert the HTML body to text → Gemini.
4. **Gemini** uses one shared prompt on both paths and returns JSON only: `vendor_name`, `invoice_number`, `invoice_date`, `due_date`, `total_amount`, `currency`, `description`, `is_invoice`, `confidence`.
5. **Parse JSON → Google Sheets "Add a Row".**

**Status** is `Processed`, or `Needs Review` if the email isn't an invoice, required fields are missing, the PDF text is empty, or an AI/parse step failed. The `Notes` column says why. Resume error handlers on PDF.co, HTTP, Gemini and Parse JSON write a Needs Review row instead of stopping the run.

## Sheet columns

Message ID, Vendor Name, Invoice Number, Invoice Date, Due Date, Total Amount, Currency, Description, Sender Email, Email Subject, Attachment URL, Processing Date, Status, Notes

## Files

| File | What it is |
|------|------------|
| `deliverables/aurelia-invoice-blueprint.json` | Exported Make blueprint |
| `deliverables/test-results.md` | Expected vs actual for all 13 test cases, plus bugs found and fixed |
| `deliverables/reflection.md` | Approach, key decisions, challenges, lessons learned |
| `deliverables/loom-script.md` | Script for the 3-5 minute walkthrough video |

## Using the blueprint

1. In Make, create a new scenario and choose **Import Blueprint** from the three-dot menu.
2. Re-select your own connections: Gmail, Google Sheets, Google Drive, PDF.co and Google Gemini AI. The connection IDs in the file belong to the original account and won't resolve in yours.
3. Point the Sheets modules and the Drive folder at your own spreadsheet and folder.
4. Use **Choose where to start** on the Gmail trigger, then **Run once** to test.

## Notes

- **Gemini instead of OpenAI.** The brief names OpenAI. I used Gemini (free API key) because there was no OpenAI credit. The prompt and output format are unchanged.
- **Credits.** Polling every 15 minutes uses roughly 100 Make credits a day. The free plan has 1,000.
- **Per-run limit.** The Gmail trigger returns up to 10 emails per run.
- **Gemini 503s.** A temporary "high demand" error produces a Needs Review row ("AI call failed") rather than a retry.
- **Keys.** No API keys are stored in this repo.
