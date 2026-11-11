# cc_to_dsh

Plan in [Claude Code](https://claude.com/claude-code), build with [DeepSeek Harness](https://www.npmjs.com/package/@deepseek-ai/dsh) (`dsh`).

This is a single Claude Code skill. You brainstorm and plan in Claude Code, then say "hand off to dsh". Claude Code writes a spec, runs `dsh` in the background, checks the result itself, and reports back.

The two tools only talk through files, so there is no MCP server, plugin, or webhook to maintain.

> **Want dsh to have full access (browser, `dnf install`, any tool) without trusting it with your machine?**
> [dsh-vm](https://github.com/stansz/dsh-vm) runs the same handoff inside a throwaway microVM
> (Podman + libkrun) on a clone of your repo, and is now the recommended default on Linux with KVM.
> See [Running in a microVM](#running-in-a-microvm) below.

## How it works

1. **Preflight:** the project must be a git repo with a clean working tree, so the executor's diff is exactly its own work and you can undo it.
2. **Spec:** Claude Code writes `.handoff/plan.md`. The executor sees none of the conversation, so the plan holds everything it needs, including runnable acceptance commands. Claude Code then tries tool-dependent commands on a small temporary example, so a command that can't work is fixed before `dsh` spends anything. This may ask for permission to run tools like `node` or `npm`. Commands that can't be tried on a small example are left as written and flagged as unchecked in the final report.
3. **Run:** `dsh --profile headless` runs in the background, logging to `.handoff/last-run.log`. You can keep chatting while it works.
4. **Verify and review:** Claude Code reads `.handoff/report.md`, checks `git status`, and runs every acceptance command itself. It then reads the full diff and any new files and reviews them against the plan. It doesn't rely on the executor's own account.
5. **Report:** you get what changed, pass or fail for each check, the code review findings, and any disagreement between the report and the checks. Claude Code doesn't fix anything unless you ask. You make the final call on the diff and commit.

When the work splits into independent pieces that share no files, the plan lists them as parallel workstreams and tells `dsh` to delegate each one to a teammate or subagent. Small or tightly coupled tasks stay sequential.

## Requirements

- Claude Code
- `dsh`, installed globally: `npm install -g @deepseek-ai/dsh`
- `DEEPSEEK_API_KEY` set in the environment that launches Claude Code

Keys saved on the `dsh web` Models page are not enough. The headless profile deliberately skips dsh's credential store and reads the key from the environment. Restart Claude Code after setting it.

Check the setup with:

```bash
dsh --profile headless "Reply with exactly: handoff pipe ok"
```

## Install

Copy the skill into your user-level Claude Code skills so it works in every project:

```bash
mkdir -p ~/.claude/skills/handoff
```

```bash
cp skills/handoff/SKILL.md ~/.claude/skills/handoff/SKILL.md
```

That is the whole install. Nothing goes in `CLAUDE.md`, because the skill holds every rule and only loads when you use it.

## Usage

Start Claude Code in your project folder, plan the work, then use any of these:

- `/handoff`
- "hand off to dsh"
- "use dsh" or "use deepseek"
- "send it to the executor"

To check the plan before it runs, say "write the handoff plan but let me review it first."

Settle open decisions before handing off, because `dsh` only sees what's in the plan.

Each project gets a `.handoff/` folder. `plan.md` is meant to be committed. `report.md` and `last-run.log` are ignored through `.handoff/.gitignore`.

## Configuring the executor

The skill works with the default headless profile. Section C of [dsh-handoff-bootstrap.md](dsh-handoff-bootstrap.md) shows how to add agent teams and Brave search, and how to set the reasoning effort, in `~/.dsh/profiles/headless/`.

By default the headless sandbox allows writes inside the project and network access. It blocks writes outside the project, and approval requests are denied automatically because nobody is there to answer them. `dsh` may work around this with files such as a project-local `.npm-cache/`, which will show up in your diff.

## Running in a microVM

[dsh-vm](https://github.com/stansz/dsh-vm) is a companion repo that keeps this skill's plan,
verify and report flow but moves the run step into a microVM:

- `dsh-vm run` clones the repo (with `.handoff/`) into `~/.cache/dsh-vm/runs/<repo>-<time>/` and
  runs `dsh --profile headless --permission-mode danger-full-access` in `podman --runtime krun`:
  its own kernel, 2 GB, 2 CPUs, deleted on exit.
- Inside, dsh runs as root with no sandbox and no approvals, so it can use headless Chromium,
  `dnf install` packages, compile native modules, and use the network. It has no SSH keys,
  `gh` token or git remote, so it cannot push.
- Claude Code verifies in the run dir, and `dsh-vm apply` brings the changes into your repo as
  uncommitted changes only after you approve.

dsh-vm ships its own skill named `handoff`, which replaces this one. Its README shows how to keep
this one as `handoff-local`, a fallback for machines without Podman or KVM:

| Skill | Where dsh runs | Security boundary |
|---|---|---|
| `handoff` (from dsh-vm) | microVM, on a clone | the VM + no push credentials |
| `handoff-local` (this repo, renamed) | your host, in your repo | dsh's own sandbox |

## Using a different harness

The `EXECUTOR:` line in [SKILL.md](skills/handoff/SKILL.md) names the executor. To switch, change that line and the `dsh` command in the skill's run step to the other tool's non-interactive command, widen `allowed-tools` to match, and update the harness names in the skill's description. The spec format, report contract, and file paths are harness-neutral.

## Repository layout

| Path | Purpose |
|---|---|
| [skills/handoff/SKILL.md](skills/handoff/SKILL.md) | The skill (source of truth) |
| [dsh-handoff-bootstrap.md](dsh-handoff-bootstrap.md) | Design notes, profile setup, and the deferred hook design |
| [stansz/dsh-vm](https://github.com/stansz/dsh-vm) (separate repo) | The microVM executor and its `handoff` skill |
