# Invoice Parser

Reads every PDF invoice in a folder, extracts **Vendor Name, Invoice
Date, Total Amount and Line Items** using the OpenAI API with strict
Structured Outputs, and appends the results to `results.csv`.

One script. No server, no database, no setup beyond `pip install`.

---

## Run it (3 steps)

**1. Install**

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

On Windows: `py -m venv .venv && .venv\Scripts\activate`

**2. Add your OpenAI key**

```bash
cp .env.example .env
```

Open `.env` and paste your key after `OPENAI_API_KEY=`.

**3. Run**

```bash
python main.py --input invoices --output output/results.csv
```

> **Do a test run first.** `python main.py --limit 1` processes a single
> invoice so you can check the output before paying for all 200.

---

## Options

| Flag | What it does |
| ---- | ------------ |
| `--input PATH` | Folder of PDFs. Default `invoices` |
| `--output PATH` | CSV to append to. Default `output/results.csv` |
| `--limit N` | Only process the first N invoices — use for a cheap test run |
| `--dry-run` | List what would be processed and estimate the cost. No API calls |
| `--recursive` | Also look in subfolders |
| `--line-items-csv PATH` | Also write a second CSV with one row per line item |
| `--verbose` | Show full technical detail when something fails |

---

## What you get

`results.csv`, one row per invoice:

`source_file`, `vendor_name`, `invoice_date`, `invoice_number`,
`total_amount`, `currency`, `line_item_count`, `line_items_json`,
`parse_status`, `notes`

- Dates are always `YYYY-MM-DD`. Amounts are plain numbers with no symbol.
- **A blank cell means the invoice did not state that value.** The script
  is built to leave a field empty rather than guess it — a blank is
  something you can fix, a wrong total is not.
- `parse_status` is `ok` or `needs_review`. Anything flagged
  `needs_review` parsed successfully but failed a sanity check (for
  example, the line items do not sum to the stated total).

---

## If something fails

One bad PDF never stops the run. Failures are printed with the filename
and a reason, and the run continues:

```
[  9/200]  FAIL  scanned-receipt.pdf   no text layer — likely a scanned image
```

**Re-running the same command retries only the failures.** Files already
in `results.csv` are skipped and cost nothing.

Common causes:

- **"no text layer"** — the PDF is a scan or a photo, so there is no text
  to read. Reading these needs OCR, which is not included.
- **"could not open"** — the file is corrupt or password-protected.

---

## What leaves your machine

The **text extracted from each PDF** is sent to the OpenAI API. That
includes vendor names, addresses, amounts and line-item descriptions.

The PDF files themselves are never uploaded. Nothing is stored anywhere
except `results.csv` on this machine. There is no telemetry and the
script makes no network call other than to the OpenAI API.

If your invoices contain personal or regulated data, confirm your
organisation's data-processing agreement with OpenAI before the first
run.

---

## Notes on the key

- Use a dedicated, project-scoped OpenAI key for this job and revoke it
  when the backlog is cleared.
- `.env` is git-ignored. The key is never written to the CSV, never
  printed, and never included in an error message.
