# Monthly inbound call report

Builds and emails leadership a monthly report of every inbound call, with each headline number re-checked against the source before it is sent.

## Problem

Leadership wanted a monthly picture of inbound calling: how many calls rang a branch location directly versus the central contact center, which IVR options callers chose, how many were handed to an AI voice agent, and what the caller lookup in the repair-management system returned. The data lives in two places, Genesys Cloud for the calls and a contact-center services vendor's datastore on AWS for the lookup results. I first produced the report on demand from my laptop as a Claude Code skill, which meant it depended on me being there.

## What I built

An Azure Container Apps Job that runs the same pipeline unattended on the first of each month. It pulls the previous month from both sources, builds a self-contained HTML report and a companion appendix that explains how to verify each figure, re-derives every headline number from Genesys, and refuses to send unless they reconcile.

## How it works

Genesys analytics + vendor datastore -> classification and attribution -> HTML report -> validation gate -> email

1. **Classify numbers.** Rebuild the sets of dialed numbers (branch, central contact center, other) from Genesys data tables on every run, so a change to the branch list is picked up automatically.
2. **Lookup index.** Run a parallel DynamoDB scan of the vendor's datastore, filtered to the month plus one day of over-scan, and keep each conversation's lookup result.
3. **Attribution.** Query Genesys analytics conversation details and flow aggregates to attribute calls by dialed number, destination, menu presses, and the AI voice agent handoff funnel (split by business hours and after hours using the IVR's schedule group).
4. **Generate.** Build the HTML report and the validation appendix.
5. **Validate.** A separate script re-derives every headline figure from Genesys and prints a PASS or FAIL line for each.
6. **Send.** Email the report as an attachment through Microsoft Graph from a shared team mailbox, with Sent Items as the archive.

Each step runs as its own subprocess, and any non-zero exit aborts the run.

## Engineering decisions

- **A validation gate that blocks the send.** Any FAIL line raises and nothing goes out. One known boundary artifact, where two counting methods differ at the window edge, is the only tolerated difference.
- **Daylight-saving-proof scheduling.** Cron in Container Apps runs in UTC, so the job fires at two UTC hours and a code guard proceeds only when the local date is the first and the local hour is the target. The other firing exits as a skip.
- **Nothing stored.** All intermediate JSON and HTML lives in a temp directory inside the container, which is discarded. No state store, blob or lock is needed.
- **Distribution is configuration.** Recipients are environment variables, so the list changes without a rebuild, and a parity mode sends only to me with a validation prefix for testing.
- **Skill first, then job.** The deterministic pipeline was proven as a Claude Code skill on my laptop, then copied into the job, and the job's output was checked against the skill's for a settled month before cutover. The laptop skill stays as the fallback producer.
- **Read-only everywhere except one email.** Genesys and the AWS datastore are only read, no AI is involved, and the report contains aggregate counts only.

## Stack

Python, Genesys Cloud analytics API, AWS DynamoDB (boto3), Microsoft Graph mail API, Azure Container Apps Jobs, Key Vault, Managed Identity, Claude Code skills.

## Skills demonstrated

AI voice agents, Production systems, APIs and API integration, Systems integration, End-to-end ownership, Prototypes and proof of concept, QA and UAT testing, Documentation and playbooks, Contact center (CCaaS), Python, AWS

Platforms: Genesys Cloud, AWS DynamoDB, Microsoft Graph, Azure Container Apps Jobs, Key Vault, Managed Identity, Claude Code

## Timeline

- 2026-07-01: Ship monthly inbound report as an unattended cloud job
