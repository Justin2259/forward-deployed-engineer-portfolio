# Branded caller ID onboarding

Adds a branch location's outbound caller ID to a branded caller ID provider's program, so customers see the company name instead of a spam label, in three phases with a human decision after each.

## Problem

Locations reported that their outbound calls showed up on customers' phones as spam. Branding a number with the provider meant pricing it, building a program that matches the ones already live, and attaching the number. The provider's API also reports a program as active even when branding is off, so "it's done" could not be taken at face value.

## What I built

One script with three subcommands: `quote`, `plan` and `apply`. Each of the first two ends in a hard stop for a person's decision.

## How it works

- **Quote (read-only).** Resolves the location, reads its Genesys Cloud site caller ID, checks that number's status with the provider, counts external outbound calls by the location group's current members over a trailing window (split into two halves to show the trend), and prices it from a contract-rates file. It exits early, with the reason, if the location is already branded, has no caller ID, or its caller ID is toll-free.
- **Plan (dry run).** Derives a template from every live location program by taking the majority value of each field, and blocks if there are too few programs to trust or they disagree. It lists outliers it will not copy, runs preflight checks (caller ID present and not toll-free, registered and approved with the provider, not already in another program, the template display name approved), and prints the exact four requests it would send.
- **Apply (production write).** Requires an explicit confirm flag and a number that must equal the caller ID the plan used. It re-runs every preflight check, then creates the program, sets its schedule, attaches the approved display name, and attaches the number, which is the go-live step. It is idempotent and resumes a half-built program.
- **Verify.** After apply, it checks that the new program's fingerprint matches the template field by field, that the number is approved and lists the carrier networks, and runs control checks: every other program's number count is unchanged, the program count rose by one, and unassigned numbers dropped by one.

## Engineering decisions

- **Phase gates with hard stops.** Cost is approved before planning, and the plan is approved before anything is written. The apply command is printed by a READY plan rather than composed by hand.
- **Derive the template from production, not from memory.** Copying the majority configuration of every live program keeps a new program identical to the ones that already work, and a disagreement among them blocks instead of guessing.
- **Bind the write to what was reviewed.** Apply refuses a number that differs from the planned caller ID, so a site caller ID change between plan and apply cannot slip through.
- **Positive and control verification.** The new program is checked against the template, and the rest of the account is checked to prove nothing else moved.
- **Do not trust the API's "active".** The contract says only the provider's display results showing the brand on every major carrier prove the number is live, and treats a number still flagged after about a week as a defect to escalate, not one still settling.
- **Block caller IDs that cause the problem.** Toll-free and inbound IVR numbers used as outbound caller ID get scored as spam, so the script refuses them and points to fixing the caller ID in Genesys Cloud instead.

The contract also covers why a location can still show as spam after branding: calls placed through a queue present the queue's caller ID instead of the site's, and a phone assigned to a different site presents that site's caller ID. Both are checked from the members' outbound conversation details, not from the analytics caller number, which holds the internal extension.

## Stack

Python, branded caller ID provider REST API, Genesys Cloud API (sites, groups, analytics conversation details).

## Skills demonstrated

System design and architecture, Production systems, APIs and API integration, Systems integration, Implementation, QA and UAT testing, Debugging and root-cause analysis, Customer experience (CX), Python

Platforms: Genesys Cloud, branded caller ID provider API

## Timeline

- 2026-07-08: Build read-only branded-calling inventory client
- 2026-07-09: Sweep branded-calling API for programs and numbers
- 2026-07-16: Baseline caller-ID validation from carrier export
- 2026-07-29: Trace spam labels to toll-free numbers used as caller ID
- 2026-09-08: Model branding pricing tiers and phone-side features
- 2026-09-15: Map portal backend and correct the branding lapse timeline
- 2026-09-28: Audit outbound caller ID config and carrier rejections
- 2026-09-29: Reproduce branding programs through the API
- 2026-09-30: Onboard a location to branded calling with a new skill
- 2026-10-01: Onboard another location with quote and preflight checks
