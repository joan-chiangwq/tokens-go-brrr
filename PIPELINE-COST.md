# Pipeline Cost

Append-only ledger of agent invocations. **Flat pipe-delimited lines — no JSON, no nesting.** One line per agent run on completion. Fields the agent cannot fill use `-`.

## Who writes here

- **team-leader** writes the `ceiling` line at kickoff and a `running-total` line after each agent completes.
- **Every agent** appends one run line on completion with its own model, task ID, duration, tokens, and cost.
- No other writers. No agent rewrites prior entries.

## Cost ceiling

First line of every pipeline run is the ceiling declaration. Defaults from `ERROR_HANDLING.md`: simple `$5`, complex `$20`. team-leader pauses before any agent invocation that would exceed it.

```
YYYY-MM-DD HH:MM | team-leader | ceiling | <feature> | usd:<n>
```

## Run format

```
YYYY-MM-DD HH:MM | <agent> | <model> | <feature> | task:<id> | duration_min:<n> | tokens_in:<n> | tokens_out:<n> | cost_usd:<x.xx> | notes
```

Append-only. No agent rewrites prior entries.

## Running total

team-leader appends a `running-total` line after each agent completes:

```
YYYY-MM-DD HH:MM | team-leader | running-total | <feature> | cost_usd:<x.xx> | ceiling_usd:<n>
```

## Example

```
2026-05-17 09:00 | team-leader          | -                | ceiling      | 01_auth-flow | usd:20
2026-05-17 09:18 | engineering-planner  | claude-opus-4-7  | 01_auth-flow | task:01-engineering-planner-1  | duration_min:15 | tokens_in:6200  | tokens_out:2400 | cost_usd:0.21 | -
2026-05-17 09:19 | team-leader          | running-total    | 01_auth-flow | cost_usd:0.21 | ceiling_usd:20
2026-05-17 10:40 | frontend-developer   | claude-sonnet-4-6| 01_auth-flow | task:01-frontend-developer-1   | duration_min:62 | tokens_in:18400 | tokens_out:7100 | cost_usd:0.34 | -
2026-05-17 11:55 | engineering-reviewer | claude-opus-4-7  | 01_auth-flow | task:01-engineering-reviewer-2 | duration_min:18 | tokens_in:8100  | tokens_out:1900 | cost_usd:0.18 | round=2
2026-05-17 11:55 | team-leader          | running-total    | 01_auth-flow | cost_usd:0.73 | ceiling_usd:20
```

## Log

<!-- entries appended below this line -->
