# skills

Personal collection of agent skills, installable into any coding agent that
supports the [Agent Skills spec](https://agentskills.io) (Claude Code, Codex,
Cursor, and others) via [`npx skills`](https://github.com/vercel-labs/skills).

Skills are grouped by category directory. Nothing in this repo references
any company-internal tooling; everything here should work anywhere.

## Categories

- [`herdr/`](skills/herdr) — using the [herdr](https://herdr.dev) terminal
  workspace manager to drive fleets of coding agents.
- [`git-workflow/`](skills/git-workflow) — shipping a feature branch: comment
  hygiene and PR prep.
- [`principles/`](skills/principles) — standing behavioral defaults: minimum
  viable change, surgical edits, and calculated tool/token use.
- [`workflow/`](skills/workflow) — routing substantial work across sessions,
  workers, worktrees, and handoffs; interactive visual learning and onboarding.

## Install

```bash
# Install everything
npx skills add arvmaan/skills --all

# List what's available first
npx skills add arvmaan/skills --list

# Install just the herdr skills
npx skills add arvmaan/skills --skill herdr-cli --skill herdr-orchestration

# Install just the git workflow skills
npx skills add arvmaan/skills --skill better-comments --skill ship-it

# Install the standing behavioral defaults
npx skills add arvmaan/skills --skill calculated-engineering

# Install the orchestration workflow
npx skills add arvmaan/skills --skill orchestrate

# Install the visual learning workflow
npx skills add arvmaan/skills --skill visual-learn

# Install to a specific agent
npx skills add arvmaan/skills -a claude-code
npx skills add arvmaan/skills -a codex
```

## Use without installing

```bash
npx skills use arvmaan/skills --skill herdr-cli --agent claude-code
```

## Layout

```
skills/
  <category>/
    <skill-name>/
      SKILL.md
```

Each `SKILL.md` has YAML frontmatter (`name`, `description`) followed by the
instructions the agent should follow when the skill is active. See
[Creating Skills](https://github.com/vercel-labs/skills#creating-skills) for
the format.

## Influences

The phase-boundary, context-pointer, behavior-level testing, and two-axis
review ideas in this collection are adapted from
[Matt Pocock's skills](https://github.com/mattpocock/skills), released under
the MIT License. See [Third-Party Notices](THIRD_PARTY_NOTICES.md). The
wording and workflows here are tailored to this provider-neutral collection
rather than vendored copies.
