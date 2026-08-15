---
name: herdr-orchestration
description: Run independent, resumable work lanes in isolated herdr workspaces. Use after orchestration identifies multiple substantial deliverables that need durable terminals or separate worktrees. Not for quick research, one-off commands, or tightly coupled edits.
---

# herdr Orchestration

A repeatable pattern for fanning a task out across several `herdr`-managed
agent workspaces, one per sub-task, then collecting their results. This skill
assumes the `herdr-cli` skill's command reference; read that first if you
haven't used `herdr` before.

Use the `orchestrate` skill first to decide whether durable Herdr workspaces
are warranted. Keep bounded read-only investigations in ordinary workers when
the active harness provides them.

## When this applies

Use this when:

- The task decomposes into **N sub-tasks that don't need to see each other's
  work while running** (they can synthesize at the end, but don't need
  mid-flight coordination).
- At least some sub-tasks may **write files** (code, notes, plans), so giving
  each one its own git worktree avoids collisions.
- You want a **durable, inspectable trail**: each sub-task's agent leaves a
  terminal transcript and, ideally, a written artifact you can read back.

Don't use this for a single tightly-scoped edit, or for sub-tasks that need to
negotiate with each other in real time (that wants a single agent with full
context, not N isolated ones).

## Step 1: Decompose

Before touching `herdr`, write down the sub-task list explicitly:

- One line per sub-task: what it covers, what "done" looks like, what
  artifact it should produce.
- Decide whether sub-tasks need write access to a shared repo. If yes, each
  one needs its own git worktree (isolated branch + checkout). If sub-tasks
  are read-only research, plain workspaces are enough and cheaper to set up.
- Decide the base ref every worktree should branch from (e.g. the repo's main
  branch) and a naming scheme for branches/labels so they're recognizable in
  `herdr workspace list` later (e.g. `proj/<area>` branches, `"<Project>:
  <Area>"` labels).

## Step 2: Provision one workspace per sub-task

For repo-writing sub-tasks, create a worktree per sub-task:

```bash
cd /path/to/repo
for area in area-a area-b area-c; do
  herdr worktree create \
    --cwd "$(pwd)" \
    --branch "proj/${area}" \
    --base <base-ref> \
    --label "Project: ${area}" \
    --no-focus
done
```

Capture each result's `root_pane.pane_id` and `workspace.workspace_id` — you
need the pane id in the next step. Run these in parallel (separate shell
calls in the same batch) since they're independent.

For read-only sub-tasks, use `herdr workspace create --cwd <path> --label
"<label>" --no-focus` instead; no worktree needed.

## Step 3: Start an agent in each pane

Pick one agent kind for the fleet (or mix kinds if some sub-tasks suit a
different model/tool better). Give each a unique, memorable name tied to its
sub-task:

```bash
herdr agent start area-a --kind codex --pane <pane_id_a> -- --dangerously-bypass-approvals-and-sandbox
herdr agent start area-b --kind codex --pane <pane_id_b> -- --dangerously-bypass-approvals-and-sandbox
herdr agent start area-c --kind codex --pane <pane_id_c> -- --dangerously-bypass-approvals-and-sandbox
```

Only pass a full-autonomy flag (Codex's `--dangerously-bypass-approvals-and-sandbox`,
Claude's `--dangerously-skip-permissions`, etc.) when the workspace is
isolated (its own worktree/branch) and the task is scoped enough that
unsupervised action is acceptable. For anything higher-stakes, omit it and let
the agent's normal approval flow run.

## Step 4: Brief each agent with a self-contained prompt

This is the step that determines output quality. Each agent starts with
**zero conversation context** — it doesn't know why the fleet exists, what
the other sub-tasks are, or what "good" looks like unless you tell it. Treat
this like briefing a new contractor, not like continuing a chat.

A good per-agent prompt includes:

1. **What this sub-task covers and what it does NOT cover** (the boundary
   with neighboring sub-tasks, so it doesn't duplicate or drift into their
   territory).
2. **The concrete facts you already know**, inlined directly — don't say "go
   read the report," paste the actual key facts, numbers, IDs, and open
   questions relevant to this slice. Point at source files/docs for depth,
   but don't make the agent hunt for the load-bearing facts.
3. **What "don't do yet" looks like**, if applicable (e.g. "catch up and plan,
   don't write code yet").
4. **The deliverable, and where to put it.** Terminal narration is for a
   human watching live, not a structured return value. Tell the agent to
   write its findings to a named file in its own cwd (e.g. `NEXT_STEPS.md`,
   `findings.json`) so you can `Read` it back deterministically afterward.
5. **Any environment quirks specific to that worktree** (missing checkouts,
   setup commands, known-stale reference material to distrust).

Require the deliverable to include conclusions, evidence, files changed,
checks run and their observed results, unresolved risks, and the recommended
next action.

Send prompts without `--wait` when kicking off several at once, so you don't
serialize on each other:

```bash
herdr agent prompt area-a "$(cat <<'EOF'
<self-contained brief per the checklist above>
EOF
)"
herdr agent prompt area-b "..."
herdr agent prompt area-c "..."
```

## Step 5: Poll, don't block

Long research/planning turns can run for minutes. Poll status instead of
holding a long `--wait`:

```bash
for name in area-a area-b area-c; do
  echo "$name: $(herdr agent get "$name" | jq -r '.result.agent.agent_status')"
done
```

A CLI `--wait --timeout` call that times out means the *wait* gave up, not
that the agent failed — always confirm via `herdr agent get` before treating
a sub-task as errored. Re-poll every 30-90s for tasks expected to take single
digit minutes; longer for open-ended research.

## Step 6: Collect and synthesize

Once every sub-task's agent reaches `idle`/`done`:

- `Read` each sub-task's deliverable file directly from its worktree's
  checkout path (don't rely on scraping terminal narration).
- Skim `herdr agent read <name>` only if you need to see *how* it got there
  (e.g. to sanity-check it actually explored the right files) — the written
  artifact is the source of truth for content.
- Synthesize across sub-tasks yourself. This is exactly the kind of
  cross-cutting judgment that shouldn't be delegated back to a sub-agent that
  only saw its own slice.

## Step 7: Clean up

Decide per sub-task whether its worktree becomes real follow-up work or was
purely exploratory:

```bash
# Keep the branch/checkout (real work to continue or hand off):
herdr workspace close <workspace_id>          # just closes panes, worktree stays on disk

# Discard only after its output and working tree have been reviewed:
herdr worktree remove --workspace <workspace_id> --force
```

Don't remove a worktree with uncommitted changes you haven't reviewed yet —
`Read` or `git diff` inside it first. If a sub-task's output should become a
tracked artifact (a doc, a follow-up plan), copy or commit it out of the
worktree before removing the checkout. Workspace deletion is an explicit
lifecycle decision, not routine cleanup.

## Common failure modes

- **Vague briefs → generic output.** An agent given "catch up on area A and
  plan next steps" with no inlined facts will spend its turn rediscovering
  what you already knew, then produce a shallow plan. Front-load the facts.
- **Forgetting the boundary.** Without an explicit "this sub-task does NOT
  cover X" line, two agents will both produce recommendations about the same
  overlapping piece of work, and you'll have to reconcile contradictions by
  hand.
- **Over-trusting stale reference docs.** If any sub-task's source material is
  known to be stale or contradicted by a more authoritative source, say so
  explicitly in the prompt — agents will otherwise happily repeat the stale
  claim as fact.
- **Skipping the "write it to a file" instruction.** Without it, you're stuck
  parsing terminal scrollback, which is lossy and hard to re-check later.
