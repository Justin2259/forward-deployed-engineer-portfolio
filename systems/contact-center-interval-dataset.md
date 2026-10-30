# Contact-center interval forecasting dataset

Produces half-hour queue history for the central contact center, per queue and language, with each call counted in the half hour it arrived, and proves every number against Genesys Cloud's own totals before anyone uses it for staffing forecasts.

## Problem

Workforce planning needed historical interval data to forecast staffing: offered, answered, service level, speed of answer, short and long abandons with wait ranges, handle, talk, hold and after-call work, transfers, outbound, and agent occupancy. Forecasting needs calls counted in the half hour they arrived. Genesys Cloud's interval totals bucket each measure by when it was emitted, which for timing measures is when the call ended, so the platform's own export does not give arrival-time numbers. A rebuilt dataset is only useful if the planner can trust it, and the planner also needed to refresh it on their own.

## What I built

A read-only pipeline in five stages (pull call records, pull agent status time, build monthly CSVs, write a Read Me from the definitions, verify) and a self-serve packaged tool the planner runs on a laptop with their own sign-in. Power Query templates load the output into the planner's Excel import workbook.

## How it works

- **Pull.** Call records come from the asynchronous analytics details jobs, one job per Central month, filtered to the contact-center queues. Each record is slimmed and stored gzipped, and the pull is resumable by day. The newest days, which jobs have not loaded yet, come from the synchronous details query, which I compared call by call against jobs output. Agent status time for occupancy comes from user aggregates at 30-minute granularity.
- **Build.** Each queue offer is placed in the Central half hour it arrived. Agent-side measures attach to the latest offer of the same queue at or before the agent leg started. Language is Spanish if any language attribute or Spanish queue appears on the call, otherwise English. Occupancy is interacting time over interacting plus idle, the same formula as the platform's status view, for agents who answered at least one contact-center call that month and were on queue.
- **Verify.** The verifier rebuilds the same data in completion mode and compares it with the platform's half-hour aggregates cell by cell, then checks the Read Me and, for a delivery run, a set of half-hour samples the planner exported from the platform UI. Every check must pass before anything ships.
- **Self-serve tool.** A zip with an embeddable Python runtime and no install. It signs in through the browser with OAuth Authorization Code and PKCE (no client secret, token held in memory only), finds the newest date in the data folder, rebuilds from the start of that month, runs the checks in a staging folder, and swaps files in only after every check passes. A cell off by one call goes to a review file and the run finishes. Larger differences stop the run.

## Engineering decisions

- **Rebuild in arrival mode, prove in completion mode.** Arrival time is what forecasting needs, but the platform's totals are in completion time. Building both from the same records lets every delivered number be checked against the platform's own figures.
- **One definitions file.** Queues, events, metric definitions and the service-level boundary live in one config that the build, the verifier and the Read Me all read, so a definition cannot change in one place and not the others.
- **Reviewed disagreements, not tolerances.** Where one call's platform detail record genuinely disagrees with the platform's own totals, the cell goes on a reviewed list only after the call is traced and approved. A listed cell passes only when both the platform value and the rebuilt value equal the recorded ones.
- **Read-only enforced in code.** Testing showed that the browser sign-in returned a token authorized for everything regardless of the scopes requested. The tool's HTTP layer keeps an allowlist and refuses any call outside it before sending.
- **Time handled by rule, not offset.** Days are built with time-zone data, so daylight saving days get the right number of half hours, and a test fails if a fixed offset appears in the code. Today is never marked as pulled, because that would lose every later call. Measures the platform adds a day or more after a call are held as "waiting" on the newest days and fully checked once they age past that window.
- **Back off on result limits.** A month of half hours for a full user batch exceeds the user aggregates result limit, so the occupancy pull halves the batch and retries.

## Stack

Python, Genesys Cloud analytics API (conversation details jobs and query, conversation aggregates, user aggregates), OAuth Authorization Code with PKCE, embeddable Python, Excel Power Query.

## Skills demonstrated

System design and architecture, Technical discovery and requirements, APIs and API integration, End-to-end ownership, QA and UAT testing, Security and compliance, Documentation and playbooks, Stakeholder management, Contact center (CCaaS), Python

Platforms: Genesys Cloud, Excel Power Query

## Timeline

- 2026-10-21: Scope interval dataset and confirm history reach
- 2026-10-22: Match service-level settings and time zone to UI exports
- 2026-10-23: Verify language tags from flow traces; plan readme tabs
- 2026-10-29: Build interval dataset skill with raw store and verify
- 2026-10-30: Verify every dataset cell against platform aggregates
