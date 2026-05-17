# HOUSEKEEPING.md

Folder structure, branch model, and environment conventions for the 5-agent mobile pipeline. See `CLAUDE.md` for the agent roster and `WORKFLOW.md` for phase-by-phase routing.

## Folder structure

```
repo-root/
├── .claude/
│   └── agents/                         ← all agent .md files (team-leader + 5 specialists)
│       ├── team-leader.md
│       ├── engineering-planner.md
│       ├── frontend-developer.md
│       ├── backend-developer.md
│       ├── engineering-reviewer.md
│       └── final-boss.md
│
├── CLAUDE.md                           ← pipeline entry point; lists globals + log files
├── WORKFLOW.md                         ← pipeline phases, gates, document reference
├── GLOBAL_AGENT_RULES.md               ← hard constraints + log schema
├── ERROR_HANDLING.md                   ← error types, retry policy, loop budget, cost ceiling
├── GLOBAL_CODING_STANDARDS.md          ← language-agnostic coding rules
├── HOUSEKEEPING.md                     ← this file
│
├── PIPELINE-STATE.md                   ← append-only event log (format at top of file)
├── PIPELINE-COST.md                    ← append-only cost ledger + ceiling
├── CHANGELOG.md                        ← internal run log + user-facing release notes
├── LESSONS-LEARNED.md                  ← release + orchestration lessons
│
├── .env.example                        ← committed; documents required env vars; no real secrets
├── .env.local                          ← not committed; local dev secrets
├── .env.staging                        ← not committed; staging secrets
├── .env.production                     ← not committed; production secrets
│
├── .github/workflows/                  ← CI/CD; owned exclusively by final-boss after /start-bootstrap emits Day-1 templates. Other agents flag needs via `## CI changes required` tables in their handoff docs.
│
├── docs/
│   ├── PRD_CHECKLIST.md                ← what an adequate PRD must contain
│   ├── DESIGN_SYSTEM.md                ← owned by frontend-developer
│   │
│   ├── plans/<feature>/
│   │   ├── PRD.md                      ← human-authored; /prd-check appends "## Clarifications & Updates"
│   │   ├── PROJECT.md                  ← engineering-planner; compact single-context file all downstream agents read
│   │   ├── BRAINSTORM.md               ← optional, engineering-planner Mode B
│   │   ├── ENGINEERING_PLAN.md         ← engineering-planner Mode A (the "how")
│   │   ├── FRONTEND_NOTES.md           ← frontend-developer
│   │   ├── BACKEND_NOTES.md            ← backend-developer
│   │   └── REVIEW_FINDINGS.md          ← engineering-reviewer (severity ledger, Reviewer-only closure)
│   │
│   ├── releases/<version>/
│   │   ├── RELEASE_CHECKLIST.md        ← final-boss
│   │   ├── ROLLOUT_PLAN.md             ← final-boss
│   │   ├── ROLLBACK_PLAN.md            ← final-boss (kill switch verified)
│   │   └── STORE_NOTES.md              ← final-boss
│   │
│   └── incidents/<id>/
│       ├── HOTFIX_PLAN.md              ← engineering-planner (Hotfix Mode)
│       └── POSTMORTEM.md               ← final-boss (Hotfix Mode close)
│
└── (feature code layout per repo conventions — typically split by directory ownership:
   `frontend/`, `backend/`, and a Planner-frozen `domain/` per feature)
```

### Notes

- **PRD clarifications live inside the PRD.** `/prd-check` appends a `## Clarifications & Updates` section to the PRD itself. **No `CLARIFICATION-NEEDED.md` file is ever created** — the PRD is the single artifact carrying clarifications.
- **`docs/DESIGN_SYSTEM.md`** is the design-system source of truth; `frontend-developer` updates it in the same commit as any shared-primitive change.
- **`LESSONS-LEARNED.md` lives at repo root** — cross-feature, append-only. `final-boss` writes release lessons; `team-leader` writes orchestration lessons.

---

## Git strategy

### Branches

```
main                      → production
feature/<name>            → shared branch for one feature; both frontend-developer and backend-developer commit here
hotfix/<incident-id>      → hotfix branch off main
release/<version>         → release-engineering branch owned by final-boss
```

### Directory ownership on `feature/<name>` (parallel work, no merge conflicts by construction)

| Directory | Owner |
|---|---|
| `frontend/` (per feature) | `frontend-developer` |
| `backend/` (per feature) | `backend-developer` |
| `domain/` (per feature) | **`engineering-planner` only — frozen after Gate 0** |
| `docs/DESIGN_SYSTEM.md` | `frontend-developer` |

A conflict in `domain/` means someone violated the freeze. Stop and re-engage the Planner.

### Full flow

```
[Kickoff]
team-leader runs pre-flight, /prd-check, initialises logs
git checkout main
git checkout -b feature/<name>

[Phase 1 — Planning]
engineering-planner writes docs/plans/<feature>/ENGINEERING_PLAN.md + domain/ stubs + fakes
[Human Gate 0 — Plan approval]

[Phase 2 — Build, parallel]
frontend-developer commits to frontend/ on feature/<name>
backend-developer  commits to backend/  on feature/<name>
(disjoint directories → no merge conflicts)

[Phase 3 — Review]
engineering-reviewer authors non-functional tests on feature/<name>
Loop up to 3 rounds. Findings closed by Reviewer only.
[Human Gate 1 — Dev build approval]

[Phase 4 — Delivery]
git checkout -b release/<version>
final-boss runs release-build pre-ship suite, signs IPA + AAB, verifies kill switch
[Human Gate 2 — Production promotion]
team-leader appends `final-boss-authorized` line to PIPELINE-STATE.md
final-boss merges release/<version> → main, tags, submits to stores, advances staged rollout

[Pipeline close]
final-boss appends release lesson to LESSONS-LEARNED.md
team-leader appends orchestration lesson + final cost summary
team-leader appends `close | complete` line to PIPELINE-STATE.md
Delete feature/<name> and release/<version>
```

### Merge policy

- Merge commits (no squash) — preserves agent-level audit trail
- `final-boss` is the only agent that merges to `main`, and only after the `final-boss-authorized` line is in `PIPELINE-STATE.md`
- Conflicts in `domain/` = halt and re-engage the Planner

### .gitignore additions

```
.env.local
.env.staging
.env.production
```
