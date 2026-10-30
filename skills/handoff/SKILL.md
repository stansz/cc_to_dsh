---
name: handoff
description: Write an execution spec to .handoff/plan.md, run it in a non-interactive agent harness (dsh) in the background, verify the result, and report. Use when the user says "hand off" or "hand it off to dsh", "send it to the executor", "run it in dsh", "use dsh", "use deepseek", "have deepseek/dsh do it", or otherwise asks to stop planning and let DeepSeek or another harness implement the work.
allowed-tools: Read, Glob, Grep, Write, Bash(dsh:*), Bash(git rev-parse:*), Bash(git status:*), Bash(git diff:*)
---

Do not implement anything yourself. You plan, the executor implements, you verify.

EXECUTOR: dsh --profile headless

(The line above is the only harness-specific part of this skill. To target a different
harness, replace it with that harness's non-interactive command, e.g. `pi -p` or
`opencode run`, and widen allowed-tools to match.)

## 1. Preflight

- Run `git rev-parse --show-toplevel`. If it fails, stop: the handoff needs git as its undo
  button. Tell the user to `git init` (or offer to) and stop.
- Work from that top-level directory for every step below.
- Run `git status --porcelain -- . ":(exclude).handoff"`. If anything is listed, stop and ask
  the user to commit or stash first, so the executor's diff is exactly its own work.
- If `.handoff/.gitignore` does not exist, write it with these two lines:
  `report.md` and `last-run.log`.

## 2. Write the spec

Write `.handoff/plan.md` with exactly these sections:

    ## Goal
    ## Files to touch
    ## Steps
    ## Acceptance
    ## Constraints
    ## Out of scope
    ## If blocked

- Decisions only — no reasoning, no alternatives, no hedges. Assume the executor has none
  of this conversation: name exact file paths, function names, commands, and versions.
- **Acceptance** is a list of runnable commands, each with its expected result
  (e.g. `npm test` exits 0; `node greet.js` prints `hello`). No prose-only criteria.
- **Constraints** always includes: do not commit, push, or switch branches; do not touch
  files outside **Files to touch** except `.handoff/report.md`.

## 3. Run the executor

From the top-level directory, run the EXECUTOR with the Bash tool and
`run_in_background: true`, sending all output to `.handoff/last-run.log`:

    dsh --profile headless "Implement .handoff/plan.md exactly as written. Before you finish, write .handoff/report.md listing files changed, commands run and their results, anything you could not do, and the exact next step. If a step is wrong or impossible, stop and report instead of improvising. Do not commit, push, or switch branches." > .handoff/last-run.log 2>&1

Tell the user the executor is running. Do not poll; you will be notified when it exits.

## 4. Verify (read-only)

When the run finishes:

- Read `.handoff/report.md`. If it is missing, read the end of `.handoff/last-run.log`.
- Run `git status --porcelain` and `git diff --stat` to see what actually changed.
- Run each **Acceptance** command from the plan and compare against its expected result.
- Do not edit any file during verification.

## 5. Report

Report: exit status, files changed (from git, not the report), each Acceptance check with
pass/fail, anything the executor said it could not do, and the exact next step. Where the
executor's report and your verification disagree, say so plainly.

Do not re-plan, patch, or fix failures yourself unless the user asks. The user reviews the
diff and commits.
