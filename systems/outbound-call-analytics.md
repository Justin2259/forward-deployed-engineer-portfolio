# Outbound call analytics

Reports outbound call volume for staffing and per-location tracking, flags outbound calling patterns that look like a stuck phone or misuse, and ranks branch locations by unanswered outbound calls.

## Problem

Outbound questions came from several directions. The workforce-management system needed half-hour outbound queue numbers for intraday staffing that matched the platform's own exports exactly. An org-wide outbound count and a per-location breakdown were also needed. Then a failing desk phone redialed one external number over and over overnight, and nothing in normal reporting would have caught it. There was also the question of which locations had the most outbound calls go unanswered.

## What I built

Four read-only tools over Genesys Cloud analytics:

- **Interval report.** Half-hour rows for the outbound queue on a given date, midnight to midnight Central, as CSV.
- **Location summary.** Org-wide external outbound total over a window or a calendar month, plus a per-location table.
- **Anomaly scan.** A fleet-wide sweep of every outbound call over a window that ranks concentration on a single destination, not volume.
- **Not-answered report.** Ranks locations by the share of their outbound external calls where the agent leg never recorded talk time.

## How it works

**Interval report.** Uses conversation aggregates at 30-minute granularity with the UTC query interval aligned to Central midnight.

**Location summary.** Counts with the details query in 7-day chunks, requesting a page size of one and reading the total hit count. Internal calls are excluded by requiring an external or customer participant. Locations resolve through their phones (filtered by site) and their groups, and the contact center resolves by its site.

**Anomaly scan.** Runs an asynchronous analytics details job and processes each page as it streams, so memory stays flat across a month of company-wide calls. It produces two ranked views. CONCENTRATED: one or a few users calling one number many times, or a burst within a short window. MULTI-USER: many users calling the same external number, with a cluster flag when one user accounts for nearly all of it. Each finding carries tells that separate a machine loop from a person: the share of calls the far end hung up, how tightly talk times cluster, and the share placed overnight. The output is a ranked list and a JSON file, for a person to judge. A suspected stuck phone goes to the desk phone forensics tool to confirm the hardware.

**Not-answered report.** Classifies each outbound agent leg as answered or not by talk time, excludes transfer legs from the denominator, and joins each call to its location through an index of user ID to location group, built from groups and cached. A separate probe script was used once to test which signals were usable (wrap-up codes, disconnect type, duration, number joins) before settling on that classification.

## Engineering decisions

- **Flag concentration, not volume.** An agent calling hundreds of different customers is normal. One phone calling the same number many times is not. Ranking by concentration keeps normal high-volume dialing out of the results.
- **Use the right analytics API for each number.** Timing measures in the platform's totals are bucketed to the interval where the call ended, so averaging per-call detail drifts from the platform's export by seconds. The interval report uses aggregates only, and I validated it as an exact match against a stakeholder export.
- **Count without paginating.** Reading the total hit count with a page size of one avoids the details query's page cap, which a busy week of calls would exceed.
- **Derive the real destination.** On an outbound call, the external leg's caller number is the internal station address, which would masquerade as the destination. The anomaly scan uses the user leg's dialed number and dedupes by conversation per number, so a ring group with several user participants becomes one finding.
- **State what a metric cannot tell you.** "Not answered" is one bucket. It cannot separate a carrier decline from no answer from a voicemail greeting with no message, and outbound wrap-up codes are almost never filled in. The contract says so and forbids reporting a split from this data.
- **Work around search gaps.** Group search has indexing gaps, so group resolution falls back to a full scan. The phone filter must use the site ID parameter, because the similar-looking one is ignored and silently returns every phone in the org.

## Stack

Python, Genesys Cloud analytics API (conversation aggregates, details query, asynchronous details jobs), Genesys Cloud phones and groups APIs, CSV and JSON output.

## Skills demonstrated

System design and architecture, Technical discovery and requirements, At scale, APIs and API integration, QA and UAT testing, Debugging and root-cause analysis, Contact center (CCaaS), Python

Platforms: Genesys Cloud

## Timeline

- 2026-07-08: Package outbound, group and flow reports as skills
