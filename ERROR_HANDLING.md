# ERROR_HANDLING.md

This document is the authoritative contract for error handling across all agents and orchestrators in the system. It defines:

- The canonical set of error types
- Precise conditions under which each error type must be used
- Required agent behavior upon encountering each error
- Orchestrator (team-leader) response logic and follow-up actions

All agents must comply with this specification. Deviation is not permitted.

## Error Schema

Every error emitted by an agent must conform to the following structure:

```json
{
  "status": "error",
  "error": {
    "error_type": "INPUT_MISSING",
    "message": "Human-readable description of the error",
    "agent_id": "agent_name",
    "task_id": "01-map-researcher-1",
    "blocking": true,
    "retryable": false,
    "severity": "medium",
    "suggested_next_action": "request_input",
    "details": {}
  }
}
```

### Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `error_type` | string | ✅ | One of the canonical types defined below |
| `message` | string | ✅ | Clear, human-readable description of what failed |
| `agent_id` | string | ✅ | Identifier of the agent that emitted the error |
| `task_id` | string | ✅ | Format: `{feature_id}-{agent_name}-{run_n}` — provided by team-leader at invocation |
| `blocking` | boolean | ✅ | Whether this error halts pipeline execution |
| `retryable` | boolean | ✅ | Whether the orchestrator may retry this operation |
| `severity` | string | ✅ | One of: `low`, `medium`, `high`, `critical` |
| `suggested_next_action` | string | ✅ | One of the canonical actions defined below |
| `details` | object | ⚠️ | Error-specific payload — structure varies by type |

---

## Error Types and Conditions

### `INPUT_MISSING`

**When to use:** A required input field is absent, the input schema validation fails due to missing data, or an upstream dependency produced incomplete output.

**Agent behavior:**
- Stop execution immediately
- Do not infer, default, or guess missing values

**Orchestrator action:**
- Request the missing input from the upstream agent or user
- Retry the agent once input has been provided

**Details payload:**
```json
{ "missing_fields": ["PRD.md", "design-specs.md"] }
```

---

### `CLARIFICATION_NEEDED`

**When to use:** Inputs exist but are too ambiguous, contradictory, or incomplete to act on safely. Used when the agent cannot proceed without a human decision. This is distinct from `INPUT_MISSING` — the document is present but its content is insufficient.

**Agent behavior:**
- Stop execution immediately
- Produce `CLARIFICATION-NEEDED.md` with a numbered list of specific, targeted questions
- Do not guess or infer answers
- Do not produce partial outputs

**Orchestrator action:**
- Present `CLARIFICATION-NEEDED.md` to the human
- Await human response
- Update inputs per human response
- Re-invoke the agent with updated inputs

**Details payload:**
```json
{
  "clarification_file": "features/01_feature-name/docs/CLARIFICATION-NEEDED.md",
  "question_count": 3,
  "questions_summary": ["Auth method not specified", "Target platform ambiguous", "Pricing tier unclear"]
}
```

**Note:** `retryable: true`. Questions in `CLARIFICATION-NEEDED.md` must be specific and actionable — not general complaints about the spec.

---

### `OUT_OF_SCOPE`

**When to use:** The task exceeds the agent's defined area of responsibility, or the action requires permissions or roles that have not been granted to this agent.

**Agent behavior:**
- Stop execution immediately
- Do not attempt partial completion or workarounds

**Orchestrator action:**
- Re-route to the correct agent, or escalate to a human operator
- If scope expansion is needed, it requires explicit approval before proceeding

**Details payload:**
```json
{ "reason": "Task requires write access to billing records, which is outside this agent's scope" }
```

---

### `TOOL_FAILURE`

**When to use:** An external tool, API, or service call fails — including timeouts, error status codes, and network failures.

**Agent behavior:**
- Capture all available error details from the tool response
- Mark as `retryable: true` if the failure is transient (e.g., timeout, 503)
- Mark as `retryable: false` if the failure is definitive (e.g., 404, invalid credentials, test runner not installed)

**Orchestrator action:**
- If retryable: retry with exponential backoff, up to the maximum attempts defined in the Retry Policy
- If still failing after max attempts, or if not retryable: escalate

**Details payload:**
```json
{
  "tool_name": "test_runner",
  "status_code": null,
  "tool_error_message": "jest: command not found"
}
```

---

### `VALIDATION_FAILED`

**When to use:** Data received by the agent fails format validation, does not match the expected schema, or contains logical inconsistencies (e.g., a negative price, an end date before a start date, tests passing on a blank codebase).

**Agent behavior:**
- Stop execution immediately
- Include all validation errors in the `details` payload

**Orchestrator action:**
- Abort the pipeline
- Route the error details for debugging or human correction

**Details payload:**
```json
{
  "validation_errors": [
    { "field": "test_result", "issue": "Tests passed on blank codebase — tests are not testing requirements", "received": "all_pass" }
  ]
}
```

---

### `UNVERIFIED_OUTPUT`

**When to use:** A tool returns output that passes schema validation but cannot be trusted — for example, the result is implausible, a required verification mechanism is absent, or the data source is unreliable.

**Agent behavior:**
- Flag the output as unverified
- Do not pass the output downstream without verification

**Orchestrator action:**
- Route to a verification agent or trigger a dedicated validation step before continuing

**Details payload:**
```json
{
  "reason": "Web search returned results from an unverifiable source",
  "unverified_field": "competitor_pricing_data"
}
```

---

### `PERMISSION_DENIED`

**When to use:** The agent lacks the credentials, API key, role, or access rights required to perform the requested action.

**Agent behavior:**
- Stop execution immediately
- Do not attempt the action

**Orchestrator action:**
- Request that the appropriate permission be granted
- Escalate to a human operator if permissions cannot be resolved programmatically

**Details payload:**
```json
{
  "required_permission": "billing:write",
  "resource": "invoice_service"
}
```

---

### `NON_IDEMPOTENT_ACTION`

**When to use:** The action cannot be safely retried because it has irreversible side effects. Examples include deploying to production, merging to main, sending emails, charging a payment method, or permanently deleting records.

**Agent behavior:**
- Do not execute the action
- Hold and await explicit confirmation

**Orchestrator action:**
- Request explicit human approval before proceeding
- Provide a preview of the intended action if possible

**Details payload:**
```json
{
  "action": "merge_to_main",
  "preview": "Will merge feature/01_auth → main and trigger CD deploy to production",
  "rollback_path": "See ROLLBACK-PLAN.md"
}
```

---

### `UNKNOWN_ERROR`

**When to use:** The error does not match any of the above categories, or the failure is an unexpected runtime exception.

**Agent behavior:**
- Capture as much diagnostic context as possible (stack trace, state snapshot, inputs)
- Do not attempt recovery

**Orchestrator action:**
- Escalate to a human operator immediately
- Log the full error context for post-incident review

**Details payload:**
```json
{
  "exception": "NullPointerException at line 214",
  "context": "Processing order_id 9981 during tax calculation step"
}
```

---

## Orchestrator Decision Table

| Error Type | Retry | Action | Notes |
|---|---|---|---|
| `INPUT_MISSING` | false | `request_input` | Blocked until input is provided |
| `CLARIFICATION_NEEDED` | true | `request_input` | Present CLARIFICATION-NEEDED.md to human; re-invoke after response |
| `OUT_OF_SCOPE` | false | `escalate` / re-route | Requires agent reassignment or scope approval |
| `TOOL_FAILURE` | conditional | `retry_with_backoff` | Retryable flag must be `true`; max attempts enforced |
| `VALIDATION_FAILED` | false | `abort` | Data is invalid; correct before retrying |
| `UNVERIFIED_OUTPUT` | dependency | `verify_output` | Route to verification step before continuing |
| `PERMISSION_DENIED` | false | `request_permission` | Blocked until access is granted |
| `NON_IDEMPOTENT_ACTION` | false | `request_confirmation` | Requires explicit human approval |
| `UNKNOWN_ERROR` | dependency | `escalate_to_human` | Investigate before any retry |

---

## Retry Policy

```yaml
retry_policy:
  max_attempts: 3
  strategy: exponential_backoff
  initial_delay_ms: 500
  max_delay_ms: 5000
```

The orchestrator must not retry any error where `retryable: false`. Exceeding `max_attempts` without resolution must trigger `escalate_to_human`. team-leader must increment the run counter in the task_id on each retry (e.g., `01-map-researcher-2`).

**Triage loop budget:** team-leader must not retry the same agent for the same failure more than 3 times total. On the third failure, escalate to human with full failure history across all attempts.

---

## Loop Prevention

Review and QA cycles that route fixes back through build agents and re-invoke review/QA are bounded:

| Loop | Max rounds | On exhaustion |
|------|-----------|---------------|
| review-code-reviewer → build fix → re-review | 3 | Escalate to human with full CODE-REVIEW.md history from all rounds |
| review-qa-engineer → build fix → re-QA | 3 | Escalate to human with full QA-REPORT.md history from all rounds |

A "round" is one complete cycle: reviewer/QA produces findings → team-leader routes fix → build agent fixes → reviewer/QA re-invokes. The initial review counts as round 1.

team-leader tracks round counts in PIPELINE-STATE.md. Exceeding the limit is a hard stop — pipeline pauses for human intervention.

**Cascading error budget:** If a single agent invocation produces 3 different error types across retries (e.g., TOOL_FAILURE → VALIDATION_FAILED → UNKNOWN_ERROR), treat the third as budget-exhausted regardless of per-type counts. Escalate to human.

---

## Pipeline Cost Ceiling

team-leader defines a cost ceiling at kickoff in PIPELINE-COST.md:

| Complexity | Default ceiling (USD) |
|-----------|----------------------|
| Simple | $5.00 |
| Complex | $20.00 |

team-leader checks the running total before invoking each agent. If the next invocation would likely exceed the ceiling, pause and present the cost summary to the human: continue (with new ceiling) or stop. The ceiling is a guideline — the human decides. But the pipeline must never silently exceed it.

---

## Canonical Next Actions

| Action | Description |
|---|---|
| `retry` | Retry the operation immediately |
| `retry_with_backoff` | Retry using the exponential backoff policy |
| `request_input` | Obtain missing input from upstream or user |
| `escalate_to_human` | Hand off to a human operator |
| `request_permission` | Request the required credentials or access rights |
| `request_confirmation` | Await explicit approval before executing |
| `abort` | Terminate the pipeline; do not continue |
| `verify_output` | Route output to a verification agent before proceeding |
| `use_fallback` | Switch to the designated fallback mechanism |

---

## Severity Levels

| Severity | Meaning |
|---|---|
| `low` | Non-blocking issue; pipeline can continue with reduced functionality |
| `medium` | Blocking for this task but does not threaten the broader system |
| `high` | Significant failure; may affect multiple downstream tasks |
| `critical` | System-level failure; requires immediate human attention |

---

## Anti-Patterns

| Anti-Pattern | Why It Is Forbidden |
|---|---|
| Returning partial success with hidden errors | Downstream agents will operate on corrupt or incomplete data |
| Guessing or inferring missing inputs | Silent assumptions cause unpredictable and hard-to-debug failures |
| Retrying non-retryable failures | Wastes resources and may amplify side effects |
| Ignoring validation failures | Allows bad data to propagate through the pipeline |
| Executing irreversible actions without confirmation | Cannot be undone; may cause real-world harm |
| Inventing custom error types | Breaks orchestrator routing logic |
| Emitting more than one error per blocking failure | Creates ambiguity in orchestrator handling |

---

## Canonical Examples

### `INPUT_MISSING`
```json
{
  "status": "error",
  "error": {
    "error_type": "INPUT_MISSING",
    "message": "Required file 'design-specs.md' is absent from the feature directory",
    "agent_id": "map-researcher",
    "task_id": "01-map-researcher-1",
    "blocking": true,
    "retryable": false,
    "severity": "high",
    "suggested_next_action": "request_input",
    "details": {
      "missing_fields": ["design-specs.md"]
    }
  }
}
```

### `CLARIFICATION_NEEDED`
```json
{
  "status": "error",
  "error": {
    "error_type": "CLARIFICATION_NEEDED",
    "message": "PRD.md does not specify authentication method — cannot assess feasibility without this",
    "agent_id": "map-researcher",
    "task_id": "01-map-researcher-1",
    "blocking": true,
    "retryable": true,
    "severity": "high",
    "suggested_next_action": "request_input",
    "details": {
      "clarification_file": "features/01_feature-name/docs/CLARIFICATION-NEEDED.md",
      "question_count": 2,
      "questions_summary": ["Auth method not specified (OAuth2, JWT, session?)", "Target deployment environment not specified"]
    }
  }
}
```

### `TOOL_FAILURE`
```json
{
  "status": "error",
  "error": {
    "error_type": "TOOL_FAILURE",
    "message": "Test runner not found — cannot validate tests fail on blank codebase",
    "agent_id": "build-test-writer",
    "task_id": "01-build-test-writer-1",
    "blocking": true,
    "retryable": false,
    "severity": "high",
    "suggested_next_action": "escalate_to_human",
    "details": {
      "tool_name": "jest",
      "tool_error_message": "jest: command not found"
    }
  }
}
```

### `NON_IDEMPOTENT_ACTION`
```json
{
  "status": "error",
  "error": {
    "error_type": "NON_IDEMPOTENT_ACTION",
    "message": "About to merge feature/01_auth → develop and trigger staging deployment",
    "agent_id": "review-release-manager",
    "task_id": "01-review-release-manager-1",
    "blocking": true,
    "retryable": false,
    "severity": "high",
    "suggested_next_action": "request_confirmation",
    "details": {
      "action": "merge_to_develop",
      "preview": "Merge feature/01_auth → develop. CD will auto-deploy to staging. Tag: v1.2.0-staging",
      "rollback_path": "See ROLLBACK-PLAN.md"
    }
  }
}
```
