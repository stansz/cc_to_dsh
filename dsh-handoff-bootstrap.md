# Bootstrap: Claude Code → agent-harness handoff

## Goal

Claude Code plans and verifies; a cheaper non-interactive harness (DeepSeek Harness, `dsh`)
implements. When I say "hand off", "send it to the executor", or "run it in dsh", Claude Code
writes an execution spec to `.handoff/plan.md`, runs the executor in the background, verifies
the result read-only, and reports back.

The only harness-specific part is one `EXECUTOR:` line in the skill. The spec format, report
contract, and paths are harness-neutral, so the executor can be swapped (Pi, opencode,
anything with a non-interactive CLI) by editing that line.

No MCP server, plugin, webhook, or other integration mechanism — files in, files out.

## Layout

| Path | Role |
|---|---|
| `skills/handoff/SKILL.md` (this repo) | Source of truth for the skill |
| `~/.claude/skills/handoff/SKILL.md` | Installed copy (user level, works in every project) |
| `~/.claude/CLAUDE.md` | Global handoff rules (section B) |
| `~/.dsh/profiles/headless/` | Executor profile (section C) |
| `.handoff/` (in each target repo) | `plan.md` committed; `report.md`, `last-run.log` ignored via `.handoff/.gitignore` |

To update the skill: edit the repo copy, then copy it over the installed one.

## Preconditions

1. `dsh --version` succeeds and resolves to a global install
   (`%APPDATA%\npm\dsh*`), not an `_npx` cache. Install / update:
   `npm install -g @deepseek-ai/dsh --allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs`
   (re-pass `--allow-scripts` on every update, or native modules are skipped).
2. `dsh --profile headless --help` exits 0.
3. `DEEPSEEK_API_KEY` and `BRAVE_API_KEY` are set as **Windows user environment variables**.
   The headless bundle deliberately disables dsh's credential store ("one-shot jobs use
   launch-time credentials"), so keys saved on the web Models page are invisible to it.
   Rotating a key in the web UI means updating the env var too. Restart Claude Code after
   changing either.
4. Pipe test: `dsh --profile headless "Reply with exactly: handoff pipe ok"` exits 0 and
   stdout contains `handoff pipe ok`. (Reasoning and diagnostics go to stderr.)

---

# Phase 1 — the skill (built)

## A. Skill

See `skills/handoff/SKILL.md`. Its flow:

1. **Preflight** — must be in a git repo with a clean tree (excluding `.handoff/`);
   otherwise stop. Ensures `.handoff/.gitignore` exists.
2. **Spec** — `.handoff/plan.md` with Goal / Files to touch / Steps / Acceptance /
   Parallel work / Constraints / Out of scope / If blocked. Acceptance = runnable commands
   with expected results. Parallel work names independent workstreams (files owned, steps,
   own acceptance command, plus a final integration step) or says `None — do it
   sequentially.`; the executor prompt tells dsh to delegate listed workstreams with its
   agent-team tools. Constraints always forbid commit/push/branch switches and edits
   outside Files to touch.
3. **Run** — `dsh --profile headless "<prompt>" > .handoff/last-run.log 2>&1` via Bash with
   `run_in_background: true` (no 2/10-minute Bash timeout; Claude is notified on exit).
4. **Verify (read-only)** — read `report.md` (fallback: `last-run.log`), `git status`,
   `git diff --stat`, run every Acceptance command.
5. **Report** — git's view of changes, pass/fail per Acceptance check, and any disagreement
   between the executor's self-report and verification. No fixing unless asked.

## B. `~/.claude/CLAUDE.md` section

```markdown
## Handoff to an external agent harness

Claude Code plans and verifies; an external harness (dsh, headless profile) executes.

- When asked to "hand off", "send it to the executor", "run it in dsh", "use dsh", or
  "use deepseek" (or anything meaning "let DeepSeek implement this"), use the `handoff` skill.
  Do not also implement the work yourself in that turn.
- The executor starts with **zero context** from this conversation. Everything it needs must
  be in `.handoff/plan.md`, and its Acceptance section must be runnable commands.
- Handoffs require a git repo with a clean working tree; git is the undo button.
- `.handoff/plan.md` is the spec and is committed. `.handoff/report.md` and
  `.handoff/last-run.log` are the executor's output and are ignored.
- The executor command is the single `EXECUTOR:` line in the `handoff` skill. Changing
  harnesses means editing that one line, nothing else.
- After a handoff, verify read-only and report. Do not re-plan, patch, or "fix up" the
  executor's work unless explicitly asked.
```

## C. Headless executor profile

Defaults (from `dsh --profile headless --dump-config`): `deepseek-flash`, reasoning effort
unset, regular subagents, web search via DeepSeek, `workspace-write` sandbox.

Sandbox (kept at default): writes inside the working directory and network are allowed;
writes outside it are blocked, and approval requests **fail closed** (no human answerer in
headless). Expect occasional workarounds such as a project-local `.npm-cache/`.
`DSH_PERMISSION_MODE=danger-full-access` would remove the sandbox — deliberately not used.

Changes made:

`~/.dsh/profiles/headless/package.json` — bundles:

```json
"bundles": [
  "@deepseek-ai/dsh-base",
  "@deepseek-ai/dsh-headless",
  "@deepseek-ai/dsh-experimental-agent-team-profile",
  "dsh-brave-search"
]
```

- Agent Teams is referenced by name only — it resolves to the copy shipped inside dsh.
  **Do not** `dsh plugin add` it: that installs a stale npm version (`0.1.5-alpha.2` vs the
  shipped `0.2.1-rc.5`). It replaces the `subagent` tools with `spawn_teammate`,
  `team_task_*`, `wait_agent`, etc.
- `dsh-brave-search` is installed from `github:stansz/dsh-brave-search` via
  `dsh plugin --profile headless add github:stansz/dsh-brave-search` (same source as the
  web profile).

`~/.dsh/profiles/headless/cordis.patch.yml`:

```yaml
- id: agent-default-model
  name: "@deepseek-ai/dsh-agent-default-model"
  config:
    provider: deepseek-official
    model: deepseek-flash
    reasoningEffort: high
- id: brave-search
  name: dsh-brave-search
  config:
    route: snippets
```

Overrides replace a row's whole `config`, so every key is restated. Brave uses the
`snippets` route (~440 tokens/search) rather than the web profile's `context`
(~6,200 tokens/search) to keep the coder's context small.

## Phase 1 acceptance criteria

- Pipe test passes (precondition 4).
- `~/.claude/skills/handoff/SKILL.md` exists, matches the repo copy, frontmatter parses,
  exactly one `EXECUTOR:` line.
- End-to-end: in a throwaway git repo, a trivial plan (create `greet.js`; Acceptance:
  `node greet.js` → `hello, world`, `node greet.js Ada` → `hello, Ada`) runs in the
  background, writes `.handoff/report.md`, changes only `greet.js`, and both Acceptance
  commands pass on verification.
- No real coding task is run against this repository during setup.

---

# Phase 2 — hook (on hold)

Only build this if Phase 1 shows Claude actually skipping or mis-running the executor.
Rationale for holding: a `PostToolUse` hook still depends on Claude writing `plan.md`
(the same cooperation as running one command), blocks the session synchronously for up to
its timeout, hides executor stdout from Claude on success, and stays armed for the rest of
the session. Phase 1's background run has none of these costs.

If it is ever built, the original design needs these fixes (verified against
https://code.claude.com/docs/en/hooks):

1. **Drop `once: true`.** The hook matches every `Write`; the first write is usually the
   draft, the script exits 0 for it, that counts as a successful run, and the hook
   disarms before `plan.md` is ever written.
2. **Wrap the executor call in `try/catch` → `exit 2`.** Under
   `$ErrorActionPreference = 'Stop'` a missing executor throws, PowerShell exits 1, and
   exit 1 is a silent non-blocking error.
3. **Write executor output to `.handoff/last-run.log`.** PostToolUse stdout on exit 0 goes
   only to the debug log. In PS 5.1, set `$ErrorActionPreference = 'Continue'` around a
   `2>&1` redirect or stderr lines become terminating errors.
4. **Skill must tell Claude to read `.handoff/report.md`** after writing the plan; the hook
   gives it nothing on success.
5. Use exec form (`"command": "powershell.exe", "args": [...]`) to avoid shell quoting.
6. Verification step 4 ("second write does not fire") inverts once `once` is dropped.

## Constraints

- Windows, PowerShell / Git Bash.
- Do not commit during setup; leave the working tree for review.
- No multi-executor config or per-harness branching. One skill, one `EXECUTOR:` line.

## Out of scope

- MCP servers, Claude Code plugins, the `dsh-web-submit` plugin, webhooks.
- The web profile and any running `dsh web` process.
- Multi-round `--session-id` flows and `--json` event parsing (follow-ups are a fresh
  handoff for now).
