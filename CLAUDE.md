# CLAUDE.md

Entry point for the 5-agent mobile-app TDD pipeline. Every agent reads this before executing.

## The pipeline

Orchestrator: **`team-leader`** — the only agent that talks to the human and the only agent that authorizes `final-boss` to take irreversible release actions. Every run enters here.

Specialists (invoked by `team-leader`):
1. `engineering-planner` — contracts, test surface, threat model, fakes
2. `frontend-developer` — UI, navigation, presentation state, design system
3. `backend-developer` — networking, persistence, auth, migrations
4. `engineering-reviewer` — non-functional test authoring, severity rubric, 3-round loop
5. `final-boss` — release-build verification, signing, staged rollout, kill switch

Pipeline phases, gates, routing, and document reference: see `WORKFLOW.md`.

## Required reading (every agent, every invocation)

- `WORKFLOW.md` — pipeline phases, gates, document reference
- `GLOBAL_AGENT_RULES.md` — hard constraints (Single Responsibility, No Silent Failures, No Scope Creep, etc.) + structured log schema
- `ERROR_HANDLING.md` — canonical error types, retry policy, loop budget, cost ceiling
- `HOUSEKEEPING.md` — folder structure, branch model
- `GLOBAL_CODING_STANDARDS.md` — language-agnostic coding rules (build + review agents)

## Log files (append-only, format declared at top of each file)

- `PIPELINE-STATE.md` — timeline of every agent run + gate decision
- `PIPELINE-COST.md` — token/duration/cost ledger + ceiling
- `CHANGELOG.md` — internal run entries + user-facing release notes (final-boss promotes at Gate 2)
- `LESSONS-LEARNED.md` — release lessons (`final-boss`) and orchestration lessons (`team-leader`)

Each file declares its own format and writer rules at the top. Agents must not invent new formats, write JSON, or nest.

## Task IDs

`team-leader` assigns `{feature_id}-{agent_name}-{run_n}` at every invocation per `GLOBAL_AGENT_RULES.md`. Agents use the provided task_id in every error emission and every log line — never generate their own.

## Human Gates (mandatory, never skipped)

Gate 0 (Plan), Gate 1 (Dev build / Review clean), Gate 2 (Production promotion). Hotfix Mode preserves all three. Only `team-leader` surfaces gates to the human; only the human approves them.

## Adequacy audits (must pass before the pipeline runs)

- `/start-bootstrap` — repo adequacy. Auto-detects greenfield vs partial vs adequate; guides bootstrap on greenfield; audits against `docs/REPO_CHECKLIST.md`. Most recent result in `REPO_STATE.md` must be `adequate` before `team-leader` will kick off.
- `/prd-check` — PRD adequacy. Audits the feature PRD against `docs/PRD_CHECKLIST.md`. Must run clean (no OPEN items in `## Clarifications & Updates`) before `engineering-planner` proceeds.
