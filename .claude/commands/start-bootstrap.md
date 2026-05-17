---
description: Guided bootstrap + adequacy audit for the repo. On a greenfield repo, interviews the human to pick stack and emits a scaffold plan. On a partial repo, gap-fills the missing pieces. On an adequate repo, runs the audit and greenlights the pipeline. Re-runnable. Halts the pipeline if any blocking checklist item is OPEN.
argument-hint: (none)
allowed-tools: Read, AskUserQuestion, Edit, Write, Bash, Glob, Grep
---

You are running the **repo bootstrap + adequacy audit**. This command makes the repo ready to run the 5-agent pipeline. It is re-runnable: each invocation moves the repo one step closer to adequate, or confirms it already is.

## Inputs

- `docs/REPO_CHECKLIST.md` — the checklist (human-editable). Read it; do not assume.
- The repo itself.
- `CLAUDE.md` — stack declarations, if already present.
- `REPO_STATE.md` — prior audit history.

## Step 1 — Detect repo state

Use `Glob` and `Bash` to classify the repo into one of three modes. Don't ask the human — detect.

| Mode | Heuristic |
|---|---|
| **greenfield** | No lockfile (`package.json`, `Cargo.toml`, `pubspec.yaml`, `go.mod`, `pyproject.toml`, `Package.swift`, etc.). No source dirs. CLAUDE.md has no stack declaration. |
| **partial** | Some scaffold exists but one or more blocking checklist items would fail an audit |
| **adequate** | Run the audit; all blocking items pass |

Record the detected mode in the run summary you write to `REPO_STATE.md` at the end.

## Step 2 — Run the matching flow

### Greenfield flow

Interview the human in batched `AskUserQuestion` calls (max 4 questions per batch). Use the question tree below; branch on each answer. Mark a recommended option `(Recommended)` only when you can justify it briefly to the human.

**Batch 1 — Product shape**
1. What are you building? `mobile app` / `web app` / `CLI` / `library` / `backend service` / `other`
2. Team size? `solo` / `small (2–10)` / `large (10+)`
3. License? `MIT` / `Apache-2.0` / `proprietary` / `decide later`

**Batch 2 — Stack** (branch on Batch 1 product type)
- *Mobile*: cross-platform vs native? Specific framework? Min OS targets?
- *Web*: SSR / SPA / static? Specific framework?
- *CLI*: language?
- *Library*: language + target ecosystem?
- *Backend*: language + framework + database?
- For each, offer 2–4 concrete options. Vendor-neutral.

**Batch 3 — Delivery & environments**
1. Environments? (`dev + prod` / `dev + staging + prod` / `prod only`)
2. Deployment target? (`TestFlight + Play` / `Vercel` / `AWS` / `npm` / `PyPI` / `Docker registry` / `other`)
3. Secrets management? (`dotenv` / `cloud secret manager` / `vault`)

**Batch 4 — Observability & test infra**
1. Crash/error reporter? (`Sentry` / `Crashlytics` / `Rollbar` / `decide later`)
2. Analytics? (`yes — which` / `no`)
3. Test runner? (recommend per stack; offer override)

After the interview, write `docs/BOOTSTRAP_PLAN.md` with three sections:

```markdown
# Bootstrap Plan — <YYYY-MM-DD>

## Stack decisions
- <key>: <value>
- ...

## Commands the human runs (each verified on re-run of /start-bootstrap)
1. `<exact command>` — <one-line purpose>
2. ...

## Manual setup (cannot be automated)
- <step that needs human/external action — accounts, signing, third-party setup>
- ...

## Verification
After completing the above, re-run `/start-bootstrap` to verify.
```

**Auto-run nothing.** Every action is emitted as an instruction for the human. Predictability over convenience.

If you can safely write a config or template file the human will need (CI YAML template, `.env.example`, `.gitignore` additions) — write it via the `Write` tool *as part of the bootstrap plan*, not silently. Mention each file you wrote in the plan.

### Partial flow

Run the audit (Step 3). For each blocking gap, ask the human via `AskUserQuestion` how to resolve it. Group related gaps. If a gap requires significant work (e.g. "no CI"), emit a targeted addendum to `docs/BOOTSTRAP_PLAN.md` rather than asking inline.

### Adequate flow

Run the audit (Step 3). Append a pass entry to `REPO_STATE.md`. Exit.

## Step 3 — Audit against `docs/REPO_CHECKLIST.md`

For each item:
- **Satisfied** — the repo passes
- **Soft gap** — silent miss, low cost; record the assumption being taken
- **Blocking gap** — silent miss, would break the pipeline; must be resolved

Group blocking gaps and ask via `AskUserQuestion` in batches of up to 4. Phrase options concretely.

## Step 4 — Write CLAUDE.md stack section (greenfield only)

After the human confirms stack decisions in Batch 2, append or update a `## Stack` section in `CLAUDE.md`:

```markdown
## Stack

- Language: <value>
- Framework: <value>
- Test runner: <value>
- ...
```

Do not modify other sections of `CLAUDE.md`.

## Step 5 — Append to `REPO_STATE.md`

Append a dated subsection at the bottom. Append-only — never rewrite prior entries.

```markdown
## YYYY-MM-DD — /start-bootstrap run

**Mode:** greenfield | partial | adequate
**Soft assumptions taken:**
- <ref> — <assumption>
**Clarifications received:**
- <ref> — Q: <question> → A: <answer>
**Open items (blocking, unresolved):**
- <ref> — Q: <question> — **OPEN**
**Result:** adequate | bootstrap-plan-emitted | blocked
```

If there are zero open items, omit the "Open items" subsection. If there are open items, the section itself is the halt signal — `team-leader` will refuse to kick off until the next run clears them.

## Outputs

**On adequate result:**
- `REPO_STATE.md` updated with a `## YYYY-MM-DD — /start-bootstrap run` entry, **Result:** adequate
- Report a short summary to the user — the pipeline is ready
- Return control

**On bootstrap-plan-emitted (greenfield or partial):**
- `docs/BOOTSTRAP_PLAN.md` written/updated
- `CLAUDE.md` `## Stack` section written (greenfield)
- `REPO_STATE.md` entry with **Result:** bootstrap-plan-emitted and any OPEN items
- Tell the human exactly what to do next: execute the plan, then re-run `/start-bootstrap`
- Halt

**On blocked (audit failed, gaps unresolved):**
- `REPO_STATE.md` entry with **Result:** blocked and the OPEN items
- Tell the human which items are OPEN and that `team-leader` cannot proceed until resolved
- Halt

## Rules

- **Never silently assume a blocking item.** Soft items can take a default; blockers cannot.
- **Auto-run nothing irreversible.** Write template files when useful; never run installers, scaffolders, or external account setup.
- **Recommend, don't decide.** Stack picks are strategic — present options, let the human choose.
- **Re-runnable, idempotent.** Each run takes the repo one step forward or confirms it's done.
- **Append-only** to `REPO_STATE.md` — never rewrite prior entries.
- **Do not modify** `docs/REPO_CHECKLIST.md` — it is human-editable team standard.
