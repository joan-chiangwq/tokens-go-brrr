# Repo Adequacy Checklist

Used by `/start-bootstrap` to audit whether the repo is ready to run the 5-agent pipeline. Humans should edit this checklist to reflect what "adequate repo" means for this team.

An item is **satisfied** when the repo passes it clearly, OR the team explicitly defers to a documented default.

An item is **blocking** when its absence would cause the pipeline to fail or skip a real concern. Blocking gaps halt the pipeline.

An item is **soft** when a reasonable default exists and the cost of being wrong is low. Soft gaps are surfaced with the bootstrap command's chosen assumption.

---

## 1. Stack
- [ ] **B** — Stack declared in `CLAUDE.md` `## Stack` section (language, framework, key libraries, versions)
- [ ] **B** — Project compiles / builds cleanly from a fresh checkout
- [ ] **S** — Versions pinned in a lockfile

## 2. Source control
- [ ] **B** — `main` branch exists and is protected (or branch protection documented)
- [ ] **B** — Branch model in `HOUSEKEEPING.md` matches reality
- [ ] **S** — `.gitignore` covers the stack's standard artifacts (`node_modules/`, `target/`, `build/`, `.DS_Store`, `.env.*`, etc.)

## 3. CI
- [ ] **B** — Build + unit-test CI runs on every PR to `main`
- [ ] **S** — Lint + format checks on PR
- [ ] **S** — Test coverage reported (or threshold documented as deferred)

## 4. Environments
- [ ] **B** — At least `dev` and `prod` environments declared (more if the product warrants)
- [ ] **B** — Secrets management mechanism defined (env vars, cloud secret manager, vault, etc.)
- [ ] **B** — `.env.example` committed with required variable names (no real values)

## 5. Test infrastructure
- [ ] **B** — Unit test runner installed and runnable locally with a single command
- [ ] **S** — Integration / E2E framework chosen (or documented as deferred)

## 6. Observability
- [ ] **B** — Crash / error reporter wired (Sentry, Crashlytics, Rollbar, etc.) — or "no observability" documented as an explicit decision
- [ ] **S** — Analytics SDK chosen (if the product needs it)
- [ ] **S** — Logger / log levels configured for production

## 7. Delivery
- [ ] **B** — A documented way to produce a deployable build artifact from a fresh checkout
- [ ] **B** — Deployment target declared in `CLAUDE.md` (TestFlight+Play / Vercel / AWS / npm / etc.)
- [ ] **S** — Staged rollout mechanism available (Play %s, LaunchDarkly, feature flags)
- [ ] **S** — Rollback procedure documented

## 8. Pipeline plumbing
- [ ] **B** — `PIPELINE-STATE.md`, `PIPELINE-COST.md`, `CHANGELOG.md`, `LESSONS-LEARNED.md` exist at repo root with their templates
- [ ] **B** — `WORKFLOW.md`, `HOUSEKEEPING.md`, `GLOBAL_AGENT_RULES.md`, `ERROR_HANDLING.md`, `GLOBAL_CODING_STANDARDS.md` exist at repo root
- [ ] **B** — `.claude/agents/` contains the 6 agent files (`team-leader` + 5 specialists)
- [ ] **B** — `.claude/commands/prd-check.md` and `.claude/commands/start-bootstrap.md` exist

---

**B** = blocking (halts pipeline if unanswered)
**S** = soft (`/start-bootstrap` picks a default, logs the assumption)
