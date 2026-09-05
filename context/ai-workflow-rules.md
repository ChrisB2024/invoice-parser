# Working Rules

Direct instructions to the agent. These are rules, not preferences.

## Operating Mode

Build against specs, one unit at a time, in the order in
`context/specs/00-build-plan.md`. Implement what is specified — do not
infer behaviour that is not written down.

**Chris is the architect; you are the implementation engine.** The
default output is structure plus TODOs, not finished logic. Before
writing any block longer than about fifteen lines of real logic, ask
whether he wants to own it.

## Scope Discipline

- One unit at a time. Finish it before starting the next.
- Never combine unrelated boundaries in one step — extraction and the
  API call are two units, not one.
- Do not refactor code the current unit did not touch.
- Do not install a dependency until the unit that needs it. `openai` is
  not installed in Unit 02.
- **The deliverable is one `main.py`.** Never split it into a package,
  never add a `src/` layout, never add a `utils.py`, however tempting.
  If a unit seems to need that, it is the wrong unit.

## Split the Work If

- It crosses two boundaries from the Ownership Map
- It needs a dependency not yet installed
- Part of it is not clearly defined in the context files

If the result cannot be verified end to end in one sitting, split it.

## Missing or Ambiguous Requirements

- Never invent behaviour. A plausible guess is still a guess, and this
  script's whole value is that its output is trustworthy.
- Ambiguous → resolve it in the context file first, then implement.
- Absent → log it under Open Questions in `progress-tracker.md` and ask.
- **Specifically: never invent a field, a column, a CLI flag, or a
  default the spec does not name.** Anything outside the agreed scope is
  a change to be discussed, not assumed.

## Cost Discipline

This is the one rule unique to this project. Every run costs the client
real money, and there are 200 invoices.

- Never write code that calls the API in a loop without a `--limit` path.
- Test against 1–3 fixture PDFs. Never against the full folder.
- Any change to the prompt or the schema invalidates prior results —
  say so before making one.

## When a Correction Fails Twice

Two failed corrections on the same item is a hard stop. There is no third
attempt.

1. Stop editing. Do not revert — the diff is evidence.
2. Compare the two failures. **Different** failures mean the spec is
   ambiguous. **Identical** failures mean the model of the code is wrong.
3. Verify the diverging assumption by running an actual check — print the
   extracted text, print the raw response. Never by reasoning about what
   the code probably does. On this project the answer is almost always
   visible in the extracted text, so look at it first.
4. Report: the failing `file:line`, the exact spec sentence it was meant
   to satisfy, expected vs actual per attempt, and the single question
   that needs answering.
5. Fix the source, then implement once, cleanly.

## Explainability

Chris must be able to explain everything that ships to the client,
including why the CSV sanitisation exists and why the schema is strict.
If he would stumble explaining it, it is not ready — scaffold it instead.

## Protected Files

Do not modify without explicit instruction:

- `invoices/*.pdf` — the operator's source documents, read-only always
- `output/results.csv` — append-only; never rewritten, never truncated
- `.env` — never read, written, or printed by anything but `load_dotenv()`

## Keep the Docs True

Update the relevant context file *before* continuing whenever
implementation changes a boundary, the storage model, an invariant, a
convention, or scope.

## Definition of Done

All three layers pass, or the unit is not done.

**Technical**
1. Works end to end within the unit's stated scope
2. A corrupt PDF, an empty PDF, and a zero-byte file are all handled without crashing
3. `python main.py --help` runs clean, and `python -m py_compile main.py` passes

**Security**
4. The unit's threat model was checked against what was built
5. PDF-derived text is validated or neutralised at every boundary it crosses
6. `grep -rEi "sk-[a-z0-9]" .` returns nothing outside `.env`

**Product**
7. No invariant in `architecture.md` was violated
8. The console output matches the formats in `ui-context.md`
9. `progress-tracker.md` reflects the finished work

A failure in any layer sends the unit back to planning. That is the
process working, not a setback.
