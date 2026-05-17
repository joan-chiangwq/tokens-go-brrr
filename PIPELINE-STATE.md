# Pipeline State

Append-only timeline of agent runs, gate decisions, and routing events. **Flat pipe-delimited lines — no JSON, no nesting.** One line per event.

## Who writes here

- **Every agent** appends its own `started` and `complete`/`error` lines.
- **team-leader** appends routing decisions, gate events, retries, escalations, and the `final-boss-authorized` line.
- **human** appends gate-approval / gate-rejection / gate-modified lines.
- No other writers. No agent rewrites prior entries.

## Format

```
YYYY-MM-DD HH:MM | <actor> | <phase> | <event> | <feature> | <note>
```

- **actor** — `team-leader` | `engineering-planner` | `frontend-developer` | `backend-developer` | `engineering-reviewer` | `final-boss` | `human`
- **phase** — `phase-0-prd` | `phase-1-plan` | `phase-2-build` | `phase-3-review` | `phase-4-ship` | `hotfix` | `close`
- **event** — `started` | `complete` | `blocked` | `error` | `retry` | `gate-surfaced` | `gate-approved` | `gate-rejected` | `gate-modified` | `final-boss-authorized` | `escalation`
- **feature** — short feature slug (e.g. `01_auth-flow`) or incident ID
- **note** — one short clause: artifact path, `task=<id>`, `round=<n>`, reason. No nested structures. Use `-` if nothing to add.
- Times are UTC. Append-only. **No agent rewrites or deletes prior entries.**

## Example

```
2026-05-17 09:00 | team-leader          | phase-0-prd     | started               | 01_auth-flow | task=01-team-leader-1
2026-05-17 09:02 | team-leader          | phase-0-prd     | complete              | 01_auth-flow | /prd-check clean
2026-05-17 09:03 | engineering-planner  | phase-1-plan    | started               | 01_auth-flow | task=01-engineering-planner-1
2026-05-17 09:18 | engineering-planner  | phase-1-plan    | complete              | 01_auth-flow | docs/plans/01_auth-flow/ENGINEERING_PLAN.md
2026-05-17 09:19 | team-leader          | phase-1-plan    | gate-surfaced         | 01_auth-flow | gate-0
2026-05-17 09:40 | human                | phase-1-plan    | gate-approved         | 01_auth-flow | gate-0
2026-05-17 11:55 | engineering-reviewer | phase-3-review  | complete              | 01_auth-flow | round=2 blockers=0
2026-05-17 13:10 | human                | phase-4-ship    | gate-approved         | 01_auth-flow | gate-2
2026-05-17 13:10 | team-leader          | phase-4-ship    | final-boss-authorized | 01_auth-flow | gate-2-cleared
```

## Log

<!-- entries appended below this line -->
