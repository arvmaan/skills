---
name: orchestrate
description: Route substantial work across one session, parallel workers, isolated worktrees, and fresh sessions. Use when a task has independent lanes, a deliverable needs durable isolation, context is crossing a phase boundary, objectives have become unrelated, or work needs a handoff.
---

# Orchestrate

Keep one durable objective in each root session. Use workers for bounded lanes
inside that objective, and create durable sessions only for work that needs its
own branch, terminal, or lifecycle.

## Define the objective

Before choosing a work topology, state:

- the outcome this session owns;
- checkable completion criteria;
- important non-goals or ownership boundaries;
- the evidence required before the work is considered complete.

Reconsider the topology at a phase boundary, such as the end of exploration,
implementation, or verification. Changing context mid-phase discards active
reasoning and usually creates more catch-up work than it saves.

## Choose the smallest adequate unit

| Unit | Use when |
|---|---|
| Current session | The work is cohesive or sequential, and the next step benefits from the reasoning already in context. |
| Worker | A bounded investigation, review, test run, or other independent lane can return a concise result. |
| Worker team | Several independent lanes need steering or must coordinate during one objective. |
| Isolated worktree or durable session | A deliverable is independently resumable or shippable and needs its own branch, checkout, or terminal. |
| Fresh root session | The objective is unrelated to the current work, even if it uses the same repository. |
| Handoff | The same objective must move to another person, harness, repository, or fresh context. |

Prefer the current session when it still holds primary reasoning that the next
phase needs. Summaries preserve conclusions but lose dead ends, tradeoffs, and
the path that made a decision credible.

## Delegate deliberately

Identify independent lanes before implementation. Use one to three workers
when parallelism or context isolation materially helps. Keep immediate blockers
and tightly coupled edits in the main session, and avoid assigning the same
question to multiple workers unless comparison is the purpose.

Give each worker:

- one bounded objective and an explicit stopping condition;
- scope boundaries, including files it may change;
- the relevant facts and source pointers it cannot infer safely;
- a concrete deliverable and a checkable definition of done;
- any commands, constraints, or environment details needed to verify the work.

Require every worker to return:

- conclusions and decisions;
- evidence with file, command, URL, or transcript references;
- files changed and checks run;
- unresolved risks or questions;
- the recommended next action.

## Isolate writes

Parallel readers can share a checkout. Parallel writers should own disjoint
files or use separate worktrees. A worktree provides filesystem and branch
isolation; it does not remove the need for clear ownership or an integration
plan.

Use a durable worktree or session only when the deliverable must be resumed,
reviewed, or shipped independently. A quick command, a narrow research lane,
or a single review does not need a new long-lived session.

## Assimilate results

Wait for required workers, inspect their evidence, and integrate the relevant
conclusions into the parent objective. Do not paste entire worker transcripts
into the main context. Preserve source pointers for claims that may need later
audit, and close completed workers once their results are incorporated.

## End with a disposition

State one lifecycle decision when substantial work ends:

- **continue**: the same objective remains active in this session;
- **park**: work has a concrete condition for resumption;
- **archive**: the deliverable is complete or retained only as reference;
- **handoff**: the objective continues in a fresh context with a concise state
  summary and source pointers.

Do not delete branches, worktrees, transcripts, or other durable state as
incidental cleanup. Verify that anything worth retaining is committed,
captured, or referenced before removing it.
