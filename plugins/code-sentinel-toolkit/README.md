# Code Sentinel Toolkit

A Claude Code plugin automating the pre-flight PR review in our **Code Review & Testing
Standards**: 6 specialist agents (one per section of the standard) plus a lightweight
`quick-reviewer`, and a `review-pr` command that picks the right depth for the diff and
assembles the results into the team's mandatory report format.

Individually, the agents give fast feedback while you're writing code. Together, they replace
the manual "read the whole doc" pass before opening a PR — at a cost proportional to how much
scrutiny the diff actually needs, not a flat six-agent tax on every PR.

## Supported stacks

The `stack-rules` skill ships one rule pack per stack. The agents detect the stack of each
changed file and apply the matching pack on top of the universal rules:

| Stack | Detected by |
|---|---|
| Kotlin | `*.kt`, `*.kts` |
| Python | `*.py`, `*.pyi` |
| CloudFormation | YAML/JSON with `AWSTemplateFormatVersion` or a top-level `Resources:` using `Type: AWS::...` |
| NestJS | JS/TS file whose nearest `package.json` depends on `@nestjs/core` |
| Next.js | nearest `package.json` depends on `next` (also applies the React pack) |
| React Native | nearest `package.json` depends on `react-native` or `expo` (also applies the React pack) |
| React | nearest `package.json` depends on `react` and none of the above |

Detection is per file, so a monorepo gets the right pack in each package. Each pack covers
naming, typing, error handling, boundaries, state and lifecycle, testing, PR size exclusions
(lockfiles, generated code) and sensitive paths. Where the repo configures something itself
(linter, type checker, coverage threshold), the repo's config wins over the pack's default.

## Repo rules: `.sentinel-rules.md`

Every agent also checks for a `.sentinel-rules.md` file, at the repository root and in the
nearest parent directory of each changed file, and treats its contents as additional, binding
constraints (e.g. framework-specific SDK or interceptor requirements). Precedence, lowest to
highest: universal rules, stack pack, root `.sentinel-rules.md`, nearest `.sentinel-rules.md`,
explicit user instructions. If your repo doesn't have one, the agents run on the universal
rules and stack packs alone, no notice.

## Agents

| Agent | Covers |
|---|---|
| `quick-reviewer` | Single-pass, condensed check across naming/typing/boundaries/reliability/PR-contract — used automatically for small, routine diffs |
| `code-reviewer` | Naming conventions, strict typing at a glance, architectural boundaries (business logic / data access / UI), N+1 query patterns |
| `silent-failure-hunter` | Empty catch blocks, missing structured logging, non-atomic multi-step writes, orphaned data |
| `test-coverage-analyzer` | Testing pyramid placement (unit/integration/e2e), the repo's configured coverage floor (80% when none is configured), mandatory regression tests for bug fixes, success + failure path coverage |
| `type-design-reviewer` | Banning each language's escape hatches (`any`, `!!`, missing type hints), explicit return types, truthiness pitfalls on numeric/string values, type invariants |
| `state-sync-guardian` | Optimistic UI update rollbacks, cache invalidation after mutations, component lifecycle/effect cleanliness |
| `pr-contract-checker` | PR size (~400 line guideline), regression-test presence, reviewer context (repro steps, screenshots) |
| `comment-analyzer` | Comment accuracy, staleness, and maintainability — advisory only, never edits code |
| `code-simplifier` | Post-review polish: simplifies already-correct code while preserving behavior exactly — runs after the review agents find no Critical/Important issues |

`code-reviewer` and `quick-reviewer` score each candidate finding 0-100 confidence and only
report findings scoring 80 or above, on top of their existing precedent/regression judgment
checks — this filters out low-confidence noise before it reaches the report.

Claude will automatically trigger the right agent based on what you're asking about:

- "Check if the tests cover all edge cases" → `test-coverage-analyzer`
- "Review the error handling in the payments client" → `silent-failure-hunter`
- "Does this OrderStatus type look right?" → `type-design-reviewer`
- "The like button updates instantly now, is that safe?" → `state-sync-guardian`
- "Check naming and boundaries on this new service" → `code-reviewer`
- "I'm ready to open this PR" → `pr-contract-checker`
- "I added detailed docstrings to this module" → `comment-analyzer`
- "Review agents are clean, can you tidy this up?" → `code-simplifier`

Or invoke any agent explicitly by name, or run the full pre-flight review:

```
/code-sentinel-toolkit:review-pr
```

Detects the stacks in the diff, loads their rule packs and any `.sentinel-rules.md`, then
picks a tier based on the diff:

- **Lite** (default) — small, routine diffs (~150 lines or fewer, no auth/payment/migration
  paths touched) get one consolidated pass from `quick-reviewer` instead of the full lineup.
- **Full** — triggered automatically by diff size/sensitive paths, or forced with
  `/code-sentinel-toolkit:review-pr --full`. Runs the applicable specialist agents (skipping
  `type-design-reviewer`/`state-sync-guardian` when the diff doesn't touch their domain).

Either way, one report comes out:

```
## PR Review: [PR_TITLE_AND_NUMBER]

### ✅ What's Done Well
### 🚨 Critical
### ⚠️ Important
### 💡 Suggestions
### 🧪 Test Coverage Gaps
### 📋 Recommended Action Order
```

## Severity model

- `test-coverage-analyzer` rates gaps 1-10 (10 = critical, must add before merge).
- All other agents classify findings `Critical`/`Important`/`Suggestion`, mapping onto the
  report's 🚨/⚠️/💡 sections. `review-pr` numbers findings continuously across agents (C1,
  C2… / I1, I2… / S1, S2…), not restarting per agent.

## Note

This toolkit automates the standards document; it doesn't replace the human Reviewer
Responsibilities in the PR Contract (24-hour SLA, user-impact judgment, actionable fixes) —
those stay with your reviewers.
