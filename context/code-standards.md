# Code Standards

## Security — non-negotiable

1. **Authentication** — not applicable. There are no users and no sessions.
   The only credential is the OpenAI API key.
2. **Authorization** — not applicable. Access control is the operator's
   filesystem permissions.
3. **Input validation** — enforced at every trust boundary in
   `architecture.md`. PDF text is hostile: capped in length, never placed
   in the system prompt, never `eval`'d, never used to build a path.
   API responses are hostile: parsed through Pydantic, refusal checked.
4. **Secrets** — `OPENAI_API_KEY` from the environment only. Never a
   default value, never a CLI flag (flags land in shell history), never
   printed, never in an exception message. Missing key = fail fast at
   startup.
5. **Transport** — HTTPS via the SDK default. The base URL is never
   overridden.
6. **Dependencies** — four, all pinned with `>=` floors and `<` ceilings
   in `requirements.txt`, each justified in the `architecture.md` stack
   table. Adding a fifth requires a decision entry in `progress-tracker.md`.

## Boundaries and Coupling

- Function boundaries from the Ownership Map are real. `extract_text()`
  must be callable with no network, and `parse_invoice()` must be
  callable with a plain string and no filesystem.
- `main()` holds all orchestration. No other function calls another
  top-level function, so each is independently testable and independently
  replaceable.
- No global mutable state except the module-level `OpenAI` client and the
  argparse config, both initialised once in `main()` and passed down as
  parameters. **Never reach for a module-level global to avoid threading
  a parameter through** — that is coupling by convenience.

## Naming

- Names state intent: `extract_text_from_pdf`, not `process`.
  `sanitize_cell_for_spreadsheet`, not `clean`.
- `snake_case` for functions and variables, `PascalCase` for Pydantic
  models, `SCREAMING_SNAKE` for module constants.
- Domain vocabulary — use these exact words, do not invent synonyms:
  **invoice** (the document), **line item** (a row within it), **vendor**
  (who issued it), **source file** (the PDF's filename, the primary key),
  **parse status** (the per-row outcome), **run** (one execution).

## Types

- Type hints on every function signature, including `-> None`.
- Pydantic v2 models are the only place external data becomes trusted.
  Nothing downstream re-validates, and nothing upstream is trusted.
- `Optional[X]` / `X | None` is used deliberately and means "the document
  did not state this", never "we failed to look".
- No `# type: ignore` without a comment naming the reason.

## Python Conventions

- Python 3.10+ syntax. `pathlib.Path` everywhere; never `os.path` string
  concatenation.
- One file, `main.py`. Section it with banner comments in the order:
  constants → models → extraction → parsing → sanitising → writing →
  orchestration → `if __name__ == "__main__"`.
- No class where a function will do. The only classes are the Pydantic
  models.
- The run instructions live in the module docstring at the very top of
  `main.py`, per the client's explicit deliverable requirement.

## The OpenAI Call

- Structured Outputs only — the SDK's `.parse()` with a Pydantic model.
  Never `json.loads()` on a free-text response, never
  `response_format={"type": "json_object"}` without a schema, never a
  regex over model output. This is the contracted requirement.
- Check `.refusal` before touching `.parsed`.
- Model id in a single module constant, `MODEL`, so it is changed in one
  place.
- `temperature=0` — this is extraction, not generation.
- Every call wrapped in retry-with-backoff: 3 attempts, exponential,
  retrying only on timeout / rate limit / 5xx. A 400 is a bug, not a
  transient, and must not be retried.
- The system prompt is a module constant. It never contains any part of
  the invoice text.

## Errors and Failure

- Catch narrow, at the per-file level in `main()`. Never a bare
  `except:` and never `except Exception` outside that one loop.
- What the operator sees: a `FAIL` line with the filename and one
  plain-English sentence. What is logged under `--verbose`: the full
  traceback.
- Never log the request payload, the response body, or any header — the
  first contains invoice data, the last contains the key.
- Exit codes: `0` all files accounted for; `1` one or more failures;
  `2` startup failure (no key, bad path).

## Data Access

- `results.csv` is opened in append mode, written, and flushed per row.
  It is never read and rewritten.
- The resume set is read once at startup into a `set[str]` of
  `source_file` values, and added to in memory as rows are written.
- `csv.DictWriter` with an explicit `fieldnames` constant. Column order is
  part of the deliverable and must not depend on dict ordering.

## Output Formatting

- All console output goes through the single `log()` helper defined in
  `ui-context.md`. A bare `print()` outside it is a defect.
- Dates are ISO 8601 (`YYYY-MM-DD`) in the CSV, always. The model is
  instructed to normalise, and the value is re-validated on the way in —
  US and EU invoices both appear in this backlog and `03/04/2024` is
  ambiguous.
- Amounts are written as plain decimal strings with no thousands
  separators and no currency symbol. Currency is its own column.

## File Organization

- `main.py` — the entire program
- `requirements.txt` — four pinned dependencies
- `.env.example` — key placeholder, committed; `.env` is not
- `README.md` — the same instructions as the docstring, for whoever
  never opens the code
- `invoices/` — operator's input, git-ignored
- `output/` — generated, git-ignored
- `context/` — this system; not part of the client deliverable

## Do Not Touch

- Nothing generated or vendored yet.
- `invoices/*.pdf` — read-only, always. The script has no business
  writing to its own input.
