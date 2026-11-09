# Team workspace rollout

A sanitized, GitHub Copilot-ready copy of my Claude Code workspace for the rest of the phone team, kept current by a three-way merge sync and packaged by an export that refuses to ship anything sensitive.

## Problem

My skills and rules lived in my own Claude Code workspace, but my teammates work in GitHub Copilot. They needed the same skills, the same standing rules, and the same guardrails, without my credentials, without the cloud deployment code they were not cleared to run, and without personal details that appear in working notes. My first attempt was a hand port, and within weeks it had fallen behind the source: skills had changed, new skills were missing, and a plain find-and-replace of the skills folder path broke scripts that built their paths from hard-coded folder names.

## What I built

- **A Copilot-native team workspace** that carries the skills, the shared library, the team standing rules, and the instructions, adapted where Copilot needs it.
- **A three-way merge sync** from my workspace into the team workspace, with a dry run before it applies anything.
- **A share export profile** that builds the package a teammate receives, with a gate that refuses to produce it if anything sensitive slips through.
- **Setup and validation scripts:** a setup check a teammate runs on their own machine to prove their configuration end to end, and a validator that checks every skill against the authoring standard.

## How it works

**Source of truth.** My workspace stays the source. The sync keeps a base snapshot of what it last copied, so for each file it can tell whether the change came from my side, from a Copilot adaptation on the team side, or from both. Copilot adaptations survive, my changes flow through, and only real overlaps are held back for review. It runs as a dry run first, then with an apply flag.

**Instructions both tools read.** The project system prompt is mirrored as `AGENTS.md`. Rules that Claude Code loads from its rules folder are also shipped as Copilot instruction files, path-scoped with Copilot's own frontmatter where the rule only applies to part of the tree. The team standing rules go to both places, so a team-wide correction lands for everyone rather than only in one person's memory.

**Export for sharing.** The share profile removes every section of a skill wrapped in cloud-deployment markers, keeps only the one credential set approved for sharing, and leaves every other secret blank. Before it writes the package, a gate scans the output and refuses on any credential value, any cloud-deployment term outside a reviewed allow list, or any listed personal phone number. The skill that isolates ticket deletion is not shipped at all.

**Teammate setup.** Each teammate creates their own API client for the telephony platform, because the platform's audit trail records the client, not the person, and shared credentials would make changes untraceable. The setup check proves each area works with their client. In production, their access comes from a role that leaves out call-flow and prompt changes, and the team rules additionally require an approved change request for those.

## Engineering decisions

- **Sync by three-way merge, never by copying.** A copy either overwrites the Copilot adaptations or forces them to be redone by hand; a base snapshot lets both sides change independently. This came directly from the hand port drifting.
- **Scripts find their files relative to themselves.** Skill scripts locate the shared library and their own files from their own location, never from a hard-coded folder name, so the same script runs unchanged under either tool's folder layout.
- **Sensitive content is cut by markers and caught by a gate.** Marking cloud-deployment sections at the source makes the export mechanical, and the gate is the backstop that fails closed if a marker is missed.
- **Validate skills before they ship.** Copilot and Claude Code both rely on the skill description to decide when a skill fires, and an over-length description can be dropped silently. The team workspace's validator enforces the limit and the rest of the authoring standard.
- **One identity per person.** Personal API clients keep the audit trail attributable, and the production role and change-control rules keep call-flow changes with the lead.
- **Guardrails that work in both harnesses.** The pre-write secret guard reads tool input in both Claude Code's and VS Code Copilot's formats, so it blocks writes to secret files under either tool.

## Stack

GitHub Copilot (agent skills, instruction files), Claude Code, Python, PowerShell, Git, Markdown contracts with YAML frontmatter.

## Skills demonstrated

System design and architecture, Technical discovery and requirements, AI agents, Implementation, QA and UAT testing, Security and compliance, Documentation and playbooks, Python

Platforms: GitHub Copilot, Claude Code, Git
