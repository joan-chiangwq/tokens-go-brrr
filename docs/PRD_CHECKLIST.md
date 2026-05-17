# PRD Adequacy Checklist

Used by `/prd-check` to audit a PRD before the `engineering-planner` produces an `ENGINEERING_PLAN.md`. Humans should edit this checklist to reflect what "adequate PRD" means for this team.

An item is **satisfied** when the PRD answers it clearly, OR when the PRD explicitly defers to a team-wide default (e.g., "supports the team's minimum OS matrix").

An item is **blocking** when its outcome would materially change the architecture, contracts, or test surface. Blocking gaps halt the pipeline with `CLARIFICATION_NEEDED.md`.

An item is **soft** when a reasonable default exists and the cost of being wrong is low. Soft gaps are surfaced with the Planner's chosen assumption — not blocking.

---

## 1. User & Purpose
- [ ] **B** — Target user / persona named
- [ ] **B** — Problem this feature solves stated
- [ ] **S** — Success metric defined (DAU, conversion, time-to-task, etc.)

## 2. Functional Scope
- [ ] **B** — Primary user journey described step by step
- [ ] **B** — Inputs and outputs at each step
- [ ] **B** — What is explicitly **out** of scope
- [ ] **S** — Notable edge cases the team cares about (or "default mobile checklist applies")

## 3. Data & State
- [ ] **B** — What persists across sessions
- [ ] **B** — What's local vs server-owned
- [ ] **B** — Multi-device sync expectations (sync / last-write-wins / device-local only)
- [ ] **B** — What happens on logout / account deletion (data retention)

## 4. Platforms & Constraints
- [ ] **B** — iOS / Android / both
- [ ] **S** — Minimum supported OS versions (or "team defaults")
- [ ] **B** — Offline behavior expectation (must-work-offline / degraded-offline / online-only)
- [ ] **S** — Localization scope (or "follow team's locale set")

## 5. External Dependencies
- [ ] **B** — Third-party APIs or services named
- [ ] **B** — Auth requirements (login required / anonymous OK / biometric)
- [ ] **B** — Existing features this depends on or interacts with
- [ ] **B** — **Server scope**: who owns the server side (this team's repo / another team / hosted SaaS / no server)

## 6. Non-Functional
- [ ] **S** — Performance hints (load time, animation expectation)
- [ ] **S** — Accessibility requirements (or "team defaults")
- [ ] **B** — Privacy / data-handling concerns (PII, payment, health, etc.)
- [ ] **B** — Security concerns explicitly flagged (deep links, WebView, file uploads)

## 7. Release & Rollout
- [ ] **S** — Target launch date (or "no constraint")
- [ ] **S** — Staged rollout expectations
- [ ] **B** — Feature-flag controlled? Kill-switch flag name suggested?

## 8. Author Hygiene
- [ ] **S** — Author has separated decided items from open questions
- [ ] **B** — No contradictions within the PRD

---

**B** = blocking (halts the pipeline if unanswered)
**S** = soft (Planner picks a default, logs the assumption)
