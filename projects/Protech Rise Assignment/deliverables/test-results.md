# Test Results: Aurelia Invoice Processing (Make.com)

Scenario: Gmail → duplicate check → router (PDF path / body path) → PDF.co / HTML-to-text → Gemini → JSON parse → Google Sheet.
Tested on 19 Nov 2026 with `Run once`. Expected values come from `expected_results.csv`.

| # | Case | Expected | Actual | Result |
|---|------|----------|--------|--------|
| P1 | PDF, table layout, NGN + VAT | BrightSpark Electrical Services, INV-2026-0412, 2026-11-03, due 2026-11-17, 614900 NGN, Processed | Same. Total includes VAT (614900, not the 572000 subtotal). Drive link recorded. | Pass |
| P2 | PDF utility bill (different labels) | Lagos Metro Water Co., LMW-88213, 2026-11-01, due 2026-11-21, 191750 NGN, Processed | Same. Account number and meter readings ignored. | Pass |
| P3 | PDF, long-form dates, USD, discount | Clearview Cleaning Co., CV-1187, 2026-10-28, due 2026-11-27, 553.50 USD, Processed | Same (553.5). | Pass |
| P4 | PDF, DD/MM/YYYY, GBP, deposit | GreenLeaf Landscaping, GL-26-077, 2026-11-10, due 2026-12-10, 1200 GBP, Processed | Same. Invoice total (1200) used, not balance due (900). Dates not misread as October. | Pass |
| P5 | Blank PDF | Needs Review, no crash | Needs Review, note "Source text empty or unreadable". Run continued. | Pass |
| P6 | Scanned image-only PDF | Needs Review, or Processed if OCR works | Processed: SecureGate Security Ltd, SG/0931, 2026-11-05, due 2026-11-20, 913750 NGN (PDF.co OCR extracted the text). | Pass |
| B1 | Clean body invoice | ProFlow Plumbing Ltd, PF-5520, 2026-11-06, due 2026-11-20, 370000 NGN, Processed | Same. Attachment URL blank. | Pass |
| B2 | Informal body, US dates, USD | HostNex, HX-00981, 2026-11-01, due 2026-11-30, 49 USD, Processed | First run misread the invoice date as 2026-01-11. After I added a USD = MONTH/DAY rule to the prompt: 2026-11-01, due 2026-11-30. | Pass (after fix) |
| B3 | Missing invoice number and due date | Needs Review | Chidi Okafor Painting, 150000 NGN, Needs Review, note "Missing required invoice fields". | Pass |
| B4 | Matches keywords, not an invoice | Needs Review (is_invoice = false) | Blank fields, Needs Review, note "Not an invoice". | Pass |
| B5 | Messy forwarded text, relative due date | Processed or Needs Review | Sunrise Generators, SG-2026/77, 2026-11-02, due 2026-11-09 (7 days computed from the invoice date), 275000 NGN, Processed. | Pass |
| B6 | Duplicate (same email re-run) | No new row | Re-ran the same emails twice. The run log shows "The bundle did not pass through the filter" for every email already in the sheet. No extra rows. | Pass |
| B7 | Subject has no keyword | No row | Never fetched (Gmail search excludes it). No row written. | Pass |

## Failures found and fixed during testing
1. **Gmail "Watch emails" does not return attachments.** Added a "List email attachments and media" module on the PDF path and routed on `hasAttachment`.
2. **PDF.co returns text as a file link.** Added an HTTP request module to fetch the text before the AI step.
3. **Status/Notes formulas silently failed.** Make has no `or()` or `empty()` functions, and the Ignore handler hid the error. Rewrote with nested `if` and `ifempty`.
4. **Gemini 503 "high demand".** Two rows (B3 and B5) came back "AI call failed" on first attempt. The Resume handler correctly wrote a Needs Review row instead of stopping. Re-running those emails succeeded. Switched model to Gemini 3.5 Flash Lite.
5. **Date ambiguity (B2)** fixed with an explicit prompt rule.
6. **Google Drive 401.** Drive login had to be re-authorized (manual sign-in).

## Known limitations
- The Gmail trigger returns up to 10 emails per run, so a burst of more than 10 emails takes more than one 15-minute cycle.
- A temporary Gemini outage produces a "Needs Review" row ("AI call failed") rather than a retry. The row can be re-run by deleting it.
- Free-plan credit budget: polling every 15 minutes uses roughly 100 credits a day.
