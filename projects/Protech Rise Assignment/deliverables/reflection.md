# Project Reflection: AI-Powered Invoice Processing for Aurelia Properties

## Approach
I planned the scenario first (modules, router paths, filters, sheet columns), wrote it as a Make blueprint JSON, imported it, and fixed what broke in the UI. I then tested every case in `email_test_cases.md` with `Run once` and compared the sheet to `expected_results.csv`.

Final flow:
1. **Gmail Watch emails**, searching subjects for invoice, billing, bill, payment due, statement, amount due.
2. **Set variables** holds one shared extraction prompt used by both paths.
3. **Duplicate check**: Sheets Search Rows on Message ID, then an Array aggregator and a filter (continue only if no match).
4. **Router**
   - Path A (PDF attached): list attachment → upload to Drive (link kept) → PDF.co upload → PDF.co PDF-to-text → HTTP fetch of the text → Gemini → Parse JSON → Add a Row.
   - Path B (no attachment, fallback route): HTML-to-text → Gemini → Parse JSON → Add a Row.
5. Status is "Processed", or "Needs Review" when the AI call fails, the source text is empty, `is_invoice` is false, or any of vendor, number, dates, amount is missing. A Notes column says why.

## Key decisions
- **Gemini instead of OpenAI.** I had no OpenAI credit, and the API has no free tier. Gemini has a free key and Make has a native module. The prompt, JSON output and sheet mapping are the same as they would have been for OpenAI.
- **One shared prompt** in a Set variables module, so the PDF and body paths give identical output.
- **Duplicate check before everything else**, keyed on the Gmail Message ID, so duplicates cost almost nothing.
- **Fallback router route** for the body path, so a PDF email never runs both paths.
- **Resume handlers on PDF.co, HTTP, Gemini and Parse JSON** write a "Needs Review" row instead of stopping the run. Add Row has an Ignore handler.
- **Invoice total, not balance due** (GreenLeaf case), and **dates by currency convention** (MM/DD for USD, DD/MM for NGN/GBP), stated in the prompt.
- **PDF.co OCR** was left on, which let the scanned PDF (P6) process correctly.

## Challenges
- **Module IDs.** My first blueprint guessed several Make module IDs and they were wrong (Gmail, Drive, PDF.co). I built the real modules once in a scratch scenario to learn the IDs, then regenerated the blueprint.
- **Gmail Watch emails has no attachments.** I only found this on the first run. It needed an extra module and a different router filter.
- **PDF.co output is a file link.** An HTTP step was needed to read the text.
- **Silent failures.** Make has no `or()` or `empty()` function. The sheet stayed empty because my Ignore handler hid the error. The run history's Module inspector showed the real error message.
- **Duplicate filter.** The aggregator always emits one bundle, so "length = 0" never passed. I tested the first element's value instead.
- **Typed formulas in the Make editor.** Function expressions typed with `{{ }}` are sometimes stored as literal braces. Editing the blueprint JSON and pasting it back was more reliable than editing in the UI.
- **Gemini 503s and Google login expiry** (Drive 401) caused two rows and one blocked run that were not logic errors.
- **Free-plan limits:** 1,000 credits and a 10-email limit per run.

## Lessons learned
- Run a single real email through every module as early as possible. Most of my bugs were wrong field names, not wrong logic.
- Do not put an Ignore handler on the module that writes your results without logging the error somewhere. It hid the root cause for about an hour.
- Read the module's actual output before mapping from it. Do not assume field names.
- A shared prompt, a status column and a notes column make the AI's behaviour much easier to audit.
- Add a retry (or a second-model fallback) for AI calls. A temporary 503 should not produce a manual-review row.
- Check the blueprint's connection IDs after any paste. A stale connection ID looks like a blank connection in the UI.
