---
name: calculated-engineering
description: Core behavioral discipline for any coding or agentic task - think before coding, minimum viable change, surgical edits, verifiable goals, and calculated tool use. Use as a standing default for all code writing/review/refactoring and for any multi-step agent task, not just when explicitly invoked.
---

# Calculated Engineering

Five rules against the most common ways LLM agents waste effort: wrong silent assumptions, bloated code, drive-by edits, vague goals, and wasteful tool calls.

**Tradeoff:** these bias toward caution over speed. For a trivial one-liner, use judgment.

## 1. Think Before Coding

Don't assume. Don't hide confusion. Surface tradeoffs.

- State assumptions explicitly. If uncertain, ask.
- Multiple valid interpretations exist → present them, don't pick silently.
- Simpler approach exists → say so. Push back when warranted.
- Something's unclear → stop, name it, ask.

## 2. Simplicity First

Minimum code that solves the problem. Nothing speculative.

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or config that wasn't requested.
- No error handling for impossible scenarios.
- 200 lines that could be 50 → rewrite it.

Test: would a senior engineer call this overcomplicated? If yes, simplify.

## 3. Surgical Changes

Touch only what you must. Clean up only your own mess.

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor what isn't broken.
- Match existing style even if you'd do it differently.
- Notice unrelated dead code → mention it, don't delete it.
- Your change orphaned an import/variable → remove it. Pre-existing dead code → leave it unless asked.

Test: every changed line traces directly to the request.

## 4. Goal-Driven Execution

Define success criteria. Loop until verified, not until it "looks done."

- "Add validation" → "write tests for invalid inputs, make them pass."
- "Fix the bug" → "write a test that reproduces it, make it pass."
- "Refactor X" → "tests pass before and after."

Multi-step task → state a brief plan with a verify step per line:
```
1. [step] -> verify: [check]
2. [step] -> verify: [check]
```
Strong criteria let you loop independently. Weak criteria ("make it work") force constant check-ins.

## 5. Calculated Tool Use

Every tool call costs tokens and time. Spend deliberately, not reflexively.

- **Batch independent reads/greps in one call**, not sequential round-trips, when they don't depend on each other's output.
- **Grep/search before you read.** Don't load a whole file to find one function - locate it first, read the relevant range.
- **Don't re-derive what you already know.** If a fact is already in context (a file you just read, a command you just ran), use it - don't re-fetch it "to be sure."
- **Don't re-run a build/test you haven't changed inputs to.** One clean run is a fact; running it again without a code change is waste.
- **Prefer a targeted diff/stat over a full dump.** `git diff --stat`, `grep -n`, or a line-range read beats printing an entire file/log when you only need a slice.
- **One pass, not three.** Plan the sequence of calls before making them instead of discovering the next one after each result lands.
- **Stop when you have the answer.** Don't keep investigating "for completeness" once the question is resolved.

Test: could this task have been done in fewer tool calls without losing correctness? If yes, that's the version to run.
