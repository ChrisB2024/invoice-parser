# Unit 01: CLI Skeleton, Config, and Fail-Fast Key Loading

## Goal

`main.py` runs end to end as a no-op program: `--help` prints every flag
with a description a non-developer can act on, config is resolved from
flags plus `.env`, and a missing `OPENAI_API_KEY` exits `2` with a plain
message **before any file is opened or any API client is constructed**.
No PDF is read and no API call is made in this unit.

## Design

Console output follows `ui-context.md` exactly — the `log()` helper and
the six prefixes are built here, because every later unit depends on
them.

`--help` text is written for the operator described in
`project-overview.md`: a finance team member, not a developer. So
`--limit N` reads "only process the first N invoices (use this for a
cheap test run first)", not "limit iterations".

Startup failure format, on stderr, exit `2`:

```
FAIL  No OPENAI_API_KEY found.

      Copy .env.example to .env and paste your OpenAI key into it:
        cp .env.example .env

      Then run this command again.
```

Names the fix, not just the problem. This is the first thing the operator
will hit, and it decides whether they email Chris or solve it themselves.

## Implementation

### Module docstring

The client's deliverable requires the run instructions at the top of the
code. Three steps: create venv + install, add key to `.env`, run. Verbatim
match with `README.md` — if they drift, the docstring is authoritative
because it is the one that ships inside the file.

### Constants

`MODEL`, `MAX_CHARS`, `MAX_PAGES`, `MAX_FILE_BYTES`, `CSV_FIELDNAMES`,
`SYSTEM_PROMPT`. Defined now, used later. `CSV_FIELDNAMES` fixes column
order as part of the deliverable.

### `log(level, filename, message)`

The single output function. Enforces the column widths from
`ui-context.md`: 9 for the counter, 6 for the level, 32 for the filename.
Long filenames push the message right, never truncate. Writes to stdout;
`FAIL` also to stderr. No colour, no emoji.

### `parse_args()`

`argparse` with: `--input` (default `invoices`), `--output` (default
`output/results.csv`), `--limit` (int, default None), `--dry-run`
(flag), `--recursive` (flag), `--verbose` (flag), `--line-items-csv`
(optional path). No `--api-key` flag — flags land in shell history.

### `load_config(args)`

`load_dotenv()`, then read `OPENAI_API_KEY` from the environment. Missing
or empty → the message above, `sys.exit(2)`. Validate `--input` exists
and is a directory → same treatment. Returns a config object; the key is
**not** stored on it — it stays in the environment for the SDK to read,
so it cannot be reached by anything that logs a config dump.

### `main()`

Wire the above, print a one-line run header naming the resolved input and
output paths, exit `0`. Everything else is a TODO keyed to its unit
number.

## Failure Modes

1. **Invalid input** — a `--input` path that does not exist, or is a
   file: caught in `load_config`, exit `2`, message names the flag.
2. **Dependency unavailable** — only `python-dotenv` is imported. If it
   is missing the import fails with pip's own error, which is already
   actionable. No extra handling.
3. **Slow** — nothing in this unit blocks.
4. **Compromised** — the only asset is the key. It is read from the
   environment, never stored on the config object, never printed. A
   `--verbose` config dump must not be able to reach it.
5. **What the operator sees on failure** — a `FAIL` block naming the
   exact command that fixes it. Never a traceback.

## Threat Model

- **Data handled:** the `OPENAI_API_KEY`, and two filesystem paths.
- **Who accesses it:** the operator, and anything that can read their
  `.env` or their process environment.
- **If an attacker controls the input:** the only inputs are CLI flags
  and the `.env` file, both operator-supplied. Paths go through
  `pathlib`, are never shelled out, and are never interpolated into a
  string that reaches a subprocess. There is no subprocess.
- **If storage is breached:** `.env` leaks the key → billing abuse on the
  client's OpenAI account. Mitigated by `.gitignore`, by recommending a
  dedicated project-scoped key in the README, and by telling the client
  to revoke it when the backlog closes.

## Dependencies

- `python-dotenv` (lets a non-developer paste a key into a file instead
  of learning shell exports)

`pdfplumber`, `openai` and `pydantic` are **not** installed in this unit.

## Verify When Done

**Technical**
- [ ] `python main.py --help` lists all seven flags with readable descriptions
- [ ] `python -m py_compile main.py` passes
- [ ] `python main.py --input does-not-exist` exits `2` with a message naming `--input`
- [ ] With a key present and a valid `--input`, the script prints its run header and exits `0`

**Security**
- [ ] `OPENAI_API_KEY` unset → exit `2`, message names `.env.example`, no traceback
- [ ] The key is not an attribute of the config object
- [ ] `grep -n "sk-" main.py .env.example` returns no real key
- [ ] `.gitignore` covers `.env`, `invoices/`, `output/`

**Product**
- [ ] Output matches the formats in `ui-context.md` — no colour, no emoji, aligned columns
- [ ] The error message tells the operator the command to run next
- [ ] `progress-tracker.md` marks Unit 01 shipped
