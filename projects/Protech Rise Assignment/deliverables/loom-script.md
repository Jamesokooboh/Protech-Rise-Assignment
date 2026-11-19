# Loom Walkthrough Script (about 4 minutes)

Have ready before recording: the Make scenario open, the Google Sheet open in another tab, and Gmail open to the inbox.

## 0:00 Intro (20 s)
"Hi, this is my invoice automation for Aurelia Properties. It watches the founder's Gmail, finds invoice emails, uses AI to read them, and logs every invoice in a Google Sheet. It works for both PDF attachments and invoices written in the email body."

## 0:20 The scenario (45 s) – show the Make canvas
- "It starts with Gmail Watch emails, searching the subject for invoice, billing, bill, payment due, statement and amount due. It runs every 15 minutes."
- "A Set variables module holds one extraction prompt, so both paths use the same instructions."
- "Next is the duplicate check. It searches the sheet for the Gmail Message ID, and the filter stops the run if it's already there."
- "Then a router. If the email has a PDF, path one runs. If not, path two reads the email body."

## 1:05 PDF path (45 s)
- "For PDFs: I list the attachment, save it to Google Drive so we keep a link, send it to PDF.co to extract text — it also does OCR for scanned files — fetch that text, and send it to Gemini."
- "Gemini returns JSON only: vendor, invoice number, dates, total, currency, description, is-invoice and confidence. A Parse JSON module turns that into fields."

## 1:50 Body path (25 s)
- "For emails without an attachment, the HTML body is converted to plain text and goes to the same Gemini prompt. Same output, same columns."

## 2:15 Error handling and status (35 s) – show an error handler and the Add a Row formula
- "PDF.co, the HTTP step, Gemini and the JSON parse all have Resume handlers. If one fails, the run keeps going and writes a row marked Needs Review instead of stopping."
- "Status is Processed, or Needs Review if it's not an invoice, fields are missing, the PDF text is empty, or the AI call failed. The Notes column says why."

## 2:50 Live results (60 s) – switch to the Google Sheet
- "Here are the results from my tests." Scroll through rows:
  - "PDF invoices from four different layouts, in Naira, dollars and pounds. Totals include VAT, and for GreenLeaf it takes the invoice total, not the balance due."
  - "A scanned PDF was read with OCR."
  - "Body invoices, including a messy forwarded one where the due date was 'in 7 days' and it worked out the date."
  - "A blank PDF, a missing-fields email and a non-invoice email are all flagged Needs Review, with a reason."
- Optional: send one new email and click Run once to show a new row appear.

## 3:50 Duplicates and wrap-up (20 s)
- "If I run the same emails again, the duplicate filter stops them. No second row."
- "The scenario is switched on and runs every 15 minutes. I used Gemini instead of OpenAI because I didn't have API credit. Thanks for watching."
