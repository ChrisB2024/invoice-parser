## Project Context

Read these before implementing anything or making any
architectural decision:

1. `context/project-overview.md` — the problem, who has it,
   scope boundaries, and what done looks like
2. `context/architecture.md` — stack, ownership map, data flow,
   trust boundaries, and the invariants that must never break
3. `context/ui-context.md` — tokens, typography, required states,
   and latency behavior
4. `context/code-standards.md` — security non-negotiables,
   conventions, and coupling rules
5. `context/ai-workflow-rules.md` — how to scope, split, verify,
   and when to stop
6. `context/progress-tracker.md` — where the work stands, what
   was decided and why, and what is still open

Then read the spec for the unit being built, in `context/specs/`.

Implement what is specified. Do not invent product behavior that
is not written down — if something is missing or ambiguous, ask,
or log it under Open Questions.

Update `context/progress-tracker.md` after every meaningful
change.

If implementation changes an invariant, a boundary, the storage
model, scope, or a convention, update the relevant context file
before continuing.

## This Project

A one-off utility script, fixed-price. Three rules override anything
inferred from general Python practice:

1. **The deliverable is a single `main.py`.** Never split it into a
   package or add helper modules.
2. **Every run costs the client money.** Test with `--limit 1`, never
   against the full folder. Never write an API call that has no limit
   path.
3. **A blank cell is recoverable; a confidently wrong total is not.**
   When the document does not state something, the answer is `null` —
   never an inference.

Scaffold-first: produce structure and TODOs. Chris writes the
implementation, you review it.
