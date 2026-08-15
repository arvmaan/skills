---
name: better-comments
description: Write and audit code comments and API documentation. Use when adding docstrings or comments, reviewing comment quality, or cleaning verbose or stale comments. Preserve public contracts and non-obvious rationale; remove implementation narration. Not for README or design prose.
---

# Better Comments

Comments that earn their place. The default is **not** "comment everything" and **not** "no comments" — it's: **doc comments carry the contract, inline comments are rare and explain WHY, and everything that only restates the code is deleted.**

The bar for every comment: **"Does this explain why the code exists or why it works this way — something the code itself cannot say?"** If it only restates *what* the code does, delete it. If a reader needs it to use the thing correctly, keep it — and make it short.

Write for three readers: future-you in six months, a teammate seeing this file cold, and an AI assistant that sees **one file at a time** (so a one-line WHY naming a cross-file constraint — a ticket, an upstream contract, a required ordering — stops it from "helpfully" deleting load-bearing code).

## The three levels

### 1. Doc comments (Javadoc/docstring/JSDoc/etc.) — DO write these (they're the contract)
Document the **contract, not the implementation** — what a caller must know to use it correctly without reading the body.

- **Every public/exported type and member gets a doc comment** — it's the API other modules consume without your source.
- **Open with a one-line summary**, third person where the convention calls for it: `Returns the resolved destination.` It's the only text shown in indexes/search/hover tooltips.
- **Document what the signature can't say:** null/none-tolerance ("may be null/None" / "null treated as X"), in-place mutation of a parameter, thread-safety, ordering guarantees, side effects (network/disk/state), and the meaning of an ambiguous return (e.g. "returns 0 for an invalid input rather than raising").
- **Add param/return/throws tags only when they add information beyond the signature.** `@param s the string` / `@return a list` (or the equivalent in your doc-comment convention) is filler — drop it. Never restate the type.
- **Reference constants by name, not value** — "defaults to `MAX_RETRIES`", not "defaults to 3" — so the doc can't rot when the constant changes.
- **SKIP the doc comment when the name already says everything:** simple getters/setters, self-evident one-liners, and overrides that don't change the inherited contract (point at the base implementation's doc instead of repeating it). But don't skip just because the summary would be short — skip only when it would add nothing a reader doesn't already know.
- **Private/internal helpers usually get no doc comment** — a descriptive name is the doc. If a private function needs a comment to explain *what* it does, rename it instead. A private function earns a comment only for a non-obvious WHY.

### 2. Inline comments — write these RARELY, and only for WHY
If doc comments + good names are doing their job, most functions need zero inline comments. When you do add one, it's because something is genuinely non-obvious:

- **A surprising workaround** — name the cause and, when it has one, the expiry: `// Retry with backoff to dodge a known rate-limit storm from the upstream API; remove once it ships backpressure.`
- **A subtle ordering / invariant / concurrency guarantee** the code shape doesn't reveal: `// Must run before flush() — flush() clears the buffer this reads.`
- **A magic-number unit translation:** `payload.length > 1048576 /* 1 MiB */` — or better, extract a named constant.
- **A non-obvious business/policy rule** that can't be expressed in code.

Keep it to **1-2 lines.** Long prose comments rot fastest and get skipped.

### 3. Architectural notes — occasionally, for load-bearing design decisions
A short block explaining a non-obvious design choice and its tradeoff (why a cache here, why this pattern) is worth it *when the decision isn't visible in the code*. Still tight — a few lines, not an essay.

## Delete on sight (comment smells)
- **Restates the what:** `// increment the counter` over `counter++`. Delete.
- **Comments a bad name:** rename the identifier instead of explaining it.
- **A comment marking a section** (`// ---- validation ----`): extract a well-named function.
- **A comment tracking an invalid state** (`// only set when isSuccess is false`): restructure so the invalid state is unrepresentable.
- **Commented-out code:** delete it — version control remembers.
- **`@author`/date tags:** trust git blame.
- **Stale comments** (contradict the code): a wrong comment is worse than none, and worse still for an AI that can't cross-check. Update comments in the same change as the code.
- **Filler doc comments** that only echo the signature.

Rule of thumb: if a reviewer would ask "what does this do?", the fix is usually clearer **code** (better name, extracted function), not a comment.

## TODO / FIXME / HACK
Make them actionable and **traceable to a ticket or issue** — a bare `// TODO` is unmanaged tech debt. Note the expiry condition when there is one.
```
// TODO(PROJ-123): fold into the shared util once another caller needs it.
// HACK: workaround for a timing flake on ARM runners; remove when CI upgrades. See issue #456.
```

## How to apply (writing or auditing)
1. **Auditing a diff:** for each added comment, run the WHY test. Restates the what → delete. Too long → cut to 1-2 lines of the actual reason. Explains a bad name → rename. Verbose doc comment → keep the summary + the non-obvious contract bits, drop the filler tags.
2. **Writing new code:** name things well first; add a doc comment for the public contract; add an inline comment only where you'd otherwise have to explain a WHY out loud.
3. **Never mass-strip.** The goal is the middle ground — keep the docs that carry the contract, remove the noise. Deleting all comments is as wrong as commenting everything.

## Before / after (Java-style doc comments; the pattern applies to any language)
```java
// Filler doc comment → meaningful (covers delimiter + null edge case)
/** Splits a string. @param s A string. @return A list of strings. */      // before
/**                                                                          // after
 * Splits on whitespace, collapsing repeated whitespace.
 * @param s the string to split; {@code null} is treated as empty
 * @return the whitespace-delimited parts
 */

// Getter noise → nothing
/** Gets the account ID. */ public String getAccountId() {...}   // before → delete the comment

// Inline "what" → renamed code
// check if the client caused this, not the server                // before
if (e.getCode() >= 400 && e.getCode() < 500) {...}
if (isClientFault(e)) {...}                                        // after

// Verbose WHY block → one line
// The SDK resolves credentials at client-build time and ignores per-request       // before (5 lines)
// overrides. Key by region + role so the credentials provider re-resolves
// through the cache keyed on the (role, source) captured for THIS destination,
// so a distinct destination must get a distinct client rather than reuse a
// frozen one.
// Cross-account calls need a client built with the assumed credentials (the      // after (2 lines)
// SDK ignores per-request overrides); key on destinationArn so each keeps its own client.
```
