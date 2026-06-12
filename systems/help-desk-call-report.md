# IT service desk call report with AI narrative

Emails IT leadership a week-to-date report on the IT service desk's calls (answered, voicemail, abandoned, outbound) with a short written analysis from Claude.

## Problem

The IT service desk is a Genesys Cloud call group: every member's phone rings, with no menu or queue in front, so a call is answered, rolls to group voicemail, or is abandoned. Leadership wanted to see how calls were handled and how much outbound calling the team did, with a written summary. The existing report ran as a workflow in a self-hosted low-code automation tool that was being retired, and it failed outright whenever its AI step failed.

## What I built

A scheduled Azure Container Apps Job that runs every evening, pulls the current week's calls, classifies each one, asks Claude on Azure AI Foundry for a few paragraphs of narrative, and emails the HTML report as an attachment from a shared support mailbox through Microsoft Graph. Every number in the report is computed in code. The model writes prose only.

## How it works

Genesys group and analytics APIs -> classify and aggregate -> Claude narrative -> HTML report -> email attachment via Graph

- **Context.** Looks up the group, its members, and the group's ring address, which gives the dialed number.
- **Dual query and dedupe.** Two paged analytics queries for the week: one on the group's dialed number (inbound) and one on the members (everything they touched). Member conversations already counted as inbound, or not outbound, are dropped.
- **Classification.** Inbound calls use an outcome ladder: handled if a member talked, else voicemail, else a short drop under a time threshold, else abandoned. The handler is the member with talk time and the longest handle time. Rounding reproduces JavaScript `Math.round` so results match the old workflow.
- **Narrative.** Aggregates, daily and per-person breakdowns and a small sample of calls go to Claude in one message. If the call fails, the report goes out with "Analysis unavailable" in place of the narrative.
- **Parity check (now off).** During the parallel run the job could poll the old workflow's API for the same evening's execution, diff every conversation field by field, and email a PASS, FAIL or INCONCLUSIVE verdict. This was the cutover gate.

## Engineering decisions

- **Numbers in code, words from the model.** The model never computes a figure that appears in the report.
- **Graceful AI degradation.** A model failure falls back to a placeholder and the email still sends. This was the one approved behavior change from the old workflow.
- **Automated parity gate before cutover.** The cutover plan required clean nightly parity runs against the old workflow's real output before the new job took over.
- **Two time zones kept on purpose.** The send hour is Eastern and the report week is Central, matching how the original behaved, so the code keeps both explicitly.
- **In-tenant AI.** The model call goes to an Azure AI Foundry deployment through its Anthropic-compatible endpoint, and only call metadata for internal calls is sent. No audio, transcripts or customer data.

## Stack

Python, Genesys Cloud Analytics and Groups APIs, Azure AI Foundry (Claude), Azure Container Apps Jobs, Microsoft Graph mail, Key Vault, Managed Identity.

## Skills demonstrated

LLMs (large language models), Generative AI, Production systems, Reliability, APIs and API integration, QA and UAT testing, Security and compliance, Python

Platforms: Genesys Cloud, Azure AI Foundry, Claude, Azure Container Apps Jobs, Microsoft Graph, Key Vault, Managed Identity

## Timeline

- 2026-04-30: Create read-only API client for help desk reporting
- 2026-05-11: Rebuild help desk report with number and member queries
- 2026-06-12: Port help desk report to cloud job in parity mode
