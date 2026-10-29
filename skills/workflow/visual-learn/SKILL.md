---
name: visual-learn
description: Help someone learn a project or system through a clear, interactive HTML page. Use when the user asks for “visual learn,” visual onboarding, clickable diagrams, background explanations, or a task plan they can explore in their browser.
compatibility: Filesystem tools and Python 3; an available browser tool helps verify rendering.
---

# Visual learn

Explain the system as a colleague would to someone joining the project. Build the page the user asked for, then improve it from their feedback.

## Understand the reader and the sources

- Establish what the reader knows, which work they own, and what they need to do next. Use the latest correction when it changes an older assignment.
- Fold new facts into the main explanation and revise affected tasks or diagrams. Keep the page current; add a separate update history only when the user asks for one.
- Read the supplied documents and relevant code. Reuse completed research; check missing details that could change the explanation or order of work.
- State when the sources were checked. Say which decisions are agreed, which changes are implemented, and which ideas are still proposals. Label your own recommendations.
- Explain disagreements between sources. A working prototype, approval comment, planned date, or assignment does not establish that a feature is deployed.
- Link sources beside the claims and components they explain. Keep restricted material in a location the user is authorized to access.

## Explain the project in a useful order

Give the reader the overall picture before asking them to learn individual components:

1. Explain what the project achieves and which part the user owns.
2. Draw the system. Highlight their components, show what passes between them, and mark where services or storage systems are separate.
3. List tasks in the order you recommend doing them. For each task, explain what to produce, what it depends on, and how to tell it is finished. Keep documented deadlines separate from your recommended order. Agree shared inputs and outputs before building components that use them.
4. Follow one concrete example through the system. Show what each component does, when users see the result, and how the system handles a failure or retry.
5. Add the background needed to understand that example: unfamiliar terms, relevant code, known limits, and decisions still to be made.

Put optional detail behind expandable sections. Keep required work and important qualifications visible.

## Write so a newcomer can follow

Name the actor and explain what it does. For example: “The metadata service publishes the replacement file. A cleanup worker deletes the old file later.” This helps the reader see that publication and deletion are separate events.

Keep technical terms when they identify something the reader needs to learn, but explain each term at first use. Replace compressed noun phrases with sentences that show the relationship. Use headings that tell the reader what the section explains.

Let the idea determine sentence and paragraph length. Split a paragraph when it changes topic; let a simple point stay short. Avoid giving every paragraph the same explanation/example/summary pattern or repeating “why this matters.” Connect ideas directly and use examples when they clarify a real question.

Make claims concrete and attributable. Preserve names, dates, numbers, technical details, and qualifications during a rewrite: “may” must not quietly become “will.” Do not invent experience, evidence, or results to make the prose sound natural.

## Choose interactions that explain the system

- Prefer a few useful controls: click a component to see its inputs and outputs, step an example forward, or expand a task to see how it will be checked.
- Use diagrams to show relationships, a dependency graph to show which tasks must happen first, and a timeline to show dates.
- Show what success and failure look like. If making a result visible and deleting old data happen separately, show both events.
- Label simulations as examples. They do not establish production behavior, scale, or measured performance.
- Use real buttons, links, labels, and native `details`/`summary`. Make controls work with a keyboard and make the selected item and status readable by assistive technology.
- Keep the main explanation readable without clicking. Use visible keyboard focus, sufficient contrast, layouts that fit smaller screens, and reduced motion.

## Build and serve the page

Put one `index.html` in a directory dedicated to the page, with inline CSS and plain JavaScript. HTML, CSS, and SVG are enough for most explanations. Reuse a suitable component if the project already has one.

Avoid adding a framework, build pipeline, remote fonts, analytics, or a dependency for a few controls. Add files only when they make the page easier to maintain.

Follow the user's visual brief across the whole page. Remake a rejected design, including its layout and navigation. Give the main diagram room, keep text easy to scan, and use charts when they explain something.

When the user asks for a desktop application style, use a compact title bar, restrained sidebar, and spacious main area with a clear text hierarchy. Subtle borders, soft shadows, rounded corners, compact controls, and a focused accent can support that style. If they request light and dark themes, warm off-white and deep navy or indigo are useful starting points. Keep dense information readable on smaller screens; avoid oversized marketing headers and repeating the same card layout for every section.

Serve only the page's directory. Bind to `127.0.0.1`, the address for this machine (loopback), so the server is not directly reachable from other machines:

```sh
python3 -m http.server 8765 --bind 127.0.0.1 --directory /absolute/path/to/artifact
```

Check that the port is available and record the server's process ID so it can be stopped later. If it runs on a remote machine, give the actual hostname and an SSH command that forwards the port:

```sh
ssh -N -L 8765:127.0.0.1:8765 user@remote-host
```

Then provide `http://127.0.0.1:8765/`. Keep the server bound to loopback and serve only the page's directory. Do not claim you opened it in the user's browser.

## Check the result and improve it

- Confirm that the page loads over HTTP and the server listens only on loopback.
- Leave one small check for interactive behavior: selecting an item changes the correct explanation, steps stay within the example, and reset restores its starting state.
- If a browser is available, inspect desktop and mobile layouts, try the controls and keyboard navigation, and check for overflow and console errors. Say when you could not inspect the page visually.
- Read the main explanations as if you were saying them to the reader. Rewrite awkward phrases, remove repeated conclusions, and check that the sequence makes sense to someone who does not already know the system.
- After editing, compare names, dates, numbers, claims, and qualifications with the sources. Check ownership against the latest user instruction and each task against its dependencies and completion conditions.
- Provide the page URL, any remote access instructions, editable file and skill paths, and what you did and did not verify.
- Use feedback on clarity, task order, and controls to guide the next revision. Run broader skill benchmarks only when requested or needed to investigate a demonstrated problem.

Install or publish the skill only within the user's authorized scope; do not change global skill configuration as an incidental step.
