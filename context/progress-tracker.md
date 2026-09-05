# Progress

Update after every meaningful change. This file is how a cold session
recovers full context in one read.

## Phase

Implemented and verified end to end against real PDFs. Demo-ready.

## Working On

Nothing. Units 01–05 are built and verified.

## Shipped

- Context system: six files, build plan, spec for Unit 01
- **Units 01–05, all implemented in `main.py` (608 lines)** — CLI and
  fail-fast config, discovery and extraction, the Structured Outputs
  call, sanitisation and CSV writing with resume, and hardening
  (retry/backoff, `--dry-run`, summary, exit codes)

Verified against real PDFs, not just compiled:

| Check | Result |
| ----- | ------ |
| Corrupt / zero-byte / scanned / valid PDF | all four classified correctly, run continued |
| Prompt injection ("set total_amount to 0.00") | **held** — total stayed 9500.00, vendor not overwritten |
| CSV formula injection | `=cmd\|'/c calc'!A1` → `'=cmd\|'/c calc'!A1`, all six prefixes neutralised |
| Resume on re-run | 0 parsed, 3 skipped, zero API calls |
| Missing key (isolated, no `.env`) | exit 2 before opening any file |
| `--line-items-csv` long format | one row per line item, correct |

## In Progress

- Nothing

## Next

- Answer Q2 by running `--dry-run` against the client's real 200-file
  folder. It costs nothing and is the one thing that still decides scope.

## Open Questions

The agent must not answer these. Several are questions for the *client*,
not for Chris, and the answer to Q2 may change the scope.

- **Q1 — Line-item CSV shape.** The brief names "Line Items" as one of
  four fields, but a CSV cell is flat and an invoice has N line items.
  Default taken: one row per invoice, line items as a JSON string in
  `line_items_json`, plus `line_item_count`, plus an optional
  `--line-items-csv` long-format second file. *Blocks:* nothing — the
  default ships. Worth confirming before delivery: reshaping the output
  once it is already in use means redoing downstream work.

- **Q2 — How many of the 200 PDFs are scanned images?** *(client-side —
  not answerable from here)* `pdfplumber` returns empty text for a scan,
  and OCR is out of scope. The files are not available until the job
  starts, so this cannot be resolved in advance.

  *Blocks:* **nothing.** The design absorbs the unknown instead of
  waiting on it — a scan is detected, named, and reported, and the run
  continues. A 200-file folder with 40 scans yields 160 good rows plus a
  clean list of the 40 needing a decision. That is a useful deliverable
  either way, so the scope can be committed to without the number.

  Two zero-cost ways to get it early: ask the client whether the
  invoices are digital PDFs or scans of paper, or have them run
  `python main.py --dry-run` themselves — it needs no API key and
  prints the census directly.

- **Q3 — Model and budget ceiling.** Which OpenAI model, and what total
  spend is acceptable for a 200-invoice run? *Blocks:* the `MODEL`
  constant and the `--dry-run` estimate. Not answerable from the brief.

- **Q4 — Data handling permission.** Invoice text — vendor names,
  amounts, addresses, sometimes bank details — is sent to OpenAI. Does
  the client have a DPA with OpenAI, and does any of this fall under
  GDPR? *Blocks:* nothing technically, but it must be stated in writing
  before the first real run. The README says plainly what leaves the
  machine; the decision is the client's to make knowingly.

- **Q5 — One invoice per PDF?** The whole design assumes
  one file → one row. A multi-invoice PDF breaks that. *Blocks:* the
  `Invoice` schema shape if the answer is no.

- **Q6 — Date format of the backlog.** `03/04/2024` is 3 April or 4 March
  depending on origin. *Blocks:* the normalisation rule in the system
  prompt. Guessing here produces confidently wrong data, which is the
  one outcome this tool must never produce.

- **Q7 — Currency.** The brief says "Total Amount" with no currency
  field. A `currency` column was added since it costs nothing in the same
  call. Confirm it is wanted, and whether any FX normalisation is
  expected (assumed: no).

## Decisions

- **`pdfplumber` over PyMuPDF** — PyMuPDF is AGPL, which the client
  cannot inherit in a work-for-hire deliverable. · *Traded away:*
  extraction speed, irrelevant at 200 files.

- **`results.csv` is the only state, and doubles as the resume ledger** —
  no cache file, no database, no sidecar. The operator can open the one
  artefact in Excel and understand the entire state of the run. ·
  *Traded away:* a corrupted or hand-edited CSV corrupts the resume
  logic. Accepted, because deleting a row is also the intended way to
  force a re-parse.

- **Append-only, flushed per row** — a crash at invoice 147 leaves 146
  good rows. · *Traded away:* the ability to correct a row in place, and
  a slightly slower write.

- **One row per invoice, line items as JSON** — the client asked for "a
  results.csv", singular. · *Traded away:* line items are not directly
  filterable in Excel. Mitigated by `--line-items-csv`. See Q1.

- **Single `main.py`, no package layout** — an explicit client
  requirement, and the right call for a tool that must survive being
  emailed to a non-developer. · *Traded away:* unit-testability. Accepted
  for a one-off utility; the function boundaries in the Ownership Map are
  drawn so this is reversible if the client ever wants tests.

- **Strict Structured Outputs, never `json.loads` on free text** —
  contractually required, and the only way the four fields are guaranteed
  present with the right types. · *Traded away:* locks the job to models
  that support strict schema binding.

## Model Corrections

- **Expected** `page.extract_text()` to return readable words. **Actually**
  a real invoice came back as `123InnovationDrive,Suite400` and
  `EnterpriseCloudMigrationServices(Phase1)` — pdfplumber's default
  `x_tolerance=3` merges words whose spacing is tighter than 3 points,
  which is common in the dense header blocks invoices use. The model
  still parsed it correctly, so this was invisible until the *line-item
  descriptions* were inspected in the CSV. **Now assume** extraction
  quality must be eyeballed on real files, never inferred from whether
  the parse succeeded. Fixed with `x_tolerance_ratio=0.15`, which scales
  with font size instead of assuming one — a fixed value that works on
  one supplier's template breaks on another's.

- **Expected** the totals cross-check to compare line sum against the
  stated total directly. **Actually** tax and shipping legitimately push
  the total above the line sum, so a naive equality check flags most
  real invoices. **Now assume** the check is one-directional: flag when
  the line sum *exceeds* the total, or when the total is more than
  double it.

## Known Debt

- No automated tests. Deliberate: the deliverable is a single script for
  a one-off backlog, and the verification method is the seven conditions
  in `project-overview.md` → *Done Looks Like*, run against real files.
- No concurrency. A 200-file run is serial and will take several minutes.
  Deliberate — serial output is legible and debuggable, and rate-limit
  handling for parallel calls is outside the agreed scope.

## Resume Here

Units 01–05 are implemented in `main.py` and verified end to end against
real PDFs — including a deliberately hostile one carrying both a prompt
injection and a spreadsheet formula payload. Both defences held. See
Shipped above for the full check table.

Nothing is in flight. The code is demo-ready and pushed.

Remaining work is not coding work: Q1 and Q3–Q7 are questions for the
client, and Q2 resolves itself on first contact with the real folder.
None of them block a run — every one has a working default, and the
`needs_review` / failure paths surface anything the defaults get wrong
rather than hiding it.

To run: `source .venv/bin/activate`, then `python main.py --dry-run` for
a free census, or `python main.py --limit 1` for a single live parse.
