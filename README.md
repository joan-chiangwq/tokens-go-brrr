# tokens-go-brrr

A 6-agent TDD build pipeline. One orchestrator routes five specialists through plan → build → review → ship, with three mandatory human gates and a strict contract-first discipline. Designed to make agentic software work reproducible, auditable, and hard to break.

The pipeline's bones are general-purpose; the current agent text leans mobile (iOS + Android), but the structure (orchestrator + planner + parallel builders + adversarial reviewer + delivery gate) works for any software product.

---

## The pipeline at a glance

```
  /start-bootstrap   (Day 1 — repo adequacy + scaffold guide)
        │
        ▼
  /prd-check         (every feature — PRD adequacy audit)
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │ team-leader  (orchestrator; only talks to human)│
  └─────────────────────────────────────────────────┘
        │
        ▼
  engineering-planner  →  PROJECT.md + ENGINEERING_PLAN.md + contracts + fakes
        │
        ▼  [Human Gate 0 — Plan approval — MANDATORY]
        │
        ▼
  frontend-developer  ∥  backend-developer
        │     (parallel — skipped if out of scope per "Feature touches")
        ▼
  engineering-reviewer  →  adversarial tests + REVIEW_FINDINGS.md (severity rubric, 3-round loop budget)
        │
        ▼  [Human Gate 1 — Dev build approval — MANDATORY]
        │
        ▼
  final-boss  →  build, sign, verify on release; CI authoring + smoke check; staged rollout + verified kill switch
        │
        ▼  [Human Gate 2 — Production promotion — MANDATORY]
        │
        ▼  (team-leader writes the final-boss-authorized line; only then irreversible actions are allowed)
  Ship + lessons captured
```

---

## Agents (in `.claude/agents/`)

| Agent | Role |
|---|---|
| `team-leader` | Orchestrator. Routes specialists, surfaces the 3 gates to the human, owns task IDs + cost ceiling, authorizes `final-boss`'s irreversible actions. |
| `engineering-planner` | Single source of truth for contracts. Produces `PROJECT.md` (compact context), `ENGINEERING_PLAN.md` (the *how*), interface stubs, and fake impls. |
| `frontend-developer` | UI / navigation / presentation state. Owns `docs/DESIGN_SYSTEM.md`. Wires Planner's fakes during dev. |
| `backend-developer` | Networking, persistence, auth, migrations. Implements concrete impls behind Planner's interfaces. |
| `engineering-reviewer` | Adversarial test author. Owns *all* non-functional test authoring (edge cases, security, perf, a11y, concurrency, l10n). 3-round loop budget; Reviewer-only closure. |
| `final-boss` | Delivery gate. Runs the existing suite on release builds; signs artifacts; owns CI/build automation; verifies kill switch. Ships only after team-leader's Gate-2 authorization. |

---

## Commands (in `.claude/commands/`)

| Command | When | What it does |
|---|---|---|
| `/start-bootstrap` | Day 1 + on infra changes | Auto-detects greenfield/partial/adequate; interview-led bootstrap on greenfield; audits against `docs/REPO_CHECKLIST.md`; writes `REPO_STATE.md`. `team-leader` refuses to run until result is `adequate`. |
| `/prd-check` | Start of every feature | Audits the PRD against `docs/PRD_CHECKLIST.md`; appends `## Clarifications & Updates` to the PRD itself. `engineering-planner` refuses to run with OPEN items. |

---

## Log files (append-only, flat pipe-delimited)

| File | Purpose | Writers |
|---|---|---|
| `PIPELINE-STATE.md` | Timeline of every agent run + gate decision | All agents + human (gates) |
| `PIPELINE-COST.md` | Token/duration/cost ledger + cost ceiling | All agents (own run) + team-leader (ceiling + running total) |
| `CHANGELOG.md` | Internal run log + user-facing release notes | All agents (internal) + `final-boss` (versioned releases at Gate 2) |
| `LESSONS-LEARNED.md` | Release + orchestration lessons | `final-boss` + `team-leader` only |
| `REPO_STATE.md` | `/start-bootstrap` audit history | `/start-bootstrap` only |

Formats and writer rules are declared at the top of each file. No JSON, no nesting. See `GLOBAL_AGENT_RULES.md` → *Pipeline log files* for the universal writing rule.

---

## Typical flow

### Day 1 (new repo)
1. Human runs `/start-bootstrap`.
2. Command interviews stack + delivery + observability + test infra preferences.
3. Writes `docs/BOOTSTRAP_PLAN.md` with concrete commands to run.
4. Human executes the plan (no auto-runs).
5. Re-runs `/start-bootstrap` until `Result: adequate`.

### Per feature
1. Human writes the PRD at `docs/plans/<feature>/PRD.md`.
2. Runs `/prd-check`. Answers any blocking clarifications inline.
3. Invokes `team-leader`.
4. `team-leader` kicks off `engineering-planner` → `PROJECT.md` + `ENGINEERING_PLAN.md`.
5. **Gate 0**: human approves the plan.
6. FE + BE build in parallel (or one, per `Feature touches`). Reviewer waits on both.
7. Reviewer authors adversarial tests, loops with build agents up to 3 rounds.
8. **Gate 1**: human approves the dev build.
9. `final-boss` builds, signs, verifies the release candidate; runs the existing suite on the release build.
10. **Gate 2**: human approves production promotion.
11. `team-leader` writes the `final-boss-authorized` line; only then can `final-boss` merge, tag, and ship.
12. `final-boss` writes the release lesson; `team-leader` writes the orchestration lesson; pipeline closes.

---

## Key principles

1. **Contract-first.** Planner's interfaces are the truth. Builders implement; they never extend silently.
2. **TDD strict.** RED-first; commit failing tests before implementation.
3. **No silent failures.** Every error emits a structured envelope per `ERROR_HANDLING.md`.
4. **Three mandatory human gates.** Never skipped, including in Hotfix Mode.
5. **Single-source-of-truth templates.** Log formats live in the log files; agent files reference, never restate.
6. **Append-only.** No agent rewrites prior entries in any state file.
7. **Reviewer-only closure.** Findings flip `open → fixed` only by Reviewer's re-run. Self-attestation forbidden.
8. **Final-boss authorization line.** Irreversible actions (merge, tag, store submission, rollout %) require Gate-2's line in `PIPELINE-STATE.md`.
9. **Loop budget.** 3 unresolved Reviewer rounds → escalate to Planner for scope cut or contract change.
10. **Lessons compound.** Every Gate 2 closure and every triage correction writes to `LESSONS-LEARNED.md` immediately.

---

## File map

```
.
├── README.md                       ← you are here
├── CLAUDE.md                       ← pipeline entry point; lists globals + log files
├── WORKFLOW.md                     ← phases, gates, routing, document reference
├── HOUSEKEEPING.md                 ← folder structure + branch model
├── GLOBAL_AGENT_RULES.md           ← hard constraints + structured log rule
├── GLOBAL_CODING_STANDARDS.md      ← language-agnostic coding rules
├── ERROR_HANDLING.md               ← error envelope, retry policy, loop budget
│
├── PIPELINE-STATE.md               ← event log
├── PIPELINE-COST.md                ← cost ledger
├── CHANGELOG.md                    ← internal log + release notes
├── LESSONS-LEARNED.md              ← institutional memory
├── REPO_STATE.md                   ← /start-bootstrap audit log
│
├── docs/
│   ├── PRD_CHECKLIST.md            ← what an adequate PRD must contain
│   ├── REPO_CHECKLIST.md           ← what an adequate repo must contain
│   └── plans/<feature>/            ← per-feature artifacts (PRD, PROJECT, plan, notes, findings)
│
└── .claude/
    ├── agents/                     ← 6 agent files (1 orchestrator + 5 specialists)
    └── commands/                   ← /start-bootstrap, /prd-check
```

---

## Status

The base workflow handles:
- Greenfield repo bootstrap (via `/start-bootstrap`)
- Full-stack feature development
- Backend-only feature updates (via PROJECT.md `Feature touches`)
- CI authoring + maintenance owned by `final-boss` with feasibility review
- Hotfix Mode (all 3 gates preserved)

Specialization for any one product type (mobile, web, CLI, library) is intended to live in overlay docs that don't modify the base.
