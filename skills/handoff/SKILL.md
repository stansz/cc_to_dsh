---
name: handoff
description: Write an execution spec to .handoff/plan.md, run it in a non-interactive agent harness (dsh) in the background, verify and review the result, and report. Use when the user says "hand off" or "hand it off to dsh", "send it to the executor", "run it in dsh", "use dsh", "use deepseek", "have deepseek/dsh do it", or otherwise asks to stop planning and let DeepSeek or another harness implement the work.
allowed-tools: Read, Glob, Grep, Write, Bash(dsh:*), Bash(git rev-parse:*), Bash(git status:*), Bash(git diff:*)
---

Do not implement anything yourself. You plan, the executor implements, you verify.

EXECUTOR: dsh --profile headless

(To target a different harness, change the line above and the command in step 3 to that
harness's non-interactive command, e.g. `pi -p` or `opencode run`, widen allowed-tools to
match, and update the harness names in the description. Nothing else here is
harness-specific.)

## 1. Preflight

- If the request is exploratory or leaves the scope open (e.g. "look into", "what could we
  add", several possible features), stop here. Look into it, give the user the options with
  a recommendation, and wait for them to choose. Do not write the spec until the scope is
  agreed; the spec holds decisions only.
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
    ## Parallel work
    ## Constraints
    ## Out of scope
    ## If blocked

- Decisions only — no reasoning, no alternatives, no hedges. Assume the executor has none
  of this conversation: name exact file paths, function names, commands, and versions.
- **Acceptance** is a list of runnable commands, each with its expected result
  (e.g. `npm test` exits 0; `node greet.js` prints `hello`). No prose-only criteria.
- **Parallel work** decides delegation for the executor:
  - Split into named workstreams when the task has two or more substantial pieces that
    share no files and need no result from each other (e.g. backend endpoint + frontend
    page + docs). For each workstream list: the files it owns, its steps, and its own
    acceptance command. Then add one final integration step for the lead that runs the
    full **Acceptance** after all workstreams finish.
  - Otherwise write `None — do it sequentially.` Small tasks, tightly coupled changes,
    or anything touching the same files stay sequential; delegation overhead is not free.
- **Constraints** always includes: do not commit, push, or switch branches; do not touch
  files outside **Files to touch** except `.handoff/report.md`.

## 3. Run the executor

From the top-level directory, run the EXECUTOR with the Bash tool and
`run_in_background: true`, sending all output to `.handoff/last-run.log`:

    dsh --profile headless "Implement .handoff/plan.md exactly as written. Before you finish, write .handoff/report.md listing files changed, commands run and their results, anything you could not do, and the exact next step. If a step is wrong or impossible, stop and report instead of improvising. Do not commit, push, or switch branches. If the plan's Parallel work section lists workstreams, delegate each one to a teammate or subagent with your agent-team or subagent tools, give each only its own files, wait for all of them, then run the integration step yourself; say in the report which workstreams were delegated." > .handoff/last-run.log 2>&1

Tell the user the executor is running. Do not poll; you will be notified when it exits.

## 4. Verify and review (read-only)

When the run finishes:

- Read `.handoff/report.md`. If it is missing, read the end of `.handoff/last-run.log`.
- Run `git status --porcelain` and `git diff --stat` to see what actually changed.
- Run each **Acceptance** command from the plan and compare against its expected result.
- Review the code: read the full `git diff` and every new untracked file (`git diff` does
  not show those). Check it against `.handoff/plan.md` for bugs, missing or half-done steps,
  changes outside **Files to touch**, and anything **Out of scope**. Passing Acceptance does
  not end the review.
- Do not edit any file during verification.

## 5. Report

Report: exit status, files changed (from git, not the report), each Acceptance check with
pass/fail, your code review findings (or that you found none), anything the executor said
it could not do, and the exact next step. Where the executor's report and your verification
disagree, say so plainly.

Do not re-plan, patch, or fix failures yourself unless the user asks. The user reviews the
diff and commits, including `.handoff/plan.md`. `report.md` and `last-run.log` stay ignored.
