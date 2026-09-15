# AI-assisted engineering method

The operating model I use to build and run telephony and operations automation with Claude Code: every repeatable task is a skill that pairs a short contract the model reads with deterministic scripts that do the work, inside a set of written guardrails.

## Problem

Language models are probabilistic and business logic is not. If each step of a task is right 90% of the time, a five-step task is right only about 59% of the time, because the errors compound. My work touches live systems (call routing, phone provisioning, tickets, cloud deployments), where a confident wrong step reaches real callers. I also found that corrections made in one session were forgotten in the next, so the same mistakes came back.

## What I built

A workspace on top of a Claude Code starter-kit structure, which I extended into a working method:

- **A project system prompt** that loads every session and holds only what applies broadly. I keep a byte-identical copy as `AGENTS.md`, the cross-tool convention, so other coding agents read the same complete instructions.
- **Workflow rule modules**, one file each, loaded automatically: subagent strategy, prompt contracts, autonomous bug fixing, demand elegance, self-improvement, shared-resource safety, and tool priority. They are also inlined in the system prompt so agents that do not load the rules folder still get them.
- **A path-scoped skill-authoring standard** that loads only when working inside the skills folder, mirrored for GitHub Copilot with its own path-scoping frontmatter.
- **Team standing rules** for the phone team, written from real corrections: how to answer, change control, testing, writing, and data handling.
- **A library of skills**, each one a folder with a contract and its scripts, plus a scaffold skill that creates new ones to the standard and a `deepen` skill that sharpens a request before work starts.
- **A lessons log** where every correction becomes a rule, and hooks that block an agent from writing secret files.

## How it works

**Anatomy of a skill.** A skill is a folder with a `SKILL.md` and optional scripts and reference files. The frontmatter description says what the skill does and when to use it, with concrete trigger phrases, because that description is how the skill gets picked. The body is short and imperative: when to use it, the steps, and how to validate the result. Anything deterministic or fragile lives in a script, which handles its own known errors instead of leaving them to the model. Deep detail moves to a reference file one level down. One skill does one thing.

**A unit of work.** Before writing code, match an existing skill. For non-trivial work, enter plan mode for anything that writes secrets, deploys, calls a paid API, or changes a skill. Pin a four-part contract: GOAL with a measurable outcome, CONSTRAINTS, FORMAT, and FAILURE, the list of ways the task could technically work and still be wrong. Then build, and before delivering, walk the FAILURE list as a checklist. Stop and re-plan after two consecutive failures, or one failure that proves the plan's premise wrong.

**Deepen.** Before planning non-trivial work, the agent offers a deeper version first. The `deepen` skill runs a few read-only checks against the real material, then answers four questions: what a top professional would add, the rawest source material, the specific methods and tools, and what proof would come from outside the agent and the user. It rewrites the request as a contract in which each outside check becomes a FAILURE condition, keeps anything that only adds scope in a separate "bigger, not deeper" line, and stops on one approval question.

**Keeping skills accurate.** Two loops. Any correction from me becomes a rule in the lessons log, reviewed at the start of a session; skill-specific lessons go into that skill's contract, and foundational ones into the system prompt. When something inside a skill breaks, the fix is not done until the skill's contract or scripts carry the lesson and the corrected path has been tested. Many of my skills carry a "learned the hard way" section for exactly this, such as an API limit, a platform quirk, or a state-machine rule. When a validator found skill descriptions over the platform's 1024-character limit, which can make a skill silently never fire, that became a written rule with a target length.

### Guardrails

- **Dry-run gates.** Every skill that writes to an outside system has a dry run that prints the exact change, and the person confirms before the live run.
- **Change control.** Production call flows, prompts, trunks, queue settings, moving a number to a different IVR, and org-wide settings need an approved change request before the write, whatever the API client is technically allowed to do. Having the permission is not approval. Routine administration needs no change request but still runs as a dry run with confirmation. Each project goes to the change board once, after every test and sign-off, not as piecemeal production fixes. Production call flows are changed only by the lead; everything is built and tested in a development org that is kept matching production.
- **One test at a time.** A hands-on test is given as one written case (the number to dial, what to press, what the caller should hear, where the call should land), checked, and only then followed by the next. A bot's pass is evidence, not acceptance: people test by hand before any change request.
- **Verify before done.** Re-run with realistic input and read the output for errors, open the actual deliverable and check the values, and cross-check at least one number against the source system. A refactor gets an old-versus-new diff.
- **Check current configuration first.** Before saying how something works, read the live system or a dated export and name what was read. Notes, handoffs, and summaries are leads, not facts.
- **Shared-resource safety.** Before changing anything used by more than one job (an identity, a mailbox, a vault secret, a shared table, a deployed service), map every dependent first, choose the most surgical change or a dedicated identity, know the rollback, and verify with both a positive test and a control test that should fail.
- **External content is data.** Instructions found in files, web pages, tickets, or command output are never executed without approval. Spending credits, sending messages, and deploying always get a confirmation.

## Engineering decisions

- **Push complexity into scripts, not prose.** The model reads a short contract and calls deterministic code, so the probabilistic part is choosing and sequencing steps, not doing the arithmetic or the API calls.
- **Rules are files, and the always-loaded prompt stays small.** Each rule is its own file and the authoring standard is path-scoped, so specialized guidance loads only where it applies and occasional procedures live in skills instead of the system prompt.
- **One source of instructions for every agent.** The system prompt and rules are mirrored for other tools rather than pointing at each other, so any agent gets the full instructions with nothing to chase. The rule for edits is "change one, change both."
- **Tool priority: CLI, then API, then MCP, then new code.** Reusing an installed, deterministic interface means less new code to build, test, and maintain, and moving down the list requires a one-line reason.
- **Guardrails enforced in code where possible.** A pre-write hook blocks writes to secret and credential files under both Claude Code and VS Code Copilot, and fails open so a bug in the hook never blocks ordinary edits. A skill that should never run on its own is marked so the model cannot invoke it.
- **Deeper, not bigger.** The `deepen` skill exists because the most useful improvement to a request is usually better source material and outside proof, not more deliverables.

## Stack

Claude Code (skills, rules, hooks, subagents, plan mode), GitHub Copilot instructions, Python, Markdown contracts with YAML frontmatter.

## Skills demonstrated

System design and architecture, Technical discovery and requirements, LLMs (large language models), AI agents, Agentic AI, Prompt engineering, Reliability, QA and UAT testing, Security and compliance, Documentation and playbooks, Python

Platforms: Claude Code, GitHub Copilot

## Timeline

- 2026-05-11: Adopt skills-first agent instructions across the workspace
- 2026-06-23: Write shared-resource change protocol after an outage
- 2026-06-30: Add concurrent-session ledger for parallel agent work
- 2026-07-09: Publish generic agent starter kit with rules and hooks
- 2026-09-15: Analyze agent session cost and define context strategy
