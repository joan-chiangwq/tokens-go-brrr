# GLOBAL_CODING_STANDARDS.md

These standards are language-agnostic. They apply to every project, every
language, every framework.

---

## Core Principles

These three principles override everything below them when in conflict.

### Simplicity First
Make every change as simple as possible.
- Prefer the straightforward solution over the clever one.
- If two approaches produce the same outcome, choose the one with fewer moving parts.
- Simple code is easier to test, debug, and hand off. Complexity is a liability.
- When in doubt, delete code rather than add more.

### Minimal Impact
Only touch what is necessary to complete the task.
- Do not refactor surrounding code while fixing a bug.
- Do not add features while implementing another feature.
- Changes should be surgical. Every line changed is a line that could introduce a regression.
- A diff that is twice as long is not twice as good — it is twice as risky.

### No Laziness
Find root causes. Never apply temporary fixes.
- A `# TODO: fix this later` is a debt that will not be paid.
- An empty catch block is not error handling.
- If a test is hard to write, that is signal that the design is wrong — fix the design.
- Senior developer standard: leave the codebase better than you found it, but only in ways that were asked for.

---

Enforced by BUILD and REVIEW. Two tiers:
- **Rules**: non-negotiable. No deviation without explicit user approval.
- **Guidelines**: strong defaults. Agents follow them unless there is a
  justified reason not to. The justification must be stated — silent
  deviation is not permitted.

---

## Rules

These are absolute. Violation is a halt condition for REVIEW.

### Single Responsibility
Every function, class, and file does one thing.
- If a function needs "and" to describe what it does — split it.
- If a class manages two distinct concerns — split it.
- No exceptions.

### No Silent Failures
Every error is handled or explicitly surfaced.
- No empty catch blocks under any circumstances.
- Errors must be caught at architectural boundaries and mapped to a
  known error type before crossing layers.
- Nothing fails without the user or system knowing about it.

### No Skipping Layers
Data flows through the architecture's defined layers in order.
- If the architecture defines DataSource → Repository → Logic → UI,
  the UI never calls a DataSource directly.
- No shortcuts between layers regardless of convenience.

### No New Dependencies Without Approval
Every new dependency requires explicit user approval before adding.
- State the dependency name, its purpose, and what alternatives were
  considered.
- Prefer ecosystem standards over niche packages.

### No TODOs in Merged Code
TODOs are development artefacts. They do not ship.
- If work is incomplete, raise it in the Completion Status Protocol
  as PENDING or ISSUE.
- If future work is needed, log it as a finding or open question in the
  relevant planning doc.

### No Magic Numbers or Strings
Every literal value that carries meaning must be extracted to a named
constant.
- Numeric thresholds, string keys, configuration values — all named.
- The name must describe the meaning, not the value.

### No Debug Output in Production
No print(), console.log(), or equivalent in production code.
- Use the project's designated logger (specified in CLAUDE.md or
  PROJECT_CODING_STANDARDS.md).
- Debug output that slips through is a REVIEW halt condition.

### No Untyped Escape Hatches
No dynamic, any, Object, or equivalent without explicit justification.
- If used, a comment must explain why the type cannot be known at
  compile time.
- REVIEW will challenge every instance.

### Doc Comments on Public APIs
Every public function, class, and data model must have a one-line doc
comment describing what it does or represents.
- Internal/private members: comment only when the logic is non-obvious.
- No redundant comments that restate what the code already says.

### Commented-Out Code Must Explain Itself
Code that is commented out rather than deleted must include a reason.
- Format: `// Removed: [reason]`
- This allows search and recovery. Unexplained commented code is
  treated as dead code and removed by REVIEW.

---

## Guidelines

These are strong defaults. Agents follow them unless deviation is justified.
Justification must be stated — never silently ignored.

### Keep Functions Short
Target: 20 lines or fewer per function.
- Longer functions are acceptable when decomposition would reduce
  readability (e.g. a single sequential pipeline that reads top to bottom).
- If a function exceeds 20 lines, the author must be able to articulate
  why splitting it would make the code worse, not better.

### One Concept Per File
A file should be describable in one phrase without "and."
- Helper functions tightly coupled to the file's primary concept can stay.
- If a file grows beyond its original concept, extract.

### Explicit Types on Public APIs
Public function signatures and class members should have explicit type
annotations.
- Type inference is acceptable for local variables where the type is
  obvious from context.
- When in doubt, annotate.

### Prefer Immutable Data
Default to immutable data structures.
- Use language-appropriate immutability patterns (freezed in Dart,
  readonly/Object.freeze in JS/TS, dataclasses(frozen=True) in Python, etc.)
- Mutable state is acceptable only where immutability creates measurable
  performance problems or unreadable code.

### Prefer Composition Over Inheritance
Build behaviour by composing small, focused pieces rather than deep
inheritance hierarchies.
- Inheritance is acceptable for genuine "is-a" relationships.
- If a class inherits from something primarily to reuse code, refactor
  to composition.

### Descriptive Naming
Names describe what something is or does. Clarity over brevity.
- No abbreviations unless universally understood (id, url, api, etc.)
- Boolean variables: prefix with is, has, can, should.
- Functions: verb-first (getUserProfile, mapErrorToException).
- No single-letter variables outside of trivial loop indices.

### Import Ordering
Consistent ordering across all files:
1. Language standard library
2. Framework imports
3. Third-party packages
4. Local project imports

Enforced per language conventions. No relative imports beyond one
directory level — use absolute/package imports.

### Test Coverage
- Logic layers first, UI last. Tests should cover the code most likely
  to break with the highest impact.
- Target: 80% coverage overall, 90% on core logic layers (business logic,
  data transformation, error mapping).
- Coverage targets are goals, not ceilings. More is fine. Less requires
  justification.
- TDD for core logic. Write the test, watch it fail, make it pass.

---

## What This File Does Not Cover

The following belong in PROJECT_CODING_STANDARDS.md, generated by map-frontend-planner and map-backend-planner agents
on first run:
- Language-specific naming conventions and casing rules
- Framework-specific architecture (e.g. BLoC, Redux, MVC)
- Folder structure
- Model patterns and serialisation
- Widget/component rules
- CI pipeline configuration
- Stack-specific tooling and linting
