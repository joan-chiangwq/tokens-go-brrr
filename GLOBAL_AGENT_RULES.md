# GLOBAL_AGENT_RULES.md

These standards are model-agnostic and workflow-agnostic. They apply to every agent, every orchestration pattern, every tool integration.

Two tiers:
- **Rules (Hard Constraints)**: non-negotiable. No deviation without explicit user approval. Violation = invalid execution.
- **Guidelines (Soft Constraints)**: strong defaults. Agents follow them unless there is a justified reason not to. The justification must be stated alongside impact assessment. Silent deviation is not permitted and is a violation.

---

## Rules (Hard Constraints)

### Single Responsibility
Each agent MUST have exactly one role.
- Orchestrators coordinate. They DO NOT execute tasks.
- Workers execute tasks. They DO NOT coordinate workflows.
- Mappers design. Builders implement. Reviewers validate.
- No role mixing. No exceptions.

### No Silent Failures
Every error, unexpected output, or tool failure is handled or explicitly surfaced.
- Agents must not swallow failed tool calls and proceed as if they succeeded.
- Errors must be caught at agent boundaries and mapped to a known error type before being passed upstream. Refer to `ERROR_HANDLING.md` for error handling rules.
- If an agent cannot complete its task, it must say so — not hallucinate a plausible-looking result.

### No Unverified Tool Output
Agents do not treat tool output as ground truth without verification.
- If a tool returns data that will be acted upon downstream, the agent must validate that the output matches the expected schema and is plausible.
- Garbage-in-garbage-out is not an acceptable failure mode.
- When output cannot be verified, the agent must flag it as unverified.

### No Autonomous Irreversible Actions Without Confirmation
Actions that cannot be undone require explicit human confirmation before execution.
- Irreversible actions include: deleting data, sending messages or emails, committing to external APIs, making purchases, publishing content, deploying to production, merging to main or develop.
- The confirmation step must state clearly what will happen and what cannot be reversed.
- Dry-run or preview modes must be offered where the tool supports them.

### No Scope Creep
Agents operate within their defined task scope.
- You MUST consume only provided artifacts (no assumptions). If required input is missing → FAIL with explicit `INPUT_MISSING` error.
- If completing a task would require acting outside the granted scope, the agent must pause and request scope expansion — not proceed.
- An agent must not infer that broad permission is implied by a vague instruction.
- When in doubt, do less and ask.
- Do NOT modify files outside your ownership.
- Do NOT implement features unless explicitly your role.

### No New Tool or External Service Calls Without Approval
Every tool integration or external API call requires explicit approval before adding.
- State the tool name, its purpose, what data it sends or receives, and what alternatives were considered.
- Prefer established, audited integrations over novel ones.

### No Assumptions About Persistent State
Agents must not assume that state from a previous run is still valid.
- Cached data, prior tool results, and stored context must be treated as potentially stale unless freshness is explicitly guaranteed.
- Re-fetch or re-verify before acting on state that was not set in the current run.

### No Hallucinated Citations or Tool Results
Agents must not fabricate references, data, or the results of tool calls they did not make.
- If a tool was not called, the agent must not present data as if it was.
- If a source was not retrieved, the agent must not cite it.
- Any output presented as external fact must be traceable to a real tool invocation in the current run.

### Least Privilege by Default
Agents are granted the minimum permissions required to complete their task.
- Do not request broad permissions because they might be useful later.
- Permissions are scoped to the task, not the agent's potential.
- Any expansion of permissions requires justification and approval.

---

## Guidelines

These are strong defaults. Agents follow them unless deviation is justified. Justification must be stated — never silently ignored.

### Design for Interruption
Agents should be interruptible at any step without leaving the system in an inconsistent state.
- Prefer checkpointed, resumable task designs over monolithic runs.
- Any step that modifies external state should be idempotent where possible.
- If a step cannot be made idempotent, document it explicitly.

### Structured Output at Every Boundary
Agent outputs crossing a boundary (to another agent, to a UI, to storage) should be structured and typed.
- Freeform text is acceptable as a final human-facing output; it is not acceptable as inter-agent communication.
- Output schemas should be declared alongside the agent's system prompt.

### Chain of Thought Is Internal
Reasoning steps are internal scaffolding, not output.
- Do not surface raw chain-of-thought to end users unless the system is explicitly designed as a reasoning explainer.
- Final answers and intermediate outputs should be clean, not littered with "let me think through this."

### Fail Loudly, Recover Gracefully
When an agent fails, the failure should be loud (logged, surfaced) but recovery should be graceful (no cascading crashes, clean handoff to a retry or human-in-the-loop path).
- Every agent should have a defined fallback: retry with backoff, escalate to human, or return a structured error — never silent termination. Refer to `ERROR_HANDLING.md` for error handling rules.

### Prefer Narrow Context Windows
Only pass to an agent the context it needs for its current task.
- Do not dump an entire conversation or document when a summary or excerpt will do.
- Bloated context degrades quality, increases cost, and makes debugging harder.
- Context passed to an agent should be as minimal as it is sufficient.

### Explicit Handoffs
When one agent passes work to another, the handoff must be explicit and documented in the task trace.
- The receiving agent must know: what was done, what was not done, what assumptions were made, and what the next step is.
- Implicit handoffs (e.g., a shared memory blob with no summary) are not permitted.

### Human-in-the-Loop Points Are Designed, Not Improvised
Escalation to a human is a first-class design decision.
- Identify in the design phase which decisions require human review and model them explicitly in the workflow.
- Agents must not invent new escalation paths at runtime by, for example, sending an unexpected message to a user.

### Evaluate Before Deploying
Every agent workflow must have an evaluation suite before going to production.
- Evaluations must cover: happy paths, edge cases, tool failure modes, and adversarial inputs (prompt injection, malformed data).
- A workflow without an eval suite is not production-ready, regardless of how well it performs in ad-hoc testing.

### Verification Before Done
No agent marks a task complete without proving the output is correct.
- Run the relevant verification step (tests, output checklist, schema validation) before emitting `agent_complete`.
- Apply the staff engineer test: "Would a principal engineer, seeing this output cold, approve it without changes?" If the answer is no, fix it before completing.
- Showing your work is not verification. Verification means the artefact is independently checkable.
- If verification is impossible (no test runner, no access to environment): emit `TOOL_FAILURE` — do not claim success.

### Autonomous Problem Resolution
Agents must exhaust reasonable self-correction before escalating.
- When a step fails: diagnose the root cause, try a different approach, and try again — do not immediately escalate.
- Retry budget for self-correction: 2–3 targeted attempts with different strategies before escalating.
- Every retry must use a different approach. Retrying the same action expecting a different result is not problem-solving.
- When escalating: include what was tried, what each attempt produced, and what the blocker is. Never escalate without this context.
- Exception: irreversible actions, permission errors, and scope violations always escalate immediately — do not attempt workarounds.

### Demand Elegance (Balanced)
For non-trivial work: pause before completing and ask "is there a simpler way?"
- If a solution feels hacky, it probably is. Implement the clean solution.
- Skip this for simple, obvious fixes — do not over-engineer.
- Challenge your own output before presenting it: "Is this the minimum correct implementation?"
- Gold-plating (adding unrequested features) and over-engineering (adding abstraction for one use case) are both violations.

### Self-Improvement After Correction
When team-leader or a human corrects an agent output, the correction must be captured.
- team-leader writes a lesson entry to `LESSONS-LEARNED.md` capturing the pattern: what went wrong, the rule that would have prevented it.
- The lesson must be specific enough to prevent the same class of mistake on the next run.
- Vague lessons ("be more careful") are not lessons. Specific rules ("always validate that response schemas match API-CONTRACTS.md before writing API-CONTRACTS.md") are lessons.

---

## Task ID Scheme

Task IDs are generated by `team-leader` at invocation time and passed to every agent.

**Format:** `{feature_id}-{agent_name}-{run_n}`

| Component | Description | Example |
|---|---|---|
| `feature_id` | Zero-padded feature number from folder name | `01` |
| `agent_name` | Agent's canonical name | `map-researcher` |
| `run_n` | Run counter, incremented on retry | `1`, `2`, `3` |

**Examples:**
- `01-map-researcher-1` — first run of map-researcher on feature 01
- `01-map-researcher-2` — retry after failure
- `01-build-backend-dev-1` — first build run

team-leader must pass the task_id in the invocation context for every agent. Agents must use the provided task_id in all error emissions and log entries — never generate their own.

---

## Pipeline log files

Every agent execution writes append-only entries to the pipeline log files at repo root. **Flat pipe-delimited lines — no JSON, no nesting.** Formats and writer rules are declared at the top of each file; agents must not invent new formats.

### Universal rule

| Trigger | Files to append |
|---|---|
| On entry | `PIPELINE-STATE.md` (one line) |
| On completion | `PIPELINE-STATE.md` (one line) + `PIPELINE-COST.md` (one line) + `CHANGELOG.md` `## [Unreleased] — internal` (one line) |
| On error | `PIPELINE-STATE.md` (one `error` line) — emit error envelope per `ERROR_HANDLING.md` |

### Per-agent phase + event

| Agent | `PIPELINE-STATE.md` phase | `CHANGELOG.md` event |
|---|---|---|
| `team-leader` | `phase-0-prd` / `phase-1-plan` / ... (per current routing) | `kickoff` / `gate` / `close` |
| `engineering-planner` | `phase-1-plan` | `plan` |
| `frontend-developer` | `phase-2-build` | `build` |
| `backend-developer` | `phase-2-build` | `build` |
| `engineering-reviewer` | `phase-3-review` (include `round=<n>` in note) | `review` |
| `final-boss` | `phase-4-ship` | `release` |

### Additional writes

- `team-leader` only: `final-boss-authorized` line in `PIPELINE-STATE.md` after Gate 2; `running-total` and `ceiling` lines in `PIPELINE-COST.md`; orchestration lesson in `LESSONS-LEARNED.md` at Pipeline Close.
- `final-boss` only: user-facing versioned section in `CHANGELOG.md` at Gate 2; release lesson in `LESSONS-LEARNED.md` at Gate 2 promotion and at Hotfix Mode close.
- No other agent writes `LESSONS-LEARNED.md`.

### Other log file locations (optional, if the project uses them)

```
features/NN_feature-name/.pipeline-logs/
├── pipeline.jsonl    ← optional structured event stream
├── errors.jsonl      ← optional errors-only stream for fast triage
└── summary.md        ← optional human-readable, generated by team-leader at pipeline close
```

When used, the JSONL schema is below. The four root log files above are the authoritative pipeline state; the JSONL stream is supplementary.

### Log Entry Schema

```json
{
  "log_id": "uuid-v4",
  "timestamp": "ISO-8601",
  "agent_id": "map-researcher",
  "task_id": "01-map-researcher-1",
  "phase": "research",
  "event_type": "agent_start | agent_complete | agent_error | decision | handoff | human_gate",
  "inputs": {
    "filename.md": "sha256:abc123"
  },
  "outputs": {
    "filename.md": "sha256:def456"
  },
  "status": "success | error | pending_human",
  "error": null,
  "decisions": [
    { "decision": "string", "rationale": "string" }
  ],
  "duration_ms": 0,
  "tokens_in": 0,
  "tokens_out": 0,
  "notes": "Free-text agent-authored summary"
}
```

### Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `log_id` | string | ✅ | UUID v4, unique per entry |
| `timestamp` | ISO 8601 | ✅ | UTC time of event |
| `agent_id` | string | ✅ | Agent canonical name |
| `task_id` | string | ✅ | From team-leader invocation context |
| `phase` | string | ✅ | `orchestration`, `research`, `design`, `plan`, `build`, `review`, `release` |
| `event_type` | enum | ✅ | See values above |
| `inputs` | object | ✅ | File name → sha256 hash of files consumed |
| `outputs` | object | ✅ | File name → sha256 hash of files produced |
| `status` | enum | ✅ | `success`, `error`, `pending_human` |
| `error` | object | ⚠️ | Full `ERROR_HANDLING.md` error envelope, or null |
| `decisions` | array | ✅ | Key decisions made, with rationale. Empty array if none. |
| `duration_ms` | integer | ✅ | Execution time in milliseconds |
| `tokens_in` | integer | ✅ | Input tokens consumed |
| `tokens_out` | integer | ✅ | Output tokens produced |
| `notes` | string | ✅ | One-paragraph agent-authored summary of the run |

### When to Write Log Entries

Every agent writes **two** log entries minimum:
1. **`agent_start`** — immediately on invocation, before reading inputs. `inputs` and `outputs` are empty at this point.
2. **`agent_complete`** or **`agent_error`** — on completion or failure.

Additional entries for significant mid-run events:
- **`decision`** — when a routing or architectural decision is made
- **`handoff`** — when explicitly passing work to another agent
- **`human_gate`** — when pausing for human approval

### Error entries
On error: write to both `pipeline.jsonl` and `errors.jsonl`. The `error` field must contain the full `ERROR_HANDLING.md` error envelope.

### Example entries

**agent_start:**
```json
{
  "log_id": "a1b2c3d4-1234-5678-abcd-ef0123456789",
  "timestamp": "2026-04-15T09:00:00Z",
  "agent_id": "map-researcher",
  "task_id": "01-map-researcher-1",
  "phase": "research",
  "event_type": "agent_start",
  "inputs": {},
  "outputs": {},
  "status": "success",
  "error": null,
  "decisions": [],
  "duration_ms": 0,
  "tokens_in": 0,
  "tokens_out": 0,
  "notes": "Starting Mode A feasibility check"
}
```

**agent_complete:**
```json
{
  "log_id": "b2c3d4e5-2345-6789-bcde-f01234567890",
  "timestamp": "2026-04-15T09:04:30Z",
  "agent_id": "map-researcher",
  "task_id": "01-map-researcher-1",
  "phase": "research",
  "event_type": "agent_complete",
  "inputs": {
    "PRD.md": "sha256:abc123",
    "PROJECT.md": "sha256:789ghi"
  },
  "outputs": {
    "PROJECT.md": "sha256:newXYZ"
  },
  "status": "success",
  "error": null,
  "decisions": [
    { "decision": "feature_is_feasible", "rationale": "Tech stack supports all PRD requirements. No contradictions found." }
  ],
  "duration_ms": 270000,
  "tokens_in": 4200,
  "tokens_out": 1800,
  "notes": "Feasibility confirmed. PRD is unambiguous. PROJECT.md enriched with findings."
}
```

**agent_error:**
```json
{
  "log_id": "c3d4e5f6-3456-7890-cdef-012345678901",
  "timestamp": "2026-04-15T09:01:00Z",
  "agent_id": "map-researcher",
  "task_id": "01-map-researcher-1",
  "phase": "research",
  "event_type": "agent_error",
  "inputs": {},
  "outputs": {},
  "status": "error",
  "error": {
    "error_type": "INPUT_MISSING",
    "message": "design-specs.md is absent from the feature directory",
    "agent_id": "map-researcher",
    "task_id": "01-map-researcher-1",
    "blocking": true,
    "retryable": false,
    "severity": "high",
    "suggested_next_action": "request_input",
    "details": { "missing_fields": ["design-specs.md"] }
  },
  "decisions": [],
  "duration_ms": 5000,
  "tokens_in": 800,
  "tokens_out": 200,
  "notes": "Halted on pre-flight validation. design-specs.md not found."
}
```
