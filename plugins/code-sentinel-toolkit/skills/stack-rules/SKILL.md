---
name: stack-rules
description: Detects which stacks a diff touches (Kotlin, Python, CloudFormation, NestJS, Next.js, React, React Native) and points to the rule pack for each. Loaded by the Code Sentinel review agents before they review; not usually invoked by hand.
---

# Stack rule packs

The universal rules in each agent assume nothing about the language. The packs in
`references/` add the stack-specific version of each rule. Load a pack only for stacks present
in the diff under review.

## 1. Detect stacks per changed file

Classify every changed file on its own, not the repo as a whole. One repo can hold several
stacks (a monorepo with web apps, a mobile app and shared packages; a service with its
infrastructure templates next to its code).

| Changed file | Stack | Pack |
|---|---|---|
| `*.kt`, `*.kts` | Kotlin | `kotlin.md` |
| `*.py`, `*.pyi` | Python | `python.md` |
| `*.yaml` / `*.yml` / `*.json` containing `AWSTemplateFormatVersion`, or a `Resources:` map whose entries have `Type: AWS::...`; parameter files of the shape `[{"ParameterKey": ...}]` or `{"Parameters": {...}}` next to such templates | CloudFormation | `cloudformation.md` |
| `*.ts`, `*.tsx`, `*.js`, `*.jsx`, `*.mjs`, `*.cjs` | Decided by the nearest `package.json` walking up from the file, see below | |

For JavaScript/TypeScript files, read `dependencies`, `devDependencies` and
`peerDependencies` of the nearest `package.json` and take the first match:

1. `expo` or `react-native` present: React Native. Load `react-native.md` and `react.md`.
2. `next` present: Next.js. Load `nextjs.md` and `react.md`.
3. `@nestjs/core` present: NestJS. Load `nestjs.md`.
4. `react` present: React. Load `react.md`.
5. None of these: plain TypeScript/JavaScript. No pack; universal rules only.

Config and build files follow the stack of the directory they sit in (`build.gradle.kts` is
Kotlin, `pyproject.toml` is Python, `next.config.*` is Next.js, `app.json` / `app.config.*`
beside an Expo app is React Native).

## 2. Read the packs

Read `references/<pack>` for each detected stack, relative to this skill's base directory.
Every pack uses the same headings, so read only the section your agent owns:

| Section | Owner |
|---|---|
| `## Naming` | `code-reviewer`, `quick-reviewer` |
| `## Typing` | `type-design-reviewer`, `code-reviewer`, `quick-reviewer` |
| `## Error handling` | `silent-failure-hunter`, `quick-reviewer` |
| `## Boundaries` | `code-reviewer`, `quick-reviewer` |
| `## State and lifecycle` | `state-sync-guardian` |
| `## Testing` | `test-coverage-analyzer`, `quick-reviewer` |
| `## PR size exclusions` | `pr-contract-checker`, `review-pr` |
| `## Sensitive paths` | `review-pr`, `quick-reviewer` |

A section that reads "Not applicable" means the agent skips that stack's files for that
concern and says so in one line.

## 3. Precedence

Lowest to highest; a later layer wins on conflict:

1. Universal rules in the agent file.
2. The stack pack for the file's stack.
3. `.sentinel-rules.md` at the repo root.
4. `.sentinel-rules.md` in the nearest parent directory of the changed file, when it is
   deeper than the repo root (lets one app in a monorepo carry its own rules).
5. Explicit instructions from the user for this review.

Repo configuration beats pack defaults: if a linter, type checker or coverage config in the
repo sets a stricter or looser rule, the repo's setting is the bar, and the finding should
cite where it was read.

## 4. Report

State the detected stacks in one line at the top of the output, for example:
`Stacks: nextjs (apps/web), react-native (apps/mobile), react (packages/ui)`.
