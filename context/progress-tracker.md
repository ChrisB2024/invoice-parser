# Progress

Update after every meaningful change. This file is how a cold session
recovers full context in one read.

## Phase

Foundations — context files written, nothing implemented.

## Working On

Nothing yet. Next action is Unit 01.

## Shipped

- Context system scaffolded: six files, build plan, spec for Unit 01
- Repo skeleton: `main.py` (structure + TODOs), `requirements.txt`,
  `.env.example`, `.gitignore`, `README.md`

## In Progress

- Nothing

## Next

- **Unit 01 — CLI skeleton, config, and fail-fast key loading.**
  Spec at `context/specs/01-cli-and-config.md`.

## Open Questions

The agent must not answer these. Several are questions for the *client*,
not for Chris, and the answer to Q2 may change the price of the job.

- **Q1 — Line-item CSV shape.** The brief names "Line Items" as one of
  four fields, but a CSV cell is flat and an invoice has N line items.
  Default taken: one row per invoice, line items as a JSON string in
  `line_items_json`, plus `line_item_count`, plus an optional
  `--line-items-csv` long-format second file. *Blocks:* nothing — the
  default ships. But confirm before delivery, because reshaping the
  output after the client has opened it in Excel is a rework cycle.

- **Q2 — How many of the 200 PDFs are scanned images?** `pdfplumber`
  returns empty text for a scan, and OCR is explicitly out of scope at
  this price. *Blocks:* whether the $50 scope is achievable at all. Unit
  02 answers this for free, before any API spend — **run it against the
  real folder before quoting any change.**

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

Nothing yet. First entry is expected from Unit 02 — the gap between
"pdfplumber extracts the text" and what a real supplier invoice's table
layout actually produces.

## Known Debt

- No automated tests. Deliberate: the deliverable is a single script for
  a one-off backlog, and the verification method is the seven conditions
  in `project-overview.md` → *Done Looks Like*, run against real files.
- No concurrency. A 200-file run is serial and will take several minutes.
  Deliberate — serial output is legible, and rate-limit handling for
  parallel calls is scope the price does not cover.

## Resume Here

Context system is complete; no code is implemented. `main.py` currently
holds structure, the finished Pydantic models, and TODOs keyed to unit
numbers — the function bodies are deliberately empty for Chris to fill.

Start by reading `context/specs/01-cli-and-config.md` and implementing
Unit 01.

Before any API spend, run Unit 02 against the client's real folder — Q2
is the question that decides whether this job is a $50 job.
