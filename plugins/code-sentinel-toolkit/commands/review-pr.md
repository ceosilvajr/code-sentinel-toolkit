---
description: Run the Code Sentinel pre-flight review — a lite single-pass for small/routine diffs, or the full specialist pipeline for large/sensitive ones — and produce the standard PR review report.
argument-hint: "[optional: file/dir scope, commit range, or PR number; add --full to force the full pipeline]"
---

Run the mandatory pre-flight PR review.

1. **Scope**: use `$ARGUMENTS` if given (minus any `--full` flag), else the unstaged changes.
   Get the touched file list and line count (`git diff --stat`) — not the full diff content —
   to decide scope, tier, and routing below. State the scope used.
2. **Rules**: load the `code-sentinel-toolkit:stack-rules` skill and detect the stack of each
   changed file with its detection table. Read each detected pack from the skill's
   `references/`. Then check for `.sentinel-rules.md` at the repo root and in the nearest
   parent directory of each changed file; if present, read them. Pass the stack list, the
   relevant pack sections and `.sentinel-rules.md` contents to every agent below as binding
   context. Universal standards always apply regardless; don't call out a missing
   `.sentinel-rules.md` in the report. State the stacks in one line (e.g. "Stacks: nextjs
   (apps/web), python (services/api)"), or "Stacks: none detected, universal rules only".
3. **Pick a tier.** Default to **Lite** unless any of these hold, in which case use **Full**:
   - `--full` was passed, or the user asked for a thorough/full review.
   - More than ~150 lines changed, not counting files the detected packs list under
     `## PR size exclusions` (lockfiles, generated code, build output).
   - Any touched path suggests auth, payments/billing, or a migration (matches
     `auth|payment|billing|migrat|security`, case-insensitive), or matches a detected pack's
     `## Sensitive paths`.
   State the tier and the reason in one line under the scope statement (e.g. "Tier: Lite — 38
   lines changed, no sensitive paths. Add --full to force the complete pipeline.").
4. **Launch:**
   - **Lite** → launch `quick-reviewer` only.
   - **Full** → select and launch the specialist agents: always `code-reviewer`,
     `silent-failure-hunter`, `test-coverage-analyzer`, `pr-contract-checker`; add
     `type-design-reviewer` only if the diff adds/changes a type, interface, DTO, or schema
     in a file whose pack does not mark Typing "Not applicable"; add `state-sync-guardian`
     only if the diff touches react, nextjs or react-native files. Note any skipped
     specialist and why.
   Either way (via Task tool, in parallel where more than one agent runs): pass the scope
   description, the stack list, the pack sections that agent owns (see the skill's
   section-owner table) and `.sentinel-rules.md` contents if found, and let each agent pull the actual
   diff content itself via `git diff` inside its own isolated context, rather than including
   the full diff in this command's own context.
5. **Merge** (Full tier only): de-duplicate overlapping findings across agents (keep the more
   specific one), sort each bucket by user/data impact, most severe first. If `quick-reviewer`
   flagged that something needs deeper scrutiny, say so above the report and suggest re-running
   with `--full`.
6. **Render** using exactly this format — no extra headers, write "None found" for empty
   sections:

```
## PR Review: [PR_TITLE_AND_NUMBER]

### ✅ What's Done Well
- [Brief highlight of good architecture, thorough testing, or clean code]

### 🚨 Critical (Must fix before merge: silent-failure, security, blocker)
- **C1. [Concise Title]** (`[file_path]:[line_numbers]`): [2-3 sentence technical explanation of bug + specific fix]. **Walkthrough:** [detailed steps to trigger live].

### ⚠️ Important (Should fix: architecture, test gaps, observability)
- **I1. [Concise Title]** (`[file_path]:[line_numbers]`): [Description of issue]. **Walkthrough:** [detailed steps to trigger live].

### 💡 Suggestions (Refactoring, minor UI polish, DRY)
- **S1. [Concise Title]** (`[file_path]:[line_numbers]`): [Brief suggestion for code cleanliness].

### 🧪 Test Coverage Gaps
- **T1. [Target File/Component]:** [Specific Test Case Name] — [Behavior to assert (e.g., "pins regression")].

### 📋 Recommended Action Order
1. **Critical:** Fix C1, C2... (Blockers & Data Integrity)
2. **Important:** Address I1, I2... (Test gaps & Observability)
3. **Suggestions:** S1, S2... (Polish)
```

Number C/I/S/T continuously across all agents, not restarting per agent.
`pr-contract-checker` findings land in Important/Suggestions unless blocking (e.g. no
regression test at all on a bug-fix PR, which is Critical).
