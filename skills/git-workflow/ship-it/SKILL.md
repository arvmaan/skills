---
name: ship-it
description: Finalize a completed feature branch and open its pull request. Use when the user says "ship it," asks to prep or raise a PR, or wants working changes made reviewable. Covers cleanup, spec and standards review, tests, docs, commit history, push, and PR creation. Not for deployment or reviewing someone else's change.
---

# Ship It — feature branch to a clean PR

End-to-end "make this branch shippable and open the PR" runbook. Turns a messy WIP feature branch (many small/cache-bust/debug commits) into **one clean, well-documented, best-practices-compliant, buildable commit** (or a few well-scoped ones), then **opens the PR** via the `gh` CLI with a description that carries the manual-testing evidence and observability summary.

This skill composes with the **`better-comments`** skill (comment/doc-comment audit) and follows **`calculated-engineering`** throughout: batch the read-only steps below (1, 3, 4, 4b, 7b) into as few tool calls as possible, don't re-run a green build without a code change, and don't pad the PR description beyond the template.

> **Blast radius / confirm first.** Squashing, rebasing, and opening a PR are outward-facing and rewrite history. Confirm the plan with the user before executing (show the commit range and the proposed commit message). The default endpoint is an **opened PR**; only stop earlier if the user has a standing "don't open it yet" constraint — in that case do all prep + build-green and ask before running `gh pr create`. NEVER `git push --force` to the default/main branch.

## Inputs to establish first

1. **Scope.** Is this one repo, or a change that spans several repos/packages in a monorepo? Each independently-versioned unit (its own build, its own review) gets its own commit and, in a multi-repo setup, its own PR.
2. **The feature branch name.**
3. **The base branch/ref** — usually `origin/main` or `origin/master`, but check (`git branch -r`, or the repo's default branch via `gh repo view --json defaultBranchRef`).
4. **The build/test command(s)** for this repo — check for a `Makefile`, `package.json` scripts, `pyproject.toml`, a CI config file (`.github/workflows/*.yml`), or just ask. Don't guess a command that doesn't exist in the repo.
5. **Any cleanup the user mentioned in this session** — re-read the conversation for things like "revert the skipped test", "remove debug logging", "drop the stray file". These are part of shipping. List them explicitly and address each.

## The pipeline (run in order)

### 1. Survey — find *your* commits, not the whole branch
Feature branches cut from an old base can sit many commits "ahead" of the base for reasons unrelated to this feature. Do **not** open a PR on the raw branch without checking:
```bash
git branch -r                                                # confirm the real base ref
git log --oneline --author="$(git config user.email)" -30    # YOUR commits
git diff --stat <base>..HEAD                                  # what actually changed vs base
```
Identify the exact set of *feature* commits (and the files they touch). Watch for commits that are **not** part of the feature: cache-bust / "reword reason" / debug-logging / stray-artifact commits. Those get dropped or reverted, not shipped.

### 2. Address user-mentioned cleanup FIRST
Before squashing, resolve every cleanup item the user raised this session. Common ones:
- **Revert a skipped/disabled test or timing hack** back to the base-branch version: `git checkout <base> -- <path/to/test_file>` then re-add only legitimate feature test changes. (Verify the ref exists first.)
- **Strip debug/print/INFO logging** added only for live diagnosis (keep the intentional observability — metrics, alerting, WARN/ERROR on real failure paths).
- **Remove stray artifacts** (scratch files, `.DS_Store`, build cruft) and ensure they're gitignored.
- **Note (don't silently drop) any deferred work** — if part of the fix was done manually outside version control (a manual config change, a hand-run migration) and the durable version is part of this ship, include it; if it's a follow-up, call it out in the PR description.

### 3. Comment / doc-comment audit — follow `better-comments`
Apply the **`better-comments`** skill to the feature diff. In short: doc comments carry the contract on public/non-trivial functions; inline comments are **rare and WHY-only, 1-2 lines**; delete anything that restates the code; rename/refactor instead of commenting a bad name.
Audit the diff, not the whole file: `git diff <base>..HEAD -- '*.<ext>'` — for each added function check it has a doc comment (if warranted), and for each added inline comment run the WHY test (restates the what → delete).

### 4. Best-practices pass

Review the feature diff on two separate axes so one cannot mask the other:

- **Spec:** identify missing or partial requirements, scope creep, and behavior
  that appears to implement the requirement incorrectly. Cite the source
  requirement for each finding. If no spec or issue exists, say so.
- **Standards:** check the repository's own instructions and the engineering
  checklist below. Cite the relevant local standard for hard violations.

Run these as independent parallel reviews when workers are available, then
report their findings separately before making fixes.

Standards checklist, adapted to the language(s) in play:
- Imports/module references at the top of the file, not inline fully-qualified names.
- No overly broad authorization in policy or config touched by the change;
  scope resources and actions to the capability required.
- Exceptions/errors caught are logged with context + a metric on real failure paths; no silent catch/swallow.
- Overrides marked as such where the language supports it; constants marked immutable; no magic numbers without a named constant or comment.
- Resource cleanup (bounded caches, closed clients/connections, thread/worker-pool shutdown) — no unbounded growth.
- Null/None/empty guards on external inputs (request fields, config values, env vars).
- Linter/formatter/static-analysis clean (suppress narrowly with a justification only for genuine false positives).
- Unit tests cover the new behavior + failure paths; tests actually assert, not just run.

**Reference-quality bar (patterns from strong reviews — apply, don't just check):**
- **Doc comments state contract AND boundaries, not restatement.** Type-level docs say what the type is and the invariant it upholds; every new field says when it is null/absent and what it must never contain. Function docs name params, return, and every error/exception with the condition that triggers it.
- **Comments explain the why / ordering / invariant.** The best comments say what breaks if you reorder or skip the line. Never comment the what.
- **Guard public entry points explicitly.** Validate/assert on every public function's args, each with a specific message naming the offending field. Reject empty/malformed input at the boundary rather than persisting it.
- **Error messages name the offending entity + why it is rejected**, not just "invalid" (`"destination '%s' header '%s' is reserved and cannot be overridden"`).
- **One small validator/helper per rule**, dispatched from a single public function; add a branch as new variants appear. Prefer early-return guards over nested conditionals.
- **Immutable model objects per operation** rather than long positional argument lists.
- **A metric and a latency timer around every external call** (network, disk, another service): a success/failure counter and a duration measurement, so both a failure spike and a latency regression are alertable.
- **Named constants for magic strings/numbers**, never inline literals.
- **Testable seams:** make a function/method overridable (not fully private) when that lets a test substitute an un-mockable client/dependency; cover the new branches you add rather than leaving them uncovered.

**Reviewer-anticipation bar (what reviewers actually ask for — pre-empt these):**
- **Instrument every decision point, not just failures.** A feature gate, a fan-out, a partition, a publish loop — each gets a counter. If a reviewer would need a log dive to answer "how often does this branch fire?", add the counter now.
- **Every new serialization/parsing/protocol branch gets its own focused unit test** — a positive case (field set → expected effect) AND a negative case (field absent → branch not taken). End-to-end coverage through a higher layer does not count; reviewers flag the missing unit test at the specific site.
- **Answer "can all inputs take the degenerate path?" in the code.** If a gate can be enabled yet produce zero work, make the invariant explicit — a comment naming it + the metric that proves it.
- **State fan-out limits or their absence.** Any one-to-many fan-out draws "is there a cap? what about downstream capacity?" — either enforce a documented limit at validation time or say explicitly in the PR description where the cap lives.
- **Cross-reference links must be live.** When the change consumes or provides a contract shipped elsewhere (a shared schema field, an API contract, another PR), name the exact reference AND verify it's current — copy-pasted trailers from a template or a rebase can point at something unrelated.
- **One logical unit, one PR.** Don't let the same file/module ride in two open PRs — the second one conflicts at merge.

### 4b. Debug-logging removal + observability sufficiency
Two-sided check on logging/metrics — remove the noise, guarantee the signal:
- **Remove debug/diagnostic logging** added only to chase a live issue: `print()`/`console.log()` markers, verbose per-record dumps, high-cardinality logs on the hot path. Grep the diff for the temporary markers added this session and delete them. Keep intentional, durable observability.
- **Operational sufficiency — could an on-call engineer diagnose an incident from this code at 3am?** For every new failure path ask: is it logged with enough context (identifiers, error type/message) to trace the cause? Is there a counter metric on the failure (alertable and graphable), and a latency timer on the critical operation? Verify success *and* failure are both metered so a silent drop-to-zero is detectable.
- **Enumerate every metric the feature adds or changes** (name, unit, labels/dimensions, emitted on success/failure/both). List these in the PR description and in the alerting note (step 8b).

### 5. Squash into one clean commit
Rebase interactively onto the base, squashing the feature commits (dropping cache-bust/debug commits) into **one commit** (or a few well-scoped ones if the branch has genuinely independent sub-features). Prefer a range reset over a long interactive rebase:
```bash
git fetch origin
git reset --soft <base>
# working tree now has all feature changes staged as one; unstage any non-feature files
git restore --staged <non-feature-file>   # if any slipped in
git commit -m "$(cat <<'EOF'
<type>: <one-line summary of the feature>

<2-5 line body: what changed and why, the key design decision, and any
cross-module contract. Mention the flag/rollout gate if there is one.
Note follow-ups if relevant.>

Closes #<issue-number>   # or Fixes/Refs, if the feature has a tracked issue
EOF
)"
```
Standardized message = conventional-commit-style header + a tight body. **Explain everything, not verbosely** — a reviewer should understand the change from the message alone. One logical change per commit.

> After every rebase/squash, read every `-` line of the diff vs the base. History rewrites silently drop base-branch code that landed under your feature's edits; reviewers catch it late. Verify nothing outside the feature's intent was removed.

### 6. Rebase onto the current base branch
```bash
git fetch origin
git rebase origin/<base-branch>       # resolve conflicts; keep the feature intent
```
If rebasing surfaces a build break unrelated to your diff (a dependency moved, a generated file is stale), fix that as a distinct concern — don't fold an unrelated fix into the feature commit.

### 7. Local build/test — MUST be green before opening the PR
Run this repo's actual build and test commands (from step 0's inputs). Redirect large output and tail it:
```bash
<build-command> > /tmp/ship-build.log 2>&1 && tail -20 /tmp/ship-build.log
<test-command> > /tmp/ship-test.log 2>&1 && tail -20 /tmp/ship-test.log
```
Do not proceed to the PR on a red build or red tests. Fix the root cause. Watch for non-compile failures that still block merge: coverage thresholds on new/changed lines, formatter/linter checks on a generated or scratch directory (fix the ignore config, don't just move the directory aside), and flaky tests in files you didn't touch (confirm they pass locally before assuming CI flake — don't silently skip/disable someone else's test).

### 7b. Integration/e2e test coverage audit
Before opening the PR, audit whether the feature's new code paths are exercised by an existing integration or end-to-end test suite (as distinct from unit tests already covered in step 4).

1. Identify the integration/e2e test location for this code — a sibling test directory, a `tests/integration/` or `e2e/` folder, or a separate test package/repo. Check the repo's README or ask the user if unclear.
2. For each new behavior/code path the feature introduces: search for a test that exercises the same path. If found, note which test validates it. If not found, flag it explicitly to the user and ask whether to proceed without it or write the test first — don't silently proceed.
3. Include the result in the PR description under a coverage section (template below). If the user overrides and accepts the gap, note it as deliberate with a justification and, ideally, a follow-up tracking link.

### 8. Open the PR with a testing-evidence description
```bash
git push -u origin <feature-branch>
gh pr create --base <base-branch> --title "<title>" --body "$(cat /tmp/ship-pr-desc.md)"
```
- The title should be a clean imperative summary; the **body MUST be supplied** and MUST include the testing evidence (template below) — this is how reviewers see the change was validated, not just built.
- Link the tracked issue with `Closes #123` / `Fixes #123` in the body if one exists.
- **Never force-push** the base/default branch.
- For a change that depends on another open PR landing first, say so explicitly in the description and mark the PR as a draft until the dependency merges.
- Default to opening the PR (not draft) unless the user says otherwise, or unless the integration-test-coverage gap (step 7b) or an unresolved dependency argues for draft.

**PR description template** — build this from the session's real evidence (do not invent results; pull from actual test runs and notes):
```md
## What
<1-3 sentences: the feature + the key design decision + rollout gate/flag.>

## Cross-module contract (if any)
<e.g. a shared schema field or API contract this change produces/consumes. Link related PRs.>

## Manual testing (extensively validated)
- **Local/staging:** <what was exercised, what was asserted — concrete inputs and observed outputs>
- **Load (if applicable):** <volume, latency p50/p90/p99, success/failure counts>
- **Failure/retry path:** <induced failure → verified behavior (no data loss, correct retry/DLQ)>
- **Regression:** <adjacent functionality unaffected>

## Integration test coverage
- **Covered by:** <test file/name per step 7b>
- **NOT covered (user override):** <gap + justification + follow-up link, if any>

## Observability
- Debug/diagnostic logging added during development has been removed.
- Failure paths log with sufficient context; success and failure are both metered.
- **New/changed metrics:** <name (unit, labels) — emitted on success|failure|both>, ...
- Removed/renamed: <...>

## Rollout
<flag/kill-switch name + default; staged rollout plan if any; manual follow-ups.>

## Docs
<Docs updated in this PR and why. Any docs living outside the repo that still need an edit, with a link.>
```

### 8b. Monitoring/alerting recommendation note (new metrics)
If the feature **adds any metric**, drop a short note to the user recommending what to alert on — do not silently wire alerts yourself. For each new metric state: name, what a spike/drop means operationally, a suggested alert (threshold or anomaly, severity), and where it belongs (existing dashboard/alert group vs new). Let the user decide; wiring alerts is a separate change unless they ask you to include it.

### 8c. Update documentation wherever the change touches it
A feature isn't shipped until the docs that describe it are true. Scan for docs the change makes stale and update them **in the same PR as the code**, so a reviewer sees code + docs together. Common targets — update only the ones the feature actually affects:
- **Repo `README` / contributor docs** — new/removed config keys, flags, endpoints, env vars, build/deploy steps, or a component that no longer exists.
- **Design/spec docs in the repo** — mark the shipped approach as implemented; if the feature diverged from the original design, record the *as-built* decision and why.
- **Config references / runbooks** — a flag whose location or semantics changed, a new operational dial, or a removed resource.
- **Inline module/README pointers** — if you deleted a module a doc names, fix or drop the reference.
Rule of thumb: if you changed *how someone operates, configures, builds, or reasons about* the system, find the doc that says the old thing and correct it. If a doc lives outside the repo (an external wiki, a hosted doc site), note the needed edit in the PR description rather than silently skipping it.

### 9. Report
List the PR opened (link + one-line summary), the build/test results, what cleanup was addressed, the integration-test coverage verdict, which docs were updated (and any external docs still needing edits), and any deliberate follow-ups left out of scope (with why).

## Standardized commit message shape
```
<type>: <imperative one-line summary>

<what + why in 2-5 lines. The key decision and any cross-module contract.
The rollout gate/flag if any. Follow-ups if any.>

Closes #<issue-number>
```
`<type>` ∈ feat | fix | refactor | test | docs | chore. Keep the body information-dense — no restating the diff, no changelog padding.

## Guardrails / do-nots
- Confirm before squash/rebase/PR; these rewrite history and are outward-facing.
- Never force-push the base/default branch.
- Never ship with a red build, a skipped/disabled feature test, an overly broad permission grant, or debug logging left in.
- Don't merge unrelated changes into one commit or one PR.
- Don't silently ship without integration test coverage — step 7b is a user decision, not a skip.
- Don't ship code whose docs now lie — update what the change makes stale in the same PR (step 8c).
