# Engineering workflow tooling

A set of small Claude Code skills that handle the overhead around engineering work: filing and updating Azure DevOps stories, scheduling and drafting mail through the desktop Outlook client, drawing diagrams, tracking model usage, onboarding teammates to company-routed Claude Code, and sharing common code between skills.

## Problem

A lot of my time went to work around the work. Azure DevOps stories had to satisfy a strict process template. Finding a meeting time meant checking several calendars across two time zones. Diagrams drawn by hand drifted in style and went stale. Interactive model usage and production model usage ran on the same Azure AI Foundry account and were easy to confuse. Teammates moving off personal Claude accounts needed a repeatable setup. And as skills multiplied, the same authentication and helper code started getting copied between them.

## What I built

- **Story and comment filing for Azure DevOps.** Drafts a user story (title, description, acceptance criteria, effort, tags) from the conversation, previews it, waits for confirmation, then creates it through the REST API. It can also post a short plain-prose status comment to an existing work item.
- **Outlook desktop automation.** One script with a subcommand per task that drives the user's own signed-in classic Outlook through COM: free/busy lookup, finding a common slot for several people across time zones, mailbox search, reading one chosen message, saving attachments, and creating drafts (reply, reply-all, new mail, meeting invite).
- **Diagram builder.** Turns a short JSON spec into an editable Excalidraw file and a PNG in one house style, with a checker that catches layout mistakes.
- **Model usage report.** Pulls token usage per model deployment on the Foundry account from Azure Monitor and groups it by role: interactive Claude Code, production, and unrelated.
- **Teammate onboarding to Foundry-routed Claude Code.** A dry-run-gated access grant that I run, and a portable script the teammate runs on their own machine.
- **A shared library** for modules imported by more than one skill.

## How it works

**Azure DevOps.** The script caches the project's field definitions, area tree, and iteration tree. It resolves the current sprint by walking the project-level iteration tree and picking the iteration whose date window contains today. It validates the area path against the cached tree, converts markdown to the HTML the fields expect, builds a JSON-patch document, and after creating the story it reads it back to confirm assignee, area, sprint, and state took. A dry-run flag prints the patch without posting. Lessons from template errors are written into the skill: stories are created in the initial state and then moved one state at a time, because the template's state machine is sequential even though the transitions API lists every state as reachable, and some fields become read-only after the first transition.

**Outlook.** Searches use the server-side Restrict filter and then sort the small result, instead of walking a folder item by item. The free/busy call returns one character per half hour; the script turns that into free runs, applies working hours in every requested zone, splits runs that cross lunch and flags them, and prints each slot in both zones. Meeting invites are read back after they are built, and the script refuses to save if the stored start time does not match what was asked for, because COM date handling shifts naive times by the UTC offset.

**Diagrams.** A spec lists panels, nodes, pills, and edges, with arrow colors that carry meaning (read, write, AI call, identity, leaves the tenant). The builder seeds every element from the spec, so rebuilding a spec gives a byte-identical file. The checker fails on overlapping text, arrows that do not touch a box, and, for code diagrams, any module or function name that does not exist in the code. Fixes go into the spec, never the output. The renderer loads the scene into Excalidraw in headless Chrome and screenshots the canvas, since the export dialog never downloads in a headless browser.

**Usage report.** The script calls the Azure CLI for request, input, output, and total token metrics per deployment over a time window and prints a table with subtotals per role and an account total.

**Onboarding.** The grant script checks whether the person already holds any role that allows model invocation and exits if so. Only with an explicit flag does it create a least-privilege, invoke-only role assignment on the Foundry account. The teammate script backs up their VS Code user settings, merges in only the two Claude Code settings it owns, and prints the remaining steps: sign in with the Azure CLI, fully restart VS Code, and confirm in Claude Code that usage now shows Foundry.

**Shared library.** Any module used by more than one skill (for example the telephony platform's OAuth client with its token cache) lives in one shared folder, and each skill script adds it to its import path with a one-line shim, so every skill stays self-contained without duplicating code.

## Engineering decisions

- **Outlook through COM, not Microsoft Graph.** App registration and device-code sign-in were dead ends in this environment, while the user's own signed-in desktop client was already there. The script has no send, delete, or move function at all: it only creates drafts for the person to review and send.
- **Previews and dry runs before any state change.** Story creation, comments, and the access grant all show what will happen and wait for an explicit go. The access grant is read-only until a confirm flag is passed, so it is safe to run just to check someone's status.
- **Token volume, not cost.** Azure Cost Management lags by hours, so a same-day dollar figure would be wrong. The report shows volume only, prints zero rows for idle deployments so you can see they exist, and lists any unmapped deployment as unclassified so a new one is never silently missing.
- **CLI before SDK.** The usage and access scripts use the standard library plus the Azure CLI, with no Azure SDK to install, in line with the workspace's tool priority of CLI, then API, then MCP, then custom code.
- **Two onboarding scripts, split by who runs them.** Only someone with elevated rights on the shared Azure resource can make the role assignment, and only the teammate can edit their own machine. VS Code also refuses this Claude Code setting at workspace scope, so a repository cannot silently redirect which account a tool uses; the teammate script writes user scope for that reason.
- **Diagrams are generated, never hand-drawn, and carry data only.** Titles name the subject, boxes hold exact names and values, and explanation belongs in the document, not the picture. A deterministic rebuild makes a diagram reviewable as a diff.

## Stack

Python, Azure DevOps REST API, Microsoft Outlook COM (pywin32), Azure Monitor metrics through the Azure CLI, Azure AI Foundry, Azure RBAC, PowerShell, Excalidraw with Playwright and Pillow, Claude Code skills.

## Skills demonstrated

System design and architecture, APIs and API integration, QA and UAT testing, Debugging and root-cause analysis, Security and compliance, Documentation and playbooks, Python

Platforms: Azure DevOps, Microsoft Outlook, Azure Monitor, Azure CLI, Azure AI Foundry, Azure RBAC, Excalidraw, Playwright, Claude Code

## Timeline

- 2026-04-30: Mirror platform API docs into a local searchable corpus
- 2026-05-20: Add skill that files backlog work items from a session
- 2026-07-08: Migrate script clusters into self-contained skills
- 2026-07-09: Move skill fleet and cloud deployment into new workspace
- 2026-08-10: Install flow-as-code CLI and verify export with app auth
- 2026-08-19: Make agent hooks work across two coding assistants
- 2026-09-29: Add diagram-as-code skill with measured text and PNG render
- 2026-10-14: Scaffold team assistant workspace with harness hooks
- 2026-10-20: Make work-item skill create items then walk states
- 2026-11-02: Add three-way sync and share export for team workspace
- 2026-11-03: Add org inventory skill and per-user API client setup
