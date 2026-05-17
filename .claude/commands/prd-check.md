---
description: Audit a PRD against docs/PRD_CHECKLIST.md before planning. Asks the human for missing blocking info via AskUserQuestion and writes the clarifications back into the PRD as a Clarifications & Updates section. Halts the pipeline if any blocking gap remains OPEN.
argument-hint: [path-to-prd]
allowed-tools: Read, AskUserQuestion, Edit, Write, Bash
---

You are running a **PRD adequacy audit** before any planning begins. This is the first step of `engineering-planner` Plan mode, and it can also be invoked directly by a human (`/prd-check <path>`) before kicking off a feature.

## Inputs

- `$1` — path to the PRD. If omitted, default to `docs/plans/<feature>/PRD.md`; if that doesn't exist, ask the user for the path.
- `docs/PRD_CHECKLIST.md` — the checklist to audit against. Read it; do not assume.

## Procedure

1. Read the PRD and `docs/PRD_CHECKLIST.md`.
2. For each item in the checklist, classify:
   - **Satisfied** — the PRD answers it clearly, OR the PRD explicitly defers to a team default
   - **Soft gap** — silent in the PRD, but a reasonable default exists; record the assumption
   - **Blocking gap** — silent in the PRD AND the answer would materially change architecture / contracts / test surface
3. For blocking gaps:
   - Group related questions and use `AskUserQuestion` in batches of up to 4 — one batch per topic area where possible
   - Phrase each option concretely (no "tell me about X"); offer 2–4 plausible answers per question
   - Mark options "(Recommended)" only when you have a defensible reason
4. Append a **`## Clarifications & Updates`** section to the end of the PRD (or update it if it already exists) recording:
   - Date of the audit
   - Soft assumptions taken (auto — no human asked)
   - Each blocking gap, with the question asked and the human's answer
   - Any blocker the human declined to resolve, marked **`OPEN`** with the exact question still pending
5. If any item is marked `OPEN` in the Clarifications section, halt — the Planner does not proceed until the human resolves it (by editing the PRD and re-running `/prd-check`).

## Writing to the PRD

- **Append-only at the bottom of the PRD.** Never modify existing PRD content above this section.
- If a `## Clarifications & Updates` section already exists, add a new dated subsection beneath the previous ones — do not rewrite prior entries.
- Use this structure:

```markdown
## Clarifications & Updates

### YYYY-MM-DD — `/prd-check` audit

**Soft assumptions taken (no human input needed):**
- <Checklist ref> — <Assumption>
- ...

**Clarifications received:**
- <Checklist ref> — Q: <question> → A: <human's answer>
- ...

**Open items (blocking, unresolved):**
- <Checklist ref> — Q: <question> — **OPEN**
- ...
```

If there are zero open items, omit the "Open items" subsection. If there are open items, the section itself is the halt signal.

## Outputs

**If all blocking items are resolved:**
- PRD updated with the new `## Clarifications & Updates` subsection
- Append a line to `PIPELINE-STATE.md`: `YYYY-MM-DD HH:MM | prd-check | Phase 0 | adequate | <feature> | <prd-path>`
- Report a short summary to the user echoing the assumptions taken and the clarifications received
- Return control — adequate, ready to plan.

**If any blocking item remains OPEN:**
- PRD updated with the new `## Clarifications & Updates` subsection, including the OPEN entries
- Append a line to `PIPELINE-STATE.md`: `YYYY-MM-DD HH:MM | prd-check | Phase 0 | blocked | <feature> | <prd-path> (open items)`
- Tell the user exactly which items are OPEN and that the Planner cannot proceed until the PRD is edited to resolve them
- Halt.

## Rules

- **Never silently assume a blocking item.** Soft items can take a default; blockers cannot.
- **Never rewrite or delete prior PRD content.** Only append the Clarifications section at the bottom.
- **Never lower a Blocking item to Soft to avoid asking.** The classification in `PRD_CHECKLIST.md` is binding.
- **Group questions** to respect the human's time — one or two `AskUserQuestion` batches, not one call per gap.
- **Append-only** to `PIPELINE-STATE.md` — never rewrite prior entries.
- The PRD is the single artifact carrying clarifications. Do **not** create a separate `CLARIFICATION_NEEDED.md` file.
