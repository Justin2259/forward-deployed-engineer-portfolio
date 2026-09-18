# Location telephony build-out pipeline

Stands up a new branch location in Genesys Cloud from one command, from the E911 address through IVR routing, users, phones and AI voice-agent enablement, and keeps that build correct as the location changes over its life.

## Problem

Bringing a branch location onto the phone system meant creating many related objects in a strict order: a location with an E911 address, a telephony site with caller ID and an outbound route, a ring group and a voicemail group, rows in several routing data tables that the IVR reads on every call, the IVR number assignment, external contacts, user accounts, phones, the AI voice-agent enablement row, and an entry in the phone team's tracking sheet. This used to be a written procedure. A step done by hand could be skipped without anyone noticing: one location built manually never had its routing rows or AI enablement finished, and a missing bot-menu row makes a location skip the caller menu and ring directly.

After the build, locations keep changing. Numbers get ported in and swapped, a location number gets reassigned to a different location, hardware phones arrive to replace softphones, and one location sometimes has to cover another's calls.

## What I built

An orchestrator that runs seven single-purpose step scripts in order and hands each step's IDs to the next. Every step is also its own tool for one-off work. Around it I built the rest of the location lifecycle:

- A bulk builder that runs the orchestrator once per location from a roster file.
- A re-key tool that moves a built location from one main number to another.
- A repurpose tool that rewrites an existing build in place when its location number is reassigned to a different location.
- A single-user creator for each employee type (corporate, branch, contact-center agent, IT service desk, and a separate business unit), each with its own site, extension pool, roles and groups.
- A hardware phone onboarding tool that turns a list of MAC addresses into working seats, and an in-place phone migration tool for model and hardware swaps.
- A call-forwarding tool that points one location's calls at another location and restores them later.
- A read-only verifier that checks a built location against the build checklist, and a site audit that finds sites parked on the wrong location.

## How it works

The build runs in this order, dry-run by default:

1. **Location.** Creates the address and assigns the E911 emergency number once the address verifies.
2. **Site.** Creates the telephony site with caller name, caller ID, time zone and firmware update window, then creates or reorders the default outbound route.
3. **Groups.** Creates the ring group and voicemail group with extensions derived from the location number, broadcast ring, and voicemail overflow back to the ring group.
4. **Routing data.** Writes the location row (keyed by main number), the internal transfer row plus the next free internal transfer number, the IVR assignment for both numbers (unassigning any conflicting IVR first), two external contacts, and the bot-menu row.
5. **Users and phones.** Creates numbered agent seats plus a voicemail user, each with an extension, roles, a WebRTC or hardware phone, station assignment, voicemail PIN and group membership.
6. **AI voice-agent enablement.** Writes the row that lets the AI voice agent offer to schedule an appointment for that location.
7. **Documentation.** Updates or inserts the location's row in the phone team's tracking sheet.

In dry-run mode the orchestrator finishes with a reference diff against a known-good location's live configuration. After an execute, the verifier walks every checklist item against the live org and exits non-zero on any failure.

Acquired locations whose official number is not yet in the platform are built on an interim number, with the site's outbound caller ID set to the official number so outbound calls still show it. On port day, the re-key tool updates every surface that carries the number: location E911, site caller ID, three routing table rows, the transfer row, the IVR number list, the external contact and the tracking sheet.

## Engineering decisions

- **Dry-run by default, idempotent everywhere.** Every script previews without `--execute`. Every step creates or reuses, so a failed chain resumes by re-running the same command. Users are matched by email and phones by name, and a result ledger is written after each user. The bulk builder runs locations one at a time with a stagger and keeps a resumable ledger to stay inside API rate limits. The phone migration tool goes further and requires the phone's UUID, printed by the dry run, as a confirmation token.
- **Validate against a known-good build before writing.** The reference diff reads a working location's live configuration and blocks the execute if the planned build differs on the routing-critical and AI-enablement fields. Time zone, name and number differences are reported, not failed.
- **Read back and diff after writing.** The repurpose tool snapshots every surface first, rewrites only the identity fields, re-reads, and exits 0 only if the diff is clean and the verifier passes. The snapshot is also the rollback source. The forwarding tool snapshots the row, writes, and reads it back.
- **A safe order for locked objects.** A location's E911 address cannot be edited in place, so the repurpose tool creates the new location, polls address verification, moves the emergency number, moves the site, then deletes the old location. A failure at any step leaves a consistent state: an unverified new location is removed, a failed number move is restored to the old location, and a site that will not move keeps the old location.
- **Guardrails written from incidents.** The IVR number update replaces the whole number list, so the scripts halt on a server error instead of retrying blindly. A location whose group extension reduces to 911 makes desk phones flag every inbound call as an emergency, so the group step hard-stops on it. The group step also warns when an extension is already owned by another resource. The duplicate-email check searches active, inactive and deleted users, because the platform keeps a deleted user's address reserved.
- **Documentation never blocks the build.** The tracking-sheet step warns and continues on an outage or rate limit, and never overwrites a status a person already set.

When an address will not verify, the build can park the site on the nearest verified location in the same state (by ZIP centroid distance) as a temporary stand-in so outbound routing works. A swap-back script moves it home once the address verifies, and the site audit lists every site still on a stand-in.

## Stack

Python, Genesys Cloud Platform API (locations, edges and sites, outbound routes, groups, users, phones, stations, data tables, IVR configs, external contacts), pgeocode, a tracking-sheet REST API.

## Skills demonstrated

System design and architecture, AI voice agents, Production systems, Reliability, At scale, APIs and API integration, End-to-end ownership, Implementation, QA and UAT testing, Documentation and playbooks, Python

Platforms: Genesys Cloud, tracking-sheet API

## Timeline

- 2026-04-27: Fix location user provisioning hang in chat tool
- 2026-05-22: Stop blind retries after a bulk number update left orphans
- 2026-07-07: Harden user and phone step; gate builds on real numbers
- 2026-07-08: Consolidate location buildout into one skill; ship two
- 2026-08-29: Bulk-build locations and map address verification limits
- 2026-08-31: Fix group extensions that collapsed to an emergency code
- 2026-09-02: Require verified address in location idempotency check
- 2026-09-18: Require voicemail user and agent on every build
