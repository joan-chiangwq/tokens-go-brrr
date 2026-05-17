# Lessons Learned

Append-only, cross-feature institutional memory. Read by `team-leader` and `engineering-planner` at the start of every run. Each entry is one short section per release, incident, or orchestration close. Capture what was **surprising** or **non-obvious** — not what is already in the code, the PR, or the changelog.

## Who writes here

- **final-boss** appends a release lesson after every Gate 2 production promotion.
- **team-leader** appends an orchestration lesson at Pipeline Close, and immediately after any human correction during triage.
- No other writers. No agent rewrites prior entries.

## Format

```markdown
## <release-version-or-incident-id> — YYYY-MM-DD
**Context:** one-line summary of the feature or incident.
**Surprise:** what we did not expect.
**Future action:** what the next Planner should do differently. Be concrete.
**Tags:** #ios #android #auth #migration #perf #l10n etc.
```

## Entries

<!-- entries appended below this line, newest first -->
