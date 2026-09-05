# UI Context — Console Output Contract

This project has no graphical UI. It has an operator staring at a
terminal for the ten-or-so minutes a 200-invoice run takes, deciding
whether to trust the output.

So this file is not skipped and it is not a stub: **the console *is* the
interface**, and every section below is the console equivalent of what
the template asks for. Colour tokens become output prefixes; layout
patterns become line formats; state coverage becomes what the operator
sees when there is nothing to show. The agent should never have to invent
an output format — if a format is not derivable from this file, that is a
gap in this file.

## Design Intent

Legible to a non-developer who has never run a Python script before, and
scannable at a glance during a long run. The operator's real question at
every moment is "is this working, and what is it going to cost me?" —
every line answers one of those. No spinners, no progress bars, no ANSI
colour: the output must survive being copy-pasted into an email to Chris
when something goes wrong.

## Output Tokens

Every line begins with one of these six prefixes. Nothing else prints.
Bare `print()` calls with no prefix are a defect.

| Role | Prefix | Used for |
| ---- | ------ | -------- |
| Progress | `[  N/200]` | One line per file, zero-padded so the column never jitters |
| Success | `  OK  ` | A file parsed and written |
| Skipped | ` SKIP ` | Already in the CSV, or past `--limit` |
| Warning | ` WARN ` | Parsed, but something needs a human eye (totals mismatch, empty vendor) |
| Failure | ` FAIL ` | This file produced no row, and why |
| Summary | `====` rule | The end-of-run block |

No emoji, no colour codes. Windows terminals mangle both, and the client
runs both platforms.

## Line Formats

**Per file** — one line, under 100 characters, filename never truncated
because it is the operator's only handle on the file:

```
[  7/200]   OK   acme-invoice-0042.pdf          Acme Ltd  ·  2024-03-11  ·  1,240.00 EUR  ·  6 items
[  8/200]  SKIP  bluecorp-9912.pdf              already in results.csv
[  9/200]  FAIL  scanned-receipt.pdf            no text layer — likely a scanned image, needs OCR
[ 10/200]  WARN  vendor-x-8871.pdf              parsed, but line items sum to 990.00 vs total 1,090.00
```

**End of run** — always printed, even on interrupt:

```
================================================================
  Run complete in 4m 12s
  Parsed      187   → output/results.csv
  Skipped      11   (already processed)
  Failed        2

  Failed files:
    scanned-receipt.pdf      no text layer — likely a scanned image
    corrupt-0031.pdf         could not open: EOF marker not found

  Re-run the same command to retry only the failed files.
================================================================
```

That last line matters more than it looks: it is what stops the operator
deleting `results.csv` and paying for all 200 invoices a second time.

## State Coverage

| State | What renders |
| ----- | ------------ |
| Empty | `No .pdf files found in invoices/ — check the --input path.` Never an empty run that exits 0 silently. Names the flag that fixes it. |
| Loading | Nothing per-file under ~2s. Files that take longer print the progress line *before* the API call, so the operator can see which file is hanging. |
| Error | The filename, then a plain-English reason, then the run continues. Never a raw traceback — tracebacks go to the log only under `--verbose`. |
| Partial | A row that parsed but failed a sanity check is written with `parse_status=needs_review` and printed as `WARN`. Partial data is more useful than no data, but only if it is labelled. |

## Feedback and Latency

| Duration | Response |
| -------- | -------- |
| Under 2s | The progress line only |
| 2s–30s (the normal API call) | Progress line printed before the call, result appended after — so a stalled file is visibly the current one |
| Over 30s | Request times out, is retried with backoff up to 3 attempts, and each retry prints ` WARN  <file>  retry 2/3 after timeout` |
| Failed | ` FAIL ` line with the reason, run continues, file stays eligible for the next run |
| Succeeded | ` OK ` line with the extracted vendor, date and total echoed back — the operator spot-checks against the PDF without opening the CSV |

Echoing the parsed values on success is the whole trust mechanism of this
tool. A silent `OK` would give the operator no reason to believe the
number in the CSV.

## Destructive Actions

- The script has no destructive operation. It never deletes, never
  modifies a source PDF, and never rewrites an existing CSV row.
- If `--output` points at an existing file, it is **appended to**, never
  truncated. There is no `--overwrite` flag — deleting the file is the
  operator's explicit, obvious way to start over.
- `--dry-run` is the confirmation step for the only expensive action:
  it prints the file count and estimated cost and exits without calling
  the API.

## Components

No component library. The only shared primitives are the six output
prefixes above and a single `log(level, filename, message)` helper that
enforces the column alignment. Every message goes through it.

## Layout Patterns

- **Column widths** — fixed: 9 chars for the counter, 6 for the level,
  32 for the filename, remainder for the message. Long filenames push the
  message right rather than being truncated.
- **`--help`** — argparse default, with every flag carrying a one-line
  description written for a non-developer.
- **Breakpoints** — none. The output is designed for an 80-column
  terminal and must not depend on width detection.
