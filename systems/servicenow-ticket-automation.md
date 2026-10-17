# ServiceNow ticket automation

A set of Claude Code skills that report on, investigate, and act on ServiceNow tickets through the Table API, with deletion locked away in its own disabled skill and production changes drafted for change-board review.

## Problem

Phone and network tickets landed in shared ServiceNow queues, and the phone team needed three things the web UI did not give us. First, a repeatable report on our tickets for a date range. Second, a way to see everything about a ticket: the standard export left out the fields that mattered, because justifications often live in linked catalog form answers, journal entries, and child records rather than on the case row. Third, a way for a person or a bot to work the queue programmatically (open, comment, reassign, close) without ever breaking the audit trail that SOC2 compliance depends on.

## What I built

- **A read-only reporting and forensics skill** with two modes. The production mode pulls our tickets for a date range (by REST API, or by parsing a CSV export when API access is not granted) and writes an HTML report with a Claude-written narrative of patterns and trends. The sandbox mode is a forensic toolkit: an access audit, a keyword refinement pass, a multi-lane bulk fetch with full enrichment, a twelve-month trends report, and a single-record dump.
- **A write skill for CSM cases** with one subcommand per operation: create a case, add a customer-visible comment or an internal work note, create a sub-task, reassign to a group or person, close, and cancel. Every write has a dry-run flag that prints the exact payload and writes nothing.
- **A quarantined delete skill** that exists only to isolate and document the one delete code path in the workspace.
- **A change request drafting skill** that turns a finished piece of work into a change-board request in the house format, for a person to edit and submit.

## How it works

**Reporting.** The production report fetches tickets for my team's assignment groups and team members, summarizes status and type, and sends the structured data to Claude for the narrative. When the service account had no REST role, a CSV parser fed the same report generator, so the report did not wait on access.

**Forensics pipeline (sandbox):**

audit -> refine keywords -> multi-lane fetch -> trends, plus an ad-hoc single-record dump

- **Audit** probes every candidate table with a one-row request and records the HTTP status, field count, and the calling account's roles and groups. All 401s means bad credentials; all 403s means a valid account with no read access. This answers "what can I actually see" before any analysis.
- **Refine keywords** samples recent tickets from our groups and assignees, sends titles and categories to Claude, and gets back a themed keyword list. A person reviews and edits that list before the fetch uses it.
- **Fetch** unions three lanes (assignment group, assignee, keyword), dedupes by record ID, and enriches each record with its journal, field audit history, attachments, catalog form answers, approvals, SLA history, and one level of children. Each record is written as one self-contained JSON line, so downstream tools never have to chase another query.
- **Trends** maps tickets onto the keyword themes, computes resolution-time distributions per theme, and asks Claude for a narrative over the aggregate.
- **Record dump** pulls everything about one ticket, auto-detecting the table from the ticket number prefix.

**Writes.** The write client resolves human values (a group name, a contact or user email) to record IDs automatically, so a caller never has to look them up. It defaults to the sandbox instance and only touches production when told to. Close sets the closed state with required notes and, if the instance rejects the resolution code, retries without it. I worked out the production case lifecycle from closure data (the API account cannot read the audit or configuration tables): a case with a linked incident or request closes through that child, so the skill documents acting on the child, not the case.

**Change requests.** The drafting skill gathers facts from the conversation, verifies anything checkable read-only against the target system and dates the check, drafts every field of the board's form, and stops at the draft. It never submits. Its house style came from corrections on a real request: state the step, do not explain the platform, keep description and justification to two sentences, and use numbered action lists for implementation, backout, and test plan.

## Engineering decisions

- **Deletion is quarantined, not just discouraged.** The write client has no delete method at all. The only delete code lives in a separate skill marked so the model cannot invoke it, and its script refuses unless it gets an exact table and record ID (no lookup, no bulk, no wildcard), an approved exception ticket, and an explicit acknowledgment flag, then a typed confirmation phrase at an interactive prompt. In a non-interactive session that prompt cannot be satisfied, so an agent cannot delete. A ticket's life ends with close or cancel.
- **Never read a non-200 as "no rows."** Every non-success response raises. A silent empty result would hide a permissions gap and make the report quietly wrong.
- **Verify every write by reading it back.** ServiceNow can return success while silently dropping a field the account may not write, so writes request display values and the documented rule is to confirm by read-back.
- **Two instances, two credential sets, never mixed.** The sandbox toolkit is GET-only and must never point at production; the write skill defaults to the sandbox.
- **Small, batched queries.** ID lists are batched into short IN clauses and each keyword runs as its own query, because long lists push the URL past what the API will page.
- **Change requests keep any step whose absence would fail silently.** Rationale gets cut freely, but an action that the change would appear to succeed without, while actually being broken, stays in.

## Stack

Python (requests), ServiceNow Table API and Aggregate API, ServiceNow CSM cases and service catalog, Claude via the Anthropic API, HTML reports, Claude Code skills.

## Skills demonstrated

Technical discovery and requirements, LLMs (large language models), Generative AI, Production systems, Reliability, APIs and API integration, Implementation, Security and compliance, Documentation and playbooks, Python

Platforms: ServiceNow, Claude, Claude Code

## Timeline

- 2026-05-04: Add read-only ticket sandbox toolkit with enrichment
- 2026-07-08: Consolidate ticket report and sandbox tools into a skill
- 2026-07-21: Verify ticket write path and add ticket ops skill
- 2026-07-23: Plan ticket intake, routing and categorization automation
- 2026-09-22: Find fulfilment work by journal authorship
- 2026-10-06: Map service account visibility on the ticket instance
- 2026-10-16: Enable production ticket API and audit closed tickets
- 2026-10-17: Extract team ticket history for categorization
