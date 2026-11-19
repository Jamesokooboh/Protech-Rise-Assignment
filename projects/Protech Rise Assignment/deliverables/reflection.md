# Project Reflection Sheet

**Project:** AI-Powered Invoice Processing Automation for Aurelia Properties
**Tools:** Make.com, Gmail, Google Sheets, Google Drive, PDF.co, Google Gemini
**Date:** 19 November 2026

---

## 1. Summary

I built a Make.com scenario that watches a Gmail inbox for invoice-related emails, reads each invoice with AI, and records the details in a Google Sheet. It works for invoices sent as PDF attachments and for invoices typed into the email body. Every invoice ends up in the same 14-column format, with a Status column that tells the finance team whether the row can be trusted or needs a human look.

I tested 13 cases (6 PDF, 7 email-body or filter cases). All 13 gave the expected result. Full details are in `test-results.md`.

---

## 2. Implementation approach

I worked in this order:

1. **Planned first.** I listed the modules, the router paths, the filters and the sheet columns, and checked the plan before building anything.
2. **Built from a blueprint.** I wrote the scenario as a Make blueprint JSON file and imported it, then fixed errors in the Make editor. This was faster than dragging modules in one by one, although some module IDs in my first draft were wrong and I had to correct them (see Challenges).
3. **Tested every case.** I used `Run once` against real test emails and compared each sheet row to `expected_results.csv`. I fixed each failure before moving on.
4. **Documented.** I exported the blueprint and wrote up the results.

**The final scenario:**

| Step | Module | What it does |
|------|--------|--------------|
| 1 | Gmail: Watch emails | Finds emails whose subject contains invoice, billing, bill, payment due, statement or amount due |
| 2 | Tools: Set variables | Holds one shared extraction prompt so both paths behave identically |
| 3 | Google Sheets: Search Rows | Looks up the Gmail Message ID |
| 4 | Array aggregator + filter | Stops the run if that ID is already in the sheet |
| 5 | Router | Sends the email down the PDF path or the body path |
| A | Gmail attachment list → Google Drive → PDF.co upload → PDF.co PDF-to-text (OCR on) → HTTP fetch of the text → Gemini → Parse JSON → Sheets | PDF path; the Drive link is saved as the Attachment URL |
| B | HTML to text → Gemini → Parse JSON → Sheets | Body path, used when there is no attachment |

The AI returns JSON only, with: `vendor_name`, `invoice_number`, `invoice_date`, `due_date`, `total_amount`, `currency`, `description`, `is_invoice` and `confidence`.

**Status logic.** A row is `Processed` unless the AI says it is not an invoice, the source text is empty, an AI or parsing step failed, or the vendor, invoice number, either date or the total is missing. In those cases it is `Needs Review`, and the Notes column says why. Resume error handlers on PDF.co, the HTTP step, Gemini and Parse JSON write a Needs Review row instead of stopping the run.

---

## 3. Key decisions and why

- **Gemini instead of OpenAI.** The brief names OpenAI, but I had no API credit and the OpenAI API has no free tier. Gemini has a free key and a native Make module, so I used it. The prompt, the JSON format and the sheet mapping are the same as they would have been for OpenAI.
- **One shared prompt.** Both paths read the same prompt from a single module. This is what makes a PDF invoice and a typed invoice come out in the same structure.
- **Duplicate check first.** The Gmail Message ID is checked before any paid steps run, so a repeated email costs almost nothing.
- **Fallback route for the body path.** Without it, an email with a PDF would have run down both paths.
- **Flag, don't discard.** Failures write a Needs Review row with a reason, rather than being dropped or stopping the scenario. This matches the brief's rule that bad records should be flagged for manual review.
- **Invoice total, not balance due.** For the GreenLeaf invoice (total 1200, deposit-adjusted balance 900) I told the AI to use the invoice total, so the sheet shows the full amount owed on the invoice.
- **Dates by currency.** The prompt reads all-numeric dates as month/day for USD and day/month for NGN and GBP. A relative due date such as "due in 7 days" is calculated from the invoice date.
- **PDF.co OCR left on.** This let the scanned, image-only PDF be read instead of being sent to manual review.

---

## 4. Challenges and how I solved them

1. **Wrong module IDs in my first blueprint.** I had guessed several Make module IDs (Gmail, Drive, PDF.co) from memory and they did not exist. I built the real modules once in a scratch scenario, read the correct IDs out of its exported blueprint, and regenerated mine.
2. **The Gmail trigger does not return attachments.** I only found this on the first real run. The Watch emails module returns just a `hasAttachment` flag. I added a "List email attachments" module on the PDF path and routed on `hasAttachment`.
3. **PDF.co returns the text as a file link, not inline text.** I added an HTTP request step to fetch the text before sending it to Gemini.
4. **The sheet stayed empty and nothing looked wrong.** Make has no `or()` or `empty()` function, so my Status and Notes formulas failed on every row. I had put an Ignore error handler on the Add a Row module, which hid the error. I only found it by opening the run's Module inspector in the history view. I rewrote the formulas with nested `if` and `ifempty`.
5. **The duplicate filter never passed.** The aggregator always emits one bundle, even when no row matches, so "length = 0" was never true. I changed the filter to test the first item's value instead.
6. **Typed formulas in the Make editor were unreliable.** Function expressions I typed with `{{ }}` were sometimes stored as literal text. Copying the blueprint out, fixing the JSON, and pasting it back was more reliable than editing in the UI.
7. **Outside problems, not logic errors.** Gemini sometimes returned "503 high demand", which produced two "AI call failed" rows until I re-ran them. My Google Drive login also expired and needed to be re-authorized. A date was misread in test B2 (US-style 11/01/2026 read as 2026-01-11) until I made the date rule explicit in the prompt.
8. **Free-plan limits.** 1,000 credits and a 10-email limit per run meant I batched my tests and turned the schedule off after testing.

---

## 5. Lessons learned

- **Run one real item through the whole flow early.** Most of my bugs were wrong field names and wrong assumptions about what a module returns, not wrong logic. Reading the actual output of each module before mapping from it would have saved a lot of time.
- **Never hide errors on the module that writes your results.** The Ignore handler on Add a Row cost me the most time, because the scenario reported success while writing nothing. Error handlers should record why they fired.
- **Make's formula language is smaller than it looks.** There is no `or()`, `empty()` or reliable `&` string join. Nested `if` and `ifempty` worked, and a plain fixed message was safer than building text with `&`.
- **A status column plus a notes column is worth the effort.** It made every result auditable, and it is what makes the automation safe to trust.
- **AI services fail temporarily.** A short outage should not mean a manual-review row. If I extended this project, I would add a retry, or a second model as a fallback.
- **Check connections after pasting a blueprint.** A stale connection ID shows as a blank connection in the editor and fails on save.

---

## 6. What I would improve with more time

- Add an automatic retry (or fallback model) for AI calls.
- Raise the 10-emails-per-run limit, or loop until the inbox is clear.
- Mark processed emails with a Gmail label as well as the sheet check.
- Notify someone (email or chat) when a row is marked Needs Review.
- Move from the free plan before relying on 15-minute polling in production; it uses about 100 credits a day.
