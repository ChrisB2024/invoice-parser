# Invoice Parser — PDF Backlog Extractor

## The Problem

> The client's internal finance/ops staffer currently opens each of 200
> supplier PDF invoices one at a time and re-types vendor, date, total and
> line items into a spreadsheet by hand, because the invoices arrive as
> unstructured PDFs with no machine-readable export and no two suppliers
> use the same layout — and this eliminates it by extracting each PDF's
> text locally, forcing an LLM to return a fixed validated schema, and
> appending the result straight into `results.csv`.

At roughly 3–5 minutes of manual re-typing per invoice, the backlog is
10–16 hours of work that currently blocks whatever reconciliation sits
downstream of it.

## Who Hurts

- **Primary user:** an internal team member at the client (finance / ops /
  bookkeeping) who has been handed the 200-invoice backlog and is
  technical enough to run a Python script from a terminal, but is not a
  developer.
- **What they do today:** open PDF → read → alt-tab → type vendor, date,
  total → type each line item → next PDF. Errors are typos and
  transpositions, and nobody catches them until reconciliation fails.
- **What it costs them:** ~10–16 hours of a salaried person's time, plus
  the error rate of manual transcription, plus the delay to every
  downstream process waiting on the data.

## What This Is

A single standalone Python script. It reads every `.pdf` in a local
folder, extracts the text with `pdfplumber`, sends that text to the
OpenAI API under a strict Pydantic schema (Structured Outputs), and
appends one validated row per invoice to `results.csv`. It runs on the
team member's own machine. It is a utility to clear a backlog, not a
service, not a pipeline, and not an app.

## Causality Chain

Extraction is accurate and every failure is visible → the team trusts the
CSV enough to stop spot-checking every row → the 200-invoice backlog
closes in an afternoon instead of two weeks → the contract closes clean
with a working, self-contained tool.

## Core Flow

1. Team member receives the folder: `main.py`, `requirements.txt`, `README.md`.
2. `python -m venv .venv && source .venv/bin/activate`
3. `pip install -r requirements.txt`
4. Copies `.env.example` to `.env`, pastes the company OpenAI key into it.
5. Drops the 200 PDFs into `invoices/`.
6. `python main.py --input invoices --output output/results.csv`
7. Console prints one line per invoice as it is processed, and a summary
   at the end: N parsed, N skipped (already done), N failed with reasons.
8. Opens `output/results.csv` in Excel/Sheets. One row per invoice, with
   vendor, date, total, currency, line items, and the source filename.
9. Re-runs the same command if anything failed — already-parsed files are
   skipped, so only the failures cost another API call.

## Capabilities

### Extraction

- Read every `.pdf` in a directory (non-recursive by default, `--recursive` to descend)
- Extract the embedded text layer per page
- Detect and skip PDFs with no text layer (scanned images), logging them by name

### Parsing

- Extract four fields per invoice: Vendor Name, Invoice Date, Total Amount, Line Items
- Also capture currency and invoice number when present (free — they're in the same call)
- Return `null` rather than a guess when a field is genuinely absent from the document

### Output

- Append to `results.csv`, one row per invoice, header written once
- Line items serialized as JSON in a single column, plus a `line_item_count`
- Optional `--line-items-csv` writes the long format: one row per line item, keyed by source file
- A `parse_status` column on every row so partial results are visible, not hidden

### Operation

- Resumable: a file already present in `results.csv` is skipped on re-run
- `--limit N` for a cheap test run before committing to all 200
- `--dry-run` to list what would be processed and estimate cost, with no API calls
- One unreadable PDF never ends the run

## Boundaries

### Building now

- Everything under Capabilities above
- A single `main.py`, a `requirements.txt`, a `README.md`, a `.env.example`
- Console logging and an end-of-run summary

### Not yet

- OCR for scanned/image-only PDFs (Tesseract or the OpenAI vision path).
  Real need, deliberately deferred — it roughly doubles the scope and the
  per-invoice cost. The script must *detect and report* these files so
  the decision is informed, and the extractor is kept behind a single
  function so OCR can be dropped in without touching anything else.
- Concurrency / batching for throughput
- Confidence scores or human-review flagging
- Currency normalisation or FX conversion

### Never

- A web UI, an API server, a database, or a queue
- A hosted or multi-tenant version
- Storing the OpenAI key anywhere but the environment
- Uploading the PDFs themselves anywhere — only extracted text is sent
- Any dependency beyond the four in `requirements.txt`

## Done Looks Like

1. `pip install -r requirements.txt` succeeds on a clean Python 3.10+ venv on macOS and Windows.
2. `python main.py --input invoices` processes a folder of 200 mixed-layout PDFs to completion without an unhandled exception.
3. `output/results.csv` opens in Excel with one row per successfully parsed invoice and the four required fields populated.
4. A deliberately corrupt PDF and a scanned image-only PDF are both logged by filename with a reason, are skipped, and do not stop the run.
5. Re-running the exact same command makes zero API calls and writes zero new rows.
6. The OpenAI key appears in no source file, no log line, and no CSV cell.
7. A spot-check of 10 invoices against their PDFs shows vendor, date and total matching exactly.
