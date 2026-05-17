# Changelog

User-facing changes per release. Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning: [SemVer](https://semver.org/spec/v2.0.0.html).

## Who writes here

- **Every agent** appends one internal one-line run entry under `## [Unreleased] — internal` while work is in progress.
- **final-boss only** writes the user-facing versioned section, cut from `## [Unreleased]` at Gate 2 promotion.
- No agent rewrites prior versioned sections.

## Internal run entry format

Flat one-liner. No JSON, no nesting.

```
YYYY-MM-DD HH:MM | <agent> | <feature> | <event> | <note>
```

- **event** — `kickoff` | `plan` | `build` | `review` | `release` | `gate` | `fix` | `close`
- **note** — short clause: artifact path, round, decision, or `-`.

### Example

```
2026-05-17 09:00 | team-leader          | 01_auth-flow | kickoff | cost-ceiling=$20 complexity=complex
2026-05-17 09:18 | engineering-planner  | 01_auth-flow | plan    | docs/plans/01_auth-flow/ENGINEERING_PLAN.md
2026-05-17 09:40 | human                | 01_auth-flow | gate    | gate-0 approved
2026-05-17 11:55 | engineering-reviewer | 01_auth-flow | review  | round=2 blockers=0
2026-05-17 13:10 | final-boss           | 01_auth-flow | release | v1.4.0 staged-rollout=10%
```

## [Unreleased] — internal

<!-- agent run entries accumulate here -->

## [Unreleased]

<!-- user-facing pending changes accumulate here until cut at the next release -->
