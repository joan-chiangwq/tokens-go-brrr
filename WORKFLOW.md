# Mobile Agentic Workflow Reference

A 5-agent TDD pipeline for building cross-platform mobile apps (iOS + Android). Framework-agnostic — adapts to whatever stack the repo defines (SwiftUI, Jetpack Compose, Flutter, React Native, etc.).

---

## Entry point

Every pipeline run enters through **`team-leader`** (orchestrator). `team-leader` is the only agent that talks to the human and the only agent that authorizes `final-boss` to take irreversible release actions. See `.claude/agents/team-leader.md` for its three modes (Kickoff, Triage, Pipeline Close).

## Universal inputs (required reading for every agent)

Globals — referenced, not duplicated. Each agent's file lists which apply to it.

- `CLAUDE.md` — pipeline entry point; lists globals + log files
- `GLOBAL_AGENT_RULES.md` — hard constraints + structured log schema
- `ERROR_HANDLING.md` — error types, retry policy, loop budget (3-round Reviewer cap), cost ceiling
- `GLOBAL_CODING_STANDARDS.md` — language-agnostic coding rules (build + review agents)
- `HOUSEKEEPING.md` — folder structure + branch model
- `WORKFLOW.md` — this file

Per-feature inputs:

- PRD at `docs/plans/<feature>/PRD.md` — `/prd-check` appends `## Clarifications & Updates` here
- `docs/DESIGN_SYSTEM.md` — owned by `frontend-developer`
- `docs/PRD_CHECKLIST.md` — what an adequate PRD must contain

Append-only log files (formats and writers declared at the top of each file):

- `PIPELINE-STATE.md` · `PIPELINE-COST.md` · `CHANGELOG.md` · `LESSONS-LEARNED.md`

---

## Pipeline Overview

```
PRD.md + CLAUDE.md + docs/DESIGN_SYSTEM.md
        │
        ▼
/prd-check  (audits against docs/PRD_CHECKLIST.md)
  → asks human for blocking gaps via AskUserQuestion
  → appends "## Clarifications & Updates" section to the PRD
  → adequate                 → proceed
  → any item marked OPEN     → HALT (human edits PRD, re-runs /prd-check)
        │
        ▼
engineering-planner
  Mode A (default): Plan  → docs/plans/<feature>/PROJECT.md          (compact single-context file)
                          + docs/plans/<feature>/ENGINEERING_PLAN.md (the "how")
                          + domain interface stubs
                          + fake/in-memory impls
  Mode B (opt-in):  Brainstorm → docs/plans/<feature>/BRAINSTORM.md
        │
        ▼
[Human Gate 0 — Plan approval — MANDATORY, NEVER SKIPPED]
  → Approve     → proceed to build
  → Modify      → re-invoke engineering-planner with feedback
  → Restart     → return to PRD
        │
        ▼
frontend-developer ──────────────────── backend-developer
→ frontend/ code                        → backend/ code
→ feature tests (RED → GREEN)           → repo tests (RED → GREEN)
→ docs/DESIGN_SYSTEM.md updates         → migration scripts (if needed)
→ docs/plans/<feature>/FRONTEND_NOTES.md → docs/plans/<feature>/BACKEND_NOTES.md
        (parallel — both consume the Planner's interfaces;
         Frontend wires fakes until Backend ships real impls)

        ⚠ Scope-conditional per PROJECT.md `Feature touches`:
          - `frontend + backend` (default) → both run
          - `backend only`                 → frontend-developer skipped
          - `frontend only`                → backend-developer skipped
        The existing suite of the skipped layer still runs at pre-ship.
        │
        ▼
engineering-reviewer
  → Additional failing tests (edge cases, security, perf, a11y, concurrency, l10n)
  → docs/plans/<feature>/REVIEW_FINDINGS.md (severity ledger)
  → Loops findings back to Frontend / Backend
  → 3 unresolved rounds → escalate to engineering-planner
        │
        ▼ (zero open Blockers; Majors fixed or waived)
[Human Gate 1 — approve to enter pre-ship]
        │
        ▼
final-boss
  → Runs unit + regression + evals on the release build, real devices
  → Build, sign, validate IPA + AAB
  → docs/releases/<version>/RELEASE_CHECKLIST.md
  → docs/releases/<version>/ROLLOUT_PLAN.md
  → docs/releases/<version>/ROLLBACK_PLAN.md
  → docs/releases/<version>/STORE_NOTES.md
  → CHANGELOG.md updated
  → Submit to TestFlight + Play internal track
        │
        ▼
[Human Gate 2 — review staged rollout metrics, approve production promotion]
        │
        ▼
final-boss (staged rollout & production promotion)
  → Advance Play % per ROLLOUT_PLAN.md
  → Promote TestFlight → App Store production
  → Version tag
```

---

## Agent Roles at a Glance

| # | Agent | Phase | Owns |
|---|---|---|---|
| 1 | engineering-planner | Planning | Specs, contracts (interfaces + models), test surface, threat model, analytics, flags, fakes |
| 2 | frontend-developer | Build | UI, navigation, presentation state, design-system stewardship, l10n wiring |
| 3 | backend-developer | Build | Networking, repositories, persistence, caching, auth lifecycle, error mapping, migrations |
| 4 | engineering-reviewer | Quality Gate | All non-functional test authoring; finds + encodes bugs; severity rubric |
| 5 | final-boss | Delivery Gate | Release-build verification; signing; store submission; rollout + rollback |

---

## Phase 0 — PRD Adequacy

### `/prd-check` (slash command)

Runs before the Planner produces anything. Audits the PRD against `docs/PRD_CHECKLIST.md`; asks the human for missing blocking info; either greenlights planning or halts with `CLARIFICATION_NEEDED.md`. Invoked automatically by `engineering-planner` at the start of every Plan-mode run, or directly by a human via `/prd-check <path>`.

| | |
|---|---|
| **Inputs** | PRD (default: `docs/plans/<feature>/PRD.md`), `docs/PRD_CHECKLIST.md` |
| **Outputs (adequate)** | New `## Clarifications & Updates` section appended to the PRD; `PIPELINE-STATE.md` entry; planner proceeds |
| **Outputs (blocked)** | Same Clarifications section in the PRD with `OPEN` items; `PIPELINE-STATE.md` entry; planner halts until human edits PRD and re-runs `/prd-check` |

The checklist (`docs/PRD_CHECKLIST.md`) is human-editable: each item is marked **B** (blocking) or **S** (soft). Blockers must be answered by the human; soft items can be defaulted with the Planner's assumption logged.

**The PRD itself is the single artifact that accumulates clarifications.** No separate `CLARIFICATION_NEEDED.md` file is ever created. Each audit appends a dated subsection under `## Clarifications & Updates` — prior entries are never rewritten.

---

## Phase 1 — Planning

### engineering-planner

| | |
|---|---|
| **Inputs** | `PRD.md`, `docs/DESIGN_SYSTEM.md`, `CLAUDE.md`, repo state |
| **Outputs (Plan mode — default)** | `docs/plans/<feature>/PROJECT.md` (compact single-context file), `docs/plans/<feature>/ENGINEERING_PLAN.md` (the "how"), interface stubs, fake impls |
| **Outputs (Brainstorm mode — opt-in)** | `docs/plans/<feature>/BRAINSTORM.md` |
| **Routing decisions** | Stack adapters, archetype confirmation, scope cut recommendations |

**ENGINEERING_PLAN.md contains:**
- Scope and Given/When/Then acceptance criteria
- Architecture + state-management conventions
- Contracts (interface names + signatures; code in domain layer)
- Test surface / coverage matrix
- Threat model (if auth / PII / deep links / WebView)
- Analytics event table
- Feature-flag plan
- Migration plan (if persisted schema touched)
- Localization decision
- Required packages/SDKs with versions

**Brainstorming mode** is only entered when the user explicitly asks. Produces 2–4 distinct options with tradeoffs, assumptions, and a recommendation — no code.

---

## Human Gate 0 — Plan Approval

**MANDATORY. NEVER SKIPPED. NO EXCEPTIONS.**

After `ENGINEERING_PLAN.md` is produced, the human reviews before any build begins — regardless of feature size, familiarity, or perceived simplicity. No agent has authority to bypass this gate.

**Gate contents:**
- `ENGINEERING_PLAN.md` (full)
- Domain interface stubs (link to code)
- Threat model section (if present)
- Plain-English summary: *"This is what we plan to build. Before any production code is written, confirm this is correct."*

**Human decisions:**

| Decision | Action |
|---|---|
| Approve | Proceed to Phase 2 |
| Modify | Re-invoke `engineering-planner` with feedback |
| Restart | Return to PRD; re-run Planner |

It is far cheaper to revise the plan than to rewrite implementation.

---

## Phase 2 — Build (parallel)

### frontend-developer

| | |
|---|---|
| **Inputs** | `ENGINEERING_PLAN.md`, interface stubs, fake impls, `docs/DESIGN_SYSTEM.md` |
| **Outputs** | Presentation-layer code, feature tests, `docs/DESIGN_SYSTEM.md` updates, `FRONTEND_NOTES.md` |
| **Branch** | `feature/<name>` (Frontend commits) |

Wires Planner's fake impls during development; swap to real impls is a one-line DI change once Backend lands. Maintains `docs/DESIGN_SYSTEM.md` in the same commit as any shared-primitive change. Reuse-first: no duplicate components.

### backend-developer

| | |
|---|---|
| **Inputs** | `ENGINEERING_PLAN.md`, interface stubs |
| **Outputs** | Data-layer code (repos, clients, persistence), unit + integration tests, migration scripts, `BACKEND_NOTES.md` |
| **Branch** | `feature/<name>` (Backend commits) |

Integration tests must hit a real local server or emulator (Firebase Emulator, WireMock, in-memory SQLite, etc.) — **never mocks**. Auth refresh must be race-free under concurrent 401s. Token storage uses platform secure storage only.

**Branch & coordination model:**
- Frontend owns `lib/features/<feature>/frontend/` (or repo equivalent)
- Backend owns `lib/features/<feature>/backend/`
- `lib/features/<feature>/domain/` is **Planner-owned and frozen** after Gate 0 — neither dev modifies it without re-engaging the Planner
- Both commit to `feature/<name>`. Because directory ownership is disjoint, merge conflicts on feature code are impossible by construction. A conflict in `domain/` means someone violated the freeze — stop and re-engage the Planner.
- **`engineering-reviewer` is the synchronization point.** It does not begin until BOTH `FRONTEND_NOTES.md` AND `BACKEND_NOTES.md` exist AND the branch is green on the unit suite.

**Parallelism note:** Frontend and Backend depend on the Planner's interfaces. If either discovers a contract gap, work stops — re-engage `engineering-planner` rather than silently extending.

---

## Phase 3 — Review

### engineering-reviewer

| | |
|---|---|
| **Inputs** | `ENGINEERING_PLAN.md`, `FRONTEND_NOTES.md`, `BACKEND_NOTES.md`, feature code, existing test suite, `docs/DESIGN_SYSTEM.md` |
| **Outputs** | Additional failing tests (`[RED-review]` marker), `docs/plans/<feature>/REVIEW_FINDINGS.md` |

Writes tests against:
- Edge cases (empty, null, unicode, RTL, extreme numerics, time)
- Network failure modes (offline, slow, 401, 5xx, partial, races)
- Platform failure modes (iOS background kill, Android process death, permissions, low memory)
- Concurrency (rapid taps, dispose-during-async, refresh races)
- Security (token storage, deep links, WebViews, log scrubbing)
- Performance budgets (startup time, app size, jank, memory)
- Accessibility (screen reader, contrast, tap targets, dynamic type)
- Localization correctness

**REVIEW_FINDINGS.md** is a living ledger:

| ID | Title | Severity | Owner | Status | Failing test | Round | Notes |
|---|---|---|---|---|---|---|---|

Severities: `Blocker` (blocks ship indefinitely), `Major` (must fix or waive), `Minor` (file but don't block).

**Findings closure: Reviewer-only.** Only `engineering-reviewer` flips `open` → `fixed`, after re-running the failing test it wrote and confirming it passes. Feature devs marking their own findings fixed is forbidden. `waived` requires explicit human sign-off in the Notes column.

**Loop budget:** 3 unresolved rounds → escalate to `engineering-planner` for scope cut or contract change. Reviewer owns the round counter.

---

## Human Gate 1 — Approve to Pre-Ship

**MANDATORY. NEVER SKIPPED. NO EXCEPTIONS.**

Pre-condition: `REVIEW_FINDINGS.md` shows zero open Blockers; Majors are `fixed` or `waived` with explicit human sign-off.

Human reviews findings, evals coverage, and DESIGN_SYSTEM updates before final-boss is engaged. No agent has authority to bypass this gate, even when the feature looks trivial.

---

## Phase 4 — Delivery

### final-boss

| | |
|---|---|
| **Inputs** | `ENGINEERING_PLAN.md`, `FRONTEND_NOTES.md`, `BACKEND_NOTES.md`, `REVIEW_FINDINGS.md` (clean), full test suite, build/signing config, last-shipped production build |
| **Outputs** | Signed IPA + AAB, `RELEASE_CHECKLIST.md`, `ROLLOUT_PLAN.md`, `ROLLBACK_PLAN.md`, `STORE_NOTES.md`, `CHANGELOG.md` updates |
| **Branch** | `release/<version>` |

**Final boss does NOT author tests, does NOT modify feature code, does NOT hunt for bugs.**

**Pre-ship testing (execute only, on release build):**
- Full unit suite passes on release configuration
- Integration + E2E pass on real iOS + Android devices
- Snapshot/golden tests pass on pinned device image
- Upgrade-path regression from last shipped version
- Eval suite passes vs baseline (or marked N/A)

**Release engineering (exclusive domain):**
- Build configs, flavors, signing, secrets
- Observability verification (force a test crash; confirm dashboard)
- dSYMs / R8 mapping / source maps uploaded
- Store metadata, privacy labels, data-safety form
- Staged rollout plan with halt criteria
- Rollback plan + kill-switch flag **executed and verified**

**Routing regressions (not fixes):**

| Symptom | Route to |
|---|---|
| Test passes on dev, fails on release | `frontend-developer` or `backend-developer` |
| Eval regression beyond threshold | `backend-developer`; `engineering-planner` if contract change |
| Budget regression (size, startup, memory, AI cost/latency) | `frontend-developer` or `backend-developer`; `engineering-reviewer` if no test exists |
| Missing test coverage discovered | `engineering-reviewer` (final-boss does **not** write it) |
| Store rejection (UI/copy) | `frontend-developer` |
| Store rejection (privacy/data) | `backend-developer` or `engineering-planner` |
| Store rejection (metadata) | `final-boss` (self) |

---

## Human Gate 2 — Production Promotion

**MANDATORY. NEVER SKIPPED. NO EXCEPTIONS.**

After TestFlight / Play internal track + staged rollout shows healthy metrics (crash-free rate, error rate, business guardrails per `ROLLOUT_PLAN.md`), the human approves promotion to full production. `final-boss` does not auto-promote — every rollout step is human-confirmed.

`final-boss` advances the rollout %, promotes TestFlight build to App Store production, and tags the release.

---

## Hotfix Mode — Post-Release Incidents

When a Blocker-severity issue surfaces in production after Gate 2 (crash spike, security flaw, data corruption, broken auth), the pipeline re-enters in **Hotfix Mode**. All five agents participate; **all three Human Gates still run**.

What's different in Hotfix Mode:
- **engineering-planner** produces a minimal plan: root cause + fix scope + regression test plan (not a full ENGINEERING_PLAN.md). Output: `docs/incidents/<id>/HOTFIX_PLAN.md`.
- **Frontend or Backend** ships the minimal fix on `hotfix/<id>` branch.
- **engineering-reviewer** runs the existing suite + authors a regression test that locks in the incident's failure mode. No broader adversarial pass — scope is the incident only.
- **final-boss** runs full pre-ship testing on the hotfix branch. May propose a compressed staged rollout (e.g., 10% → 100% in hours instead of days) for critical security fixes; the compression itself requires human approval at Gate 2.
- Post-incident: `docs/incidents/<id>/POSTMORTEM.md` is produced, and the regression test stays in the suite forever.

**Hotfix Mode never skips a Human Gate. It compresses scope, not safety.**

---

## Pipeline State & Cost

Four repo-level append-only files keep the pipeline auditable:

| File | Purpose | Writers |
|---|---|---|
| `PIPELINE-STATE.md` | Timeline of every agent run + gate decision. The "where are we?" file. | All agents on entry + completion; humans on gate decisions |
| `PIPELINE-COST.md` | Token / duration / cost ledger per agent run | All agents on completion (cost field filled by harness if available) |
| `CHANGELOG.md` | User-facing release notes | `final-boss` on Gate 2 only |
| `LESSONS-LEARNED.md` | Cross-feature institutional memory | `engineering-planner` reads at every planning run; `final-boss` appends after Gate 2 and after every incident postmortem |

**Append-only rule:** no agent ever rewrites or deletes prior entries in any of these files. Parallel agents (Frontend + Backend) appending in the same minute is safe by construction — each writes a single line.

Format for each file is defined in its own header. Reviewer's round counter lives in `REVIEW_FINDINGS.md` (per-feature), not in `PIPELINE-STATE.md`.

---

## Document Reference

| Document | Produced by | Consumed by |
|---|---|---|
| `PRD.md` | human | all agents |
| `docs/PRD_CHECKLIST.md` | human (editable team standard) | `/prd-check` command |
| `PRD.md` `## Clarifications & Updates` section | `/prd-check` command (append-only) | `engineering-planner`, human |
| `CLAUDE.md` | human | all agents |
| `docs/DESIGN_SYSTEM.md` | `frontend-developer` (maintained) | all agents read; Frontend writes |
| `docs/plans/<feature>/BRAINSTORM.md` | `engineering-planner` (Mode B) | human, `engineering-planner` (Mode A) |
| `docs/plans/<feature>/PROJECT.md` | `engineering-planner` (Mode A) | **primary context for** `frontend-developer`, `backend-developer`, `engineering-reviewer`, `final-boss` |
| `docs/plans/<feature>/ENGINEERING_PLAN.md` | `engineering-planner` (Mode A) | `frontend-developer`, `backend-developer`, `engineering-reviewer`, `final-boss` |
| Interface stubs + fake impls | `engineering-planner` | `frontend-developer`, `backend-developer` |
| `docs/plans/<feature>/FRONTEND_NOTES.md` | `frontend-developer` | `engineering-reviewer`, `final-boss` |
| `docs/plans/<feature>/BACKEND_NOTES.md` | `backend-developer` | `engineering-reviewer`, `final-boss` |
| `docs/plans/<feature>/REVIEW_FINDINGS.md` | `engineering-reviewer` | `frontend-developer`, `backend-developer`, `final-boss`, human |
| Feature tests (UI + presentation-state) | `frontend-developer` | `engineering-reviewer`, `final-boss` |
| Repo tests (unit + integration + migration) | `backend-developer` | `engineering-reviewer`, `final-boss` |
| Adversarial tests | `engineering-reviewer` | `final-boss`, feature devs (for fixes) |
| Migration scripts | `backend-developer` | `final-boss` (upgrade-path regression) |
| `docs/releases/<version>/RELEASE_CHECKLIST.md` | `final-boss` | human |
| `docs/releases/<version>/ROLLOUT_PLAN.md` | `final-boss` | human, on-call |
| `docs/releases/<version>/ROLLBACK_PLAN.md` | `final-boss` | human, on-call |
| `docs/releases/<version>/STORE_NOTES.md` | `final-boss` | store review, human |
| `CHANGELOG.md` | `final-boss` (append per release) | human, store |
| `PIPELINE-STATE.md` | all agents + humans (gates) | all agents on re-entry, humans for status |
| `PIPELINE-COST.md` | all agents | humans, finance/eng leadership |
| `LESSONS-LEARNED.md` | `engineering-planner` reads; `final-boss` appends | `engineering-planner` on every plan |
| `docs/incidents/<id>/HOTFIX_PLAN.md` | `engineering-planner` (Hotfix Mode) | all agents during incident |
| `docs/incidents/<id>/POSTMORTEM.md` | `final-boss` (Hotfix Mode close) | human, future Planner inputs |

---

## Routing Rules

### When to use brainstorming mode
- Direction not yet chosen
- Multiple viable approaches and the PRD doesn't pick one
- Human explicitly asks for options
- Otherwise: always default to Plan mode

### When to re-engage engineering-planner mid-flight
- Frontend or Backend finds the contract can't represent a real requirement
- Reviewer's findings expose missing error type, missing state, or missing field
- 3 unresolved Reviewer rounds without convergence
- Scope changes mid-feature

### When final-boss routes back
- Any test that fails on release but not on dev
- Any budget regression vs the Planner's spec
- Any missing test coverage discovered during pre-ship → route to Reviewer
- Any store rejection

---

## CI ownership

`final-boss` is the only agent that may modify CI / build-automation files (`.github/workflows/*.yml`, fastlane configs, build scripts). All other agents flag CI needs via a `## CI changes required` table in their handoff doc with status `PROPOSED`. `team-leader` invokes `final-boss` in **Feasibility Review mode** before the change is committed to the plan; verdict is `APPROVED`, `APPROVED-WITH-CONDITIONS`, or `REJECTED`. Implementation happens during Phase 4 (or inline if blocking). Initial CI templates are emitted by `/start-bootstrap` on Day 1.

Gates verify CI table cleanliness:
- **Gate 0** — any CI changes proposed by the Planner have been feasibility-reviewed
- **Gate 1** — zero `PROPOSED` rows remain across `FRONTEND_NOTES.md`, `BACKEND_NOTES.md`, `REVIEW_FINDINGS.md`

---

## Key Rules

1. **The Planner is the single source of truth for contracts.** Frontend and Backend never extend interfaces silently — re-engage the Planner.
2. **Fake/in-memory impls are a Planner deliverable.** Without them, Frontend/Backend parallelism is fiction.
3. **Frontend owns the design system.** `docs/DESIGN_SYSTEM.md` updates ship in the same commit as the component change.
4. **Reuse before you build.** A new shared primitive is added to the design system before being used in a feature.
5. **No agent skips RED.** A failing test is committed before implementation.
6. **Backend integration tests hit a real local server or emulator — never mocks.**
7. **Frontend never calls SDKs directly.** All networking/persistence goes through Planner interfaces.
8. **Engineering-reviewer owns all non-functional test authoring** — final-boss never writes tests.
9. **Engineering-reviewer never fixes bugs** — only encodes them as failing tests and routes.
10. **Engineering-reviewer has a 3-round loop budget per feature** before escalating to the Planner.
11. **Final-boss never modifies feature code or tests** — regressions route back to the responsible agent.
12. **Final-boss verifies on the release build, on real devices** — release-config bugs are real bugs.
13. **Kill switches are executed, not declared.** Final-boss must disable the new feature on a release build before submission.
14. **Every release has a documented rollback plan** before submission.
15. **Staged rollout when the platform supports it** — no direct-to-100% production pushes.
16. **No secrets in the repo.** Build-time injection only.
17. **Tokens, passwords, and PII never appear in logs, analytics, or crash reports.**
18. **Destructive migrations require explicit Planner sign-off.** Backend stops and surfaces, never silently ships data loss.
19. **All human gates are mandatory and NEVER skipped.** Gate 0 (Plan), Gate 1 (Pre-Ship), Gate 2 (Production) run on every feature, every release, regardless of size or familiarity. No agent has authority to bypass.
20. **Severity rubric is binding.** Minor findings never block ship; Blockers always do.
