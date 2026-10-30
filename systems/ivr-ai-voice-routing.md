# IVR and AI voice agent routing

Decides which callers the main IVR sends to a vendor-hosted AI voice agent for appointment scheduling, hands them over with context over SIP, takes them back when they need a person, and adds a Spanish-language AI voice agent alongside the English one.

## Problem

The contact center added an AI voice agent, hosted by an AI voice-agent vendor, to book repair appointments by phone. The phone system had to decide on every call whether the caller should reach the AI voice agent or a person, pass enough context that the agent could greet the caller about their own repair, and take the call back cleanly when the agent handed off. The English routing was first built by a contact-center services vendor. When I took it over, there were open questions: why identical test calls sometimes reached the AI voice agent and sometimes a person, why locations that had opted out still sent some callers to it, and how to exclude a specific caller number, which no existing setting supported. On top of that the business wanted a Spanish-language AI voice agent.

## What I built

- A traced, documented model of every path into the AI voice agent, built from exports of the published flows and modules, which became the basis for fixes and for the later IVR cleanup.
- A fix to the location opt-out gate so a location that is turned off never sends any caller to the AI voice agent, in both shared routing modules (built and tested in dev, queued for the single production change request).
- Dedicated direct-to-agent demo lines with their own self-contained inbound flow, bridge queue and trunk, so demos never depend on live queue conditions.
- The Spanish-language AI voice agent routing, built internally in dev: its own SIP bridge queue and trunk, a staffed Spanish hand-back queue, a Spanish after-hours voicemail group, and a per-location pilot flag in the location table.
- A tiered test method for IVR changes, plus a test-harness data table that lets a tester's own phone number behave like a chosen customer.

## How it works

caller -> main IVR menu -> eligibility module -> wait-time gate -> zero-agent bridge queue -> SIP transfer with UUI -> AI voice agent -> (optional) SIP REFER back -> staffed hand-back queue or after-hours voicemail

- **Eligibility.** When a caller chooses the appointment option, a shared routing module looks the caller's number up in the repair-management system and buckets them by the age of their newest record against a lookback window held in a data table. New callers (no record) and current customers continue; returning customers with an old record bypass the AI voice agent and go to the human scheduling queue. Current customers must also pass criteria such as an enabled location, a supported insurance carrier and no open repair order.
- **Wait-time gate.** Before queueing, the IVR compares the scheduling queue's estimated wait against a threshold in a data table that workforce planning controls. Above the threshold the caller goes to the AI voice agent, otherwise straight to a person. This explained the "sometimes it works" test results: the outcome depended on live queue wait, not on the caller.
- **In-queue offer.** Callers already holding in the scheduling queue get one eligibility check on a timed pass of the in-queue loop. Eligible callers move to the AI voice agent before they hear the callback offer; ineligible callers get the callback offer on that same pass.
- **Hand-off over SIP.** The IVR transfers to a queue with no agents. Its in-queue flow sets a UUI header carrying the caller's number, a fixed signaling DNIS and the Genesys conversation ID, then transfers to the vendor's SIP endpoint. The vendor reads the rest of the context back from the conversation's participant attributes by conversation ID. If the SIP transfer fails, the call falls back to the main IVR.
- **Hand-back.** When the AI voice agent needs a person, it sends an in-dialog SIP REFER to a fixed address. The call stays the same conversation with its attributes intact, and a return flow routes it to a staffed queue for AI voice hand-backs, or to after-hours voicemail when the contact center is closed.
- **Spanish routing.** Spanish callers at eligible pilot locations go to a separate bridge queue whose cloned trunk points at the Spanish agent. Spanish hand-backs land in a staffed queue made up only of Spanish-skilled agents, and after hours in a Spanish voicemail group. The dev build reaches the Spanish agent end to end; the hand-back leg waits on the vendor enabling transfers on its test agent.
- **Testing.** Tier 1 is offline: compile, path check and a read-back of the published version. Tier 2 is one call per changed path plus a short smoke set, graded only from Genesys analytics and flow traces. Tier 3, the full conversational test bench with Whisper transcripts, runs only on approval before a change request.

## Engineering decisions

- **Separate Spanish bridge queue, trunk and hand-back queue** instead of sharing the English ones, so the English path and its reporting stay untouched.
- **Clone the trunk rather than edit it.** The existing trunk's DNIS override rewrites every outbound call to the English agent, so changing it would have redirected all English traffic.
- **Exclusion checks go inside both shared routing modules, just before the transfer decision.** Tracing showed several paths into the AI voice agent, some of which skip the IVR's global switch, so a check in the IVR alone would miss calls. Checks key on the raw call ANI, because some flows overwrite the caller-number attribute with a literal string.
- **The location opt-out applies to every customer bucket.** The original gate only checked current customers, so new callers at an opted-out location still reached the AI voice agent.
- **Self-contained demo lines.** The direct lines set the attributes the vendor needs as literals and use their own bridge queue, so demos are not subject to the wait-time gate or shared data tables.
- **Tiered testing.** A full bench after every step was too slow to iterate on. The basic tier marks any case that needs a spoken phrase as not covered, never as a pass.

## Stack

Genesys Cloud CX (Architect inbound and in-queue flows, common modules, data tables, analytics and audit APIs), SIP with UUI headers and REFER, Archy, Python, Whisper.

## Skills demonstrated

System design and architecture, AI voice agents, Conversational AI, Systems integration, Contact center (CCaaS), Customer experience (CX), Technical discovery and requirements, Debugging and root-cause analysis, QA and UAT testing, Implementation, Python

Platforms: Genesys Cloud CX, Architect, SIP, Archy, Python, Whisper

## Timeline

- 2026-07-29: Build on-demand test harness for AI voice routing
- 2026-08-07: Trace intermittent AI voice routing to a wait-time gate
- 2026-08-17: Document SIP bridge requirements for AI voice handoff
- 2026-09-21: Build a dedicated direct line flow to the AI voice agent
- 2026-09-28: Add demo lines with test caller identities
- 2026-10-12: Remove post-call survey from the AI hand-off flow
- 2026-10-19: Map every path into AI voice routing from flow exports
- 2026-10-24: Analyze a location opt-out bypass in AI voice routing
- 2026-10-26: Turn off language menu for one location; verify by call
- 2026-10-27: Plan in-house Spanish AI voice pilot routing
- 2026-10-28: Define Spanish pilot decision table and queue design
- 2026-10-29: Settle IVR cleanup handover and contact naming rules
- 2026-10-30: Build Spanish AI voice routing in dev and prove a call
