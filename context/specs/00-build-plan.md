# Build Plan

Five units. Each produces one visible result inside one boundary from the
Ownership Map, and each is one focused session.

## The three subsystems

Identified before ordering, because none can be bolted on later:

- **Value engine** — PDF text → validated `Invoice`. Units 02 and 03.
- **Experience layer** — the CLI, the console contract, the CSV. Units 01 and 04.
- **Trust infrastructure** — key handling, injection defences, resumability,
  the failure ledger. Units 01, 04 and 05.

Trust infrastructure is deliberately *not* last. The key handling and the
fail-fast land in Unit 01 before anything can call the API, and the
sanitisation lands in Unit 04 in the same unit that first writes a CSV —
never as a retrofit.

---

## Unit 01 — CLI skeleton, config, and fail-fast key loading

**Builds:** `main.py` runs. `--help` prints every flag with a
non-developer-readable description. Running with no `OPENAI_API_KEY`
exits `2` with a plain-English message naming `.env.example`, before any
file is touched. `requirements.txt` and `.env.example` exist.

**Depends on:** nothing
**Boundary:** `main()` — orchestration only, no extraction, no API
**Subsystem:** experience layer + trust infrastructure

Flags: `--input`, `--output`, `--limit`, `--dry-run`, `--recursive`,
`--verbose`, `--line-items-csv`. No flag for the API key — flags land in
shell history.

---

## Unit 02 — PDF discovery and text extraction

**Builds:** the script walks the input folder, prints one progress line
per PDF per `ui-context.md`, and reports each file's extracted character
count. A corrupt PDF and a scanned image-only PDF are both classified and
reported by filename. **No API call, no CSV, no `openai` dependency yet.**

**Depends on:** Unit 01
**Boundary:** `discover_pdfs()` + `extract_text()`
**Subsystem:** value engine

This unit is where the real-world surprise lives. Running it against the
client's actual 200 files answers, for free, the question that decides
whether the job is even viable: **how many of them are scanned images?**
Ordered second for exactly that reason — it is the cheapest possible test
of the riskiest assumption.

**Dependency introduced here:** `pdfplumber`.

---

## Unit 03 — Pydantic schema and the Structured Outputs call

**Builds:** `LineItem` and `Invoice` models, the system prompt constant,
and `parse_invoice()`. `python main.py --limit 1` extracts one PDF,
calls the API, and prints the validated object. Still no CSV.

**Depends on:** Unit 02
**Boundary:** `parse_invoice()` — the only function that touches the network
**Subsystem:** value engine

The contracted requirement lives here: `.parse()` with a strict schema,
refusal checked, no `json.loads` on free text anywhere.

**Dependency introduced here:** `openai`, `pydantic`.

---

## Unit 04 — Sanitisation, CSV writer, and resume

**Builds:** `output/results.csv` with the fixed column order, one row per
invoice, flushed per row. Formula-injection neutralisation on every
untrusted cell. The resume set built from the existing CSV at startup, so
a second run of the same command makes zero API calls and writes zero
rows. Optional `--line-items-csv` long-format output.

**Depends on:** Unit 03
**Boundary:** `sanitize_cell()` + `write_rows()`
**Subsystem:** trust infrastructure

Sanitisation and writing are one unit deliberately: the first moment a
CSV exists is the first moment it can carry an injection payload, and
splitting them would mean shipping a knowingly unsafe intermediate.

---

## Unit 05 — Hardening and the full run

**Builds:** retry-with-backoff on transient API errors, the per-file
timeout, `--dry-run` cost estimation, the end-of-run summary block with
the failed-file list, correct exit codes, and `README.md`. Then a real
run against all 200 invoices.

**Depends on:** Unit 04
**Boundary:** `main()`
**Subsystem:** trust infrastructure + experience layer

**Done when:** all seven conditions in `project-overview.md` → *Done
Looks Like* verify against the client's real folder.

---

## Order check

- Every unit's dependencies exist in a previous unit. ✔
- No two adjacent units are always done in one session with no standalone
  result between them — each of the five produces something the operator
  can run and see. ✔
- Dependencies are installed just in time: nothing in Unit 01, `pdfplumber`
  in 02, `openai` + `pydantic` in 03. ✔
- Security precedes the functionality it protects: key handling before any
  call (01 → 03), sanitisation in the same unit as the first write (04). ✔
