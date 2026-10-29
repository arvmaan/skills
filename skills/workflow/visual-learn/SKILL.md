---
name: visual-learn
description: Build a digestible, interactive learning page from project documents, code, or research. Use when the user asks for “visual learn,” visual onboarding, a system picture, clickable diagrams, background knowledge, or an ordered task plan explained through a minimal locally hosted HTML page.
compatibility: Filesystem tools and Python 3; an available browser tool helps verify rendering.
---

# Visual learn

Turn evidence into a small learning experience that helps the reader understand the system and decide what to do next. Use the actual requested artifact as the first pilot; tune the skill from feedback on that artifact.

## Establish the learning goal

- Infer the reader's role, prior knowledge, ownership, and intended next action from the conversation. Honor corrections before older documents.
- Read the supplied sources and relevant code. Reuse research already completed; investigate gaps that could change the explanation or task order.
- State the evidence date. Distinguish settled decisions, implementation evidence, proposals, and your recommendations.
- Reconcile conflicting documents visibly. Do not turn a POC, “ship it” comment, planned date, or assigned task into proof of production completion.
- Cite the sources near the claims and components they support. Keep restricted sources within the user's authorized local environment.

## Design the learning sequence

Start with the useful picture, then reveal enough detail to act:

1. Explain the goal and the user's ownership in a few sentences.
2. Show a system map with the user's components highlighted, dependencies labeled, and storage or service boundaries visible.
3. Provide an ordered task plan. Define priorities, dependencies, outputs, and done criteria; separate source deadlines from recommended execution order. Put shared contracts and early consultations ahead of dependent implementation.
4. Walk one concrete example through the system. Show what changes, what stays, when readers see the change, and what happens on failure or retry.
5. Reveal prerequisite concepts, a concise glossary, known limits, code reading pointers, and unresolved decisions.

For each concept, connect **what it is → why it matters here → what the reader must do with it**. Define unfamiliar terms at first use. Expandable detail should make the default page easier to digest, not hide required work.

## Make interactions teach

- Prefer two or three purposeful interactions: click a component for its contract, step an example forward, or reveal a task's acceptance criteria.
- Use diagrams for relationships, dependency graphs for ordering, and timelines for dates. Choose the smallest visual that answers the reader's question.
- Show success and failure boundaries. Distinguish logical publication from physical cleanup when they differ.
- Label simulations as illustrative. Never imply that a toy example establishes scale, a production guarantee, or measured performance.
- Keep controls keyboard accessible. Use real buttons, links, labels, and native `details`/`summary`; expose selection and status text accessibly.
- Keep core content readable without interaction. Use clear focus styling, sufficient contrast, responsive layout, and reduced motion.

## Build the smallest page

Use a dedicated output directory containing one `index.html` with inline CSS and plain JavaScript. Native HTML, CSS, and SVG are enough for most learning pages. Reuse an existing project component when one is already present.

Avoid a framework, build pipeline, CDN, remote fonts, analytics, or a new dependency for a few controls. Add files only when they make the result simpler to maintain. Keep explanatory content separate from unnecessary implementation detail.

Choose a compact palette and type scale grounded in the subject. Give the main diagram the visual emphasis; keep prose and supporting controls quiet. Avoid repetitive cards and decorative charts.

Serve only the dedicated artifact directory:

```sh
python3 -m http.server 8765 --bind 127.0.0.1 --directory /absolute/path/to/artifact
```

Check that the port is available and retain the process handle or PID. For a remote devbox, give the actual hostname and SSH forwarding command:

```sh
ssh -N -L 8765:127.0.0.1:8765 user@remote-host
```

Then provide `http://127.0.0.1:8765/`. Never widen the bind to make access convenient, serve the workspace root, or claim the page was opened in the user's browser.

## Verify and iterate

- Confirm HTTP success and a loopback listener.
- Leave one small behavioral check for nontrivial controls: selection changes the right explanation, step navigation stays in bounds, and a reset restores the example.
- If browser tooling is available, inspect desktop and mobile rendering and check interaction, keyboard focus, overflow, and console errors. Otherwise state that visual verification was unavailable.
- Review factual claims against sources, ownership against the latest user instruction, and every task against its dependency and done criteria.
- Deliver the working page URL, remote access instructions if needed, editable file and skill paths, and concise limits of verification.
- Ask for feedback on clarity, task order, and interactions through the normal conversation. Improve those specific issues before adding features. Run broader skill benchmarks only when requested or needed to resolve a demonstrated weakness.

Install or publish the skill only within the user's authorized scope; do not change global skill configuration as an incidental step.
