# cc_to_dsh

Plan in [Claude Code](https://claude.com/claude-code), build with [DeepSeek Harness](https://www.npmjs.com/package/@deepseek-ai/dsh) (`dsh`).

This is a single Claude Code skill. You brainstorm and plan in Claude Code, then say "hand off to dsh". Claude Code writes a spec, runs `dsh` in the background, checks the result itself, and reports back.

The two tools only talk through files, so there is no MCP server, plugin, or webhook to maintain.

## How it works

1. **Preflight:** the project must be a git repo with a clean working tree, so the executor's diff is exactly its own work and you can undo it.
2. **Spec:** Claude Code writes `.handoff/plan.md`. The executor sees none of the conversation, so the plan holds everything it needs, including runnable acceptance commands.
3. **Run:** `dsh --profile headless` runs in the background, logging to `.handoff/last-run.log`. You can keep chatting while it works.
4. **Verify:** Claude Code reads `.handoff/report.md`, checks `git status` and `git diff --stat`, and runs every acceptance command itself. It doesn't rely on the executor's own account.
5. **Report:** you get what changed, pass or fail for each check, and any disagreement between the report and the checks. You review the diff and commit.

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

## Using a different harness

The `EXECUTOR:` line in [SKILL.md](skills/handoff/SKILL.md) names the executor. To switch, change that line and the `dsh` command in the skill's run step to the other tool's non-interactive command, then widen `allowed-tools` to match. The spec format, report contract, and file paths are harness-neutral.

## Repository layout

| Path | Purpose |
|---|---|
| [skills/handoff/SKILL.md](skills/handoff/SKILL.md) | The skill (source of truth) |
| [dsh-handoff-bootstrap.md](dsh-handoff-bootstrap.md) | Design notes, profile setup, and the deferred hook design |
