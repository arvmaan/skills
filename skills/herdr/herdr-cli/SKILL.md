---
name: herdr-cli
description: Quick reference for driving the herdr terminal-workspace-manager CLI to create workspaces/worktrees, launch AI coding agents (Claude, Codex, etc.) in panes, and script prompts to them non-interactively. Use whenever a task needs one or more isolated agent workspaces spun up and driven from a script or another agent.
---

# herdr CLI

`herdr` is a terminal workspace manager for AI coding agents. It manages
workspaces, tabs, panes, git worktrees, and the interactive agent processes
running inside panes, all through a local socket API that the `herdr` binary
talks to. Every command returns JSON on stdout (errors are also JSON, with a
non-zero exit code).

This skill is a command reference, not an explanation of herdr's UI. Run
`herdr <subcommand> --help` any time you need exact flags; the tree below is
stable but flags can gain options over time.

## Mental model

- **Workspace**: a top-level container (roughly "a project tab group"). Has an
  id like `w7`, a label, and one or more tabs/panes.
- **Worktree workspace**: a workspace whose root pane's cwd is a fresh `git
  worktree` checkout on its own branch. Use this whenever multiple agents will
  touch the same repo concurrently, so they can't stomp on each other's
  uncommitted changes.
- **Pane**: a terminal pane inside a tab. Panes have ids like `w7:p1`.
- **Agent**: an interactive CLI coding agent (`claude`, `codex`, `gemini`,
  `cursor`, etc.) running inside a pane. Once started, the agent gets a name
  you choose, and you drive it by name from then on instead of by pane id.

## Discovering current state

```bash
herdr workspace list                 # all open workspaces, ids + labels + agent_status
herdr workspace get <id>             # detail on one workspace
herdr worktree list                  # git-worktree-backed workspaces (must run inside a git repo)
herdr agent list                     # every agent pane across all workspaces
herdr agent get <name-or-pane-id>    # one agent's status, cwd, session id
herdr pane read <pane-id>            # raw terminal text from a pane
```

`agent_status` is one of `idle`, `working`, `blocked`, `done`, `unknown`.
`working` means a turn is in flight; `idle`/`done`/`blocked` mean it's waiting
on you.

## Creating an isolated workspace per agent

Prefer one git worktree per agent when several agents will work on the same
repo at once. Each worktree gets its own branch and its own checkout directory
so uncommitted changes never collide.

```bash
cd /path/to/repo   # must already be a git checkout with the target base ref fetched

herdr worktree create \
  --cwd "$(pwd)" \
  --branch <new-branch-name> \
  --base <base-ref>          `# e.g. origin/main` \
  --label "<Human label>" \
  --no-focus                 `# omit --no-focus (or pass --focus) if you want it to steal focus`
```

The result JSON includes `workspace.workspace_id` and
`root_pane.pane_id` (format `<workspace_id>:p1`) — capture both; you need
`pane_id` to start an agent and `workspace_id` for later cleanup.

If you don't need git isolation (e.g. read-only research, or agents that
won't write files), a plain workspace is cheaper:

```bash
herdr workspace create --cwd <path> --label "<label>" --no-focus
```

## Starting an agent in a pane

```bash
herdr agent start <name> --kind <kind> --pane <pane_id> [--timeout <ms>] [-- <agent-args>...]
```

- `<name>` is yours to choose; it becomes the handle for every later command
  (`herdr agent prompt <name> ...`, `herdr agent get <name>`, etc). Keep it
  short and unique per running agent.
- `--kind` must be one of the agent's supported executables: `claude`,
  `codex`, `gemini`, `cursor`, `devin`, `cline`, `opencode`, `copilot`, and
  others — run `herdr agent start --help` to see the current list.
- `--pane` must be an existing pane sitting at an interactive shell prompt
  (e.g. the `root_pane.pane_id` from a fresh worktree/workspace create).
- Anything after `--` is passed straight through as CLI args to the agent
  binary itself. This is how you set the agent's own permission/sandbox mode:

```bash
# Codex, full autonomy, no sandbox, no approval prompts:
herdr agent start my-codex-agent --kind codex --pane wC:p1 -- \
  --dangerously-bypass-approvals-and-sandbox

# Claude Code, full autonomy:
herdr agent start my-claude-agent --kind claude --pane wD:p1 -- \
  --dangerously-skip-permissions
```

Only pass those bypass flags in a workspace/sandbox you already trust, since
they remove the agent's own confirmation prompts entirely.

## Sending a prompt and waiting for it to finish

```bash
herdr agent prompt <name> "<prompt text>" --wait --until idle --until blocked --timeout <ms>
```

- Without `--wait`, the prompt is submitted and the call returns immediately
  (fire-and-forget) — use this when you have several agents to kick off and
  will poll them separately.
- With `--wait`, the call blocks until the agent reaches one of the `--until`
  states (repeat `--until` for multiple acceptable end states). Omit
  `--until` entirely to default to matching `idle`, `done`, or `blocked`.
- `--timeout` bounds how long the CLI call itself will block. If the agent is
  still working when the timeout hits, the CLI call returns a `timeout`
  error — **that is not the same as the agent failing**. The agent keeps
  running in its pane; poll with `herdr agent get <name>` afterward instead of
  assuming failure.
- For long or open-ended tasks (deep research, a full implementation pass),
  prefer fire-and-forget plus polling over a single long `--wait` call, since
  you don't know up front how long the turn will take.

Polling pattern:

```bash
herdr agent prompt worker1 "do the thing"     # no --wait: returns instantly
# ...do other work, or start other agents...
while [ "$(herdr agent get worker1 | jq -r '.result.agent.agent_status')" = "working" ]; do
  sleep 15
done
herdr agent read worker1                      # inspect what it actually did
```

## Reading results back

```bash
herdr agent read <name>              # tail of the pane's terminal output (agent's own narration)
herdr pane read <pane_id>             # same, by raw pane id
```

Agent narration in the terminal is for a human watching over your shoulder,
not a structured return value. If you need a structured result, tell the
agent in its prompt to write its findings to a specific file in its cwd (e.g.
`NEXT_STEPS.md`, `findings.json`), then `Read` that file directly instead of
parsing terminal text.

## Cleanup

```bash
herdr workspace close <workspace_id>            # closes panes/tabs in that workspace
herdr worktree remove --workspace <workspace_id> [--force]   # also removes the git worktree checkout
```

Removing a worktree deletes its checkout directory on disk. If the agent left
uncommitted work you still want, `git add`/commit or copy it out of the
worktree's checkout path before removing.

## Common pitfalls

- **Pane not ready**: `herdr agent start` requires the target pane to already
  be sitting at an interactive shell prompt. A freshly created
  workspace/worktree's `root_pane` satisfies this; a pane mid-command does
  not.
- **Wrong wait semantics**: `herdr agent prompt --wait` does not track turns.
  If the agent is already `working` when you submit, that in-flight turn's
  completion (not necessarily your new prompt's) may be what satisfies the
  wait. Check `agent_status` before assuming your specific prompt finished.
- **Treating a CLI timeout as agent failure**: a `timeout` error from `--wait`
  means the *CLI call* gave up waiting, not that the agent crashed. Always
  confirm with `herdr agent get <name>` before reporting a failure.
- **One repo, one branch per worktree**: don't point two `herdr worktree
  create` calls at the same `--branch`; give each agent its own branch name
  even if they'll never be merged, to avoid worktree/branch conflicts.
