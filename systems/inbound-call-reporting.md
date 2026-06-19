# Inbound call and AI voice-agent reporting

A set of inbound call reports for branch locations, groups and IVR flows, including how callers reach an AI voice agent, plus the acceptance tests that keep a warehouse-fed location dashboard honest when call flows change.

## Problem

Several recurring questions needed numbers people could act on: how many calls a group or location handled in a week and by whom, how calls moved through an IVR flow and how many reached the AI voice agent, and how inbound callers narrowed down from the main menu to the agent each week. Separately, a business-intelligence dashboard showed per-location inbound outcomes from a vendor-operated data warehouse. Its query had been written against an earlier call-flow design, and as the flows changed (a new IVR rollout, an appointment gate in front of the AI agent, a revised voicemail flow) its metrics drifted wrong without anyone noticing.

## What I built

- **Group call report.** Calls handled by a group or location and its members for a week, as a self-contained HTML report with a Claude-written narrative, or as plain text for a quick answer in chat.
- **Flow report.** Every call through a chosen IVR flow over a date range, classified by disposition, whether it reached the AI voice agent, whether the agent contained it or escalated it, and the IVR path, rendered as HTML with day-by-day volume, anomaly days and a narrative.
- **AI agent funnel report.** A weekly five-step funnel for each IVR: calls into the menu, caller looked up in the repair-management system, matched to a repair, routed to the AI agent (split into gated business hours and ungated after hours), and came back. Each week carries a short note on what changed, and a panel shows where matched calls that did not reach the agent went.
- **Location dashboard rebuild and acceptance tests.** New metric definitions, a rebuilt warehouse SQL query with reference tables, a script that builds a ground-truth dataset from the Genesys Cloud analytics API, and a diff script that compares the warehouse output against it call by call.
- **IVR flow visualizer.** A small local Flask app that draws an IVR's menu structure as a clickable drill-down diagram.

## How it works

**Group and flow reports** fetch from the analytics API to a JSON file, then a second step renders HTML. Claude writes only the narrative section. The counts are computed by the scripts, and the text mode prints them with no model involved.

**Funnel report** is a five-step orchestrator. Calls into the menu, AI agent routing and returns come from Genesys Cloud analytics. The lookup and match steps come from a DynamoDB attribute store, because it keeps the per-call repair-lookup attributes that the platform deletes after a short retention window. The index builder parses both attribute formats the two IVRs write. Business versus after hours is computed from the main IVR's schedule group in Central time. Weekly change notes come from the flow version history and from week-over-week changes in the count of distinct numbers reaching each flow. A final step takes Playwright screenshots of the report for visual checking.

**Dashboard acceptance tests.** The ground-truth builder pulls every call for a chosen set of locations over a frozen window and classifies each into the report's buckets, extending the same classifier the call investigation tool uses. The reporting team runs the rebuilt SQL for the same window, and the diff script compares the two by conversation ID and requires each mismatch to be classified as a SQL bug, a definition gap or a timing artifact before the next run.

## Engineering decisions

- **Anchor metrics on what the caller experienced, not on flow structure.** The old query broke because it keyed on flow names, group names and the shape of the voicemail loop. The rebuild uses only the dialed number, whether a location endpoint alerted, human talk time, the voicemail deposit record, queue entry and external legs. Which number belongs to which location, and which queue belongs to which team, lives in reference tables.
- **"Voicemail left" means a message, not a mailbox visit.** A voicemail record exists for every mailbox visit, including callers who hang up during the greeting. A deposit counts only when its recorded audio meets a minimum duration held in one configuration row, and I calibrated the warehouse duration field against the platform's voicemail metric before relying on it.
- **Ship the drift alarm with the report.** One row per call, an "Other" bucket and a data-quality output mean a future flow change shows up as a rising "Other" count instead of silently wrong numbers, with an agreement gate on re-runs of the diff.
- **Reconcile before calling it done.** The funnel's AI agent count must equal the transfer queue's total for the window, each flow's funnel must narrow at every step, and the disposition panel must add back up to the matched total. After-hours calls bypass the match gate, so the report keeps the two paths separate instead of forcing a nested funnel.
- **Counts from code, prose from the model.** Claude only writes narrative. When a large range would overflow the prompt, it sees the first and last slices of calls, while the summary statistics stay complete.
- **Small traps written down.** Cross-segment filters need two segment filter entries because the queue and the flow live on different segments, and per-conversation batch loops use `continue` rather than `return` so an early match does not skip the rest of the batch and undercount.

## Stack

Python, Genesys Cloud analytics API (conversation details and aggregates, flow aggregates, flow versions), AWS DynamoDB (boto3), SQL Server, Power BI, Claude, Playwright, Flask.

## Skills demonstrated

Technical discovery and requirements, LLMs (large language models), Generative AI, AI voice agents, APIs and API integration, Systems integration, QA and UAT testing, Debugging and root-cause analysis, Customer experience (CX), Measurable outcomes, Python, AWS

Platforms: Genesys Cloud, AWS DynamoDB, SQL Server, Power BI, Claude, Playwright, Flask

## Timeline

- 2026-04-06: Add weekly group call summary report
- 2026-06-09: Inventory call flows and reconcile funnel from flow data
- 2026-06-19: Build weekly AI voice agent call-funnel report skill
