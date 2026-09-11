# Call investigation and routing change validation

Explains what happened on any call, classifies every call to a number over a date range, proves whether an IVR routing change lost calls or only moved them, and records how the production and development orgs are configured today.

## Problem

The same questions came up constantly. During testing: "I just made a test call, what did the caller hear and where did it go?" On escalations: "What happened on this specific call?" or "How is this location's line actually being answered?" After a routing change: "Contact-center volume dropped, did we lose calls?" Flow execution data, which shows what the caller heard, expires after a short platform retention window. Statements about how something was configured also tended to come from old notes rather than the live system.

## What I built

- **Test-call finder.** Finds the newest calls from or to a test phone number in the last few minutes and reports, per call, each flow with its version and how it exited, the caller-visible decision path (language, menus, hours, holiday, AI agent, queue), the queue, the outcome, and a link to the interaction.
- **Single-call deep dive.** Given a conversation ID, assembles the analytics record, participants, the full flow execution history, recording metadata and the SIP trace, and runs required checks for known failure patterns.
- **Number-over-a-date-range report.** Pulls every inbound call to a number, enriches each with the location's routing table rows, classifies it into one of nine dispositions, and renders an HTML report with a Claude-written narrative.
- **Routing change validator.** Compares a target day with a matched baseline day across calls arriving on the changed numbers, AI voice-agent queues, all other queues, answered calls, hang-ups and the flow-to-flow shift, and returns a HEALTHY or INVESTIGATE verdict in plain language.
- **Org inventory.** A dated, GET-only snapshot of the production and development orgs plus a diff between them.

## How it works

**Deep dive.** The script fetches the analytics detail record, the conversation object, flow instance execution data (prompts, DTMF, decisions, data-table lookups, data-action outputs, transfers), recordings, and downloads and parses the SIP trace PCAP. Agent IDs resolve to names. Three checks run on every call: a re-entry loop where the main IVR transfers to itself repeatedly after a hand-back from the AI voice agent because the number lookup fails (identified by a run of short consecutive flow instances that all end in transfer), a call that never disconnects (most often an agent left in pending after-call work), and decoding the SIP User-to-User header, the only place the AI agent's routing intent is recorded.

**Date-range report.** Dispositions are evaluated in strict priority order: answered by the contact center, answered at the location, short call, transferred out, voicemail recorded, after-hours voicemail, and abandoned in the ring group, the queue or the IVR. Internal callers (short extensions or internal transfer numbers) are surfaced separately. Large ranges use stratified sampling for the narrative so the prompt stays bounded while the statistics stay complete.

**Routing validator.** Both days use the identical clock window in Central time, with the offset computed from the US daylight saving rule. "Calls into the locations" counts arrivals by dialed number, so it is independent of which flow handled them. Queue counts come from conversation aggregates and flow counts from flow aggregates. HEALTHY requires three things at once: arrivals held, the hang-up rate did not rise, and total queue volume held. When those hold, a drop in contact-center volume is reported as deflection to the AI agent, self-service or the location, not as lost calls.

**Inventory.** One run captures every flow with its published version, IVR configurations and the numbers on each, schedules and schedule groups, queues, data tables with row counts and full rows of the routing tables, sites, groups, prompts, integrations, data actions and user counts. It writes Markdown summaries, detail CSVs and a dev-vs-prod diff, each stamped with the capture time and the client used. A staleness check flags a snapshot older than a week. For what a flow does inside, the contract points to an Archy export of the flow as YAML.

## Engineering decisions

- **Read-only by contract.** Every tool here only reads, so it runs immediately without a confirmation step.
- **Judge a routing change on arrivals and hang-ups, not one flow.** A source flow's count dropping is expected when its numbers move. Health is judged on calls arriving at the numbers plus the hang-up rate, and a forecast miss against pre-change routing is called out as the intended outcome.
- **Define buckets so they cannot silently miss queues.** The high-volume contact-center intake queues are not named after the contact center, so a name filter would miss them. The validator defines the contact-center bucket as all voice queues minus the AI agent queues, and excludes test queues everywhere.
- **Classify a queue by its membership, not its participants.** One conversation spans transfers, so a location desk phone can appear as an agent on a later leg of a contact-center call. Who staffs a queue is read from the queue's member list.
- **Handle platform limits explicitly.** Detail queries are capped near 31 days, flow execution data has short retention, and flow-instance jobs are chunked by ID. The deep-dive contract requires an explicit org flag, because querying the wrong org returns an empty report that looks the same as a call that never happened.
- **Cite the configuration, with a date.** Answers about how something is built name the object, its published version and the inventory capture time, and the snapshot is refreshed before any change.

## Stack

Python, Genesys Cloud API (analytics conversation details and aggregates, flow aggregates, flow instance execution data, conversations, recordings, SIP message PCAP, architect, data tables), Archy, Claude.

## Skills demonstrated

LLMs (large language models), Generative AI, AI voice agents, APIs and API integration, QA and UAT testing, Debugging and root-cause analysis, Documentation and playbooks, Customer experience (CX), Contact center (CCaaS), Measurable outcomes, Python

Platforms: Genesys Cloud, Archy, Claude

## Timeline

- 2026-04-01: Add call forensic report with IVR and agent detail
- 2026-04-08: Fetch and parse SIP traces in call investigation
- 2026-06-25: Move first wave of numbers between IVRs in safe batches
- 2026-07-08: Package call investigation as a self-contained skill
- 2026-07-30: Bulk-move numbers between IVRs and verify every route
- 2026-07-31: Add routing change validation by queue membership
- 2026-09-11: Capture and diff a restructured IVR against baseline
