# Desk phone diagnostics and remediation

Finds out why a branch location's desk phone is misbehaving, from a single pointer such as an extension or a station name, and fixes the configuration defects behind it with dry-run-gated, one-at-a-time writes.

## Problem

Desk phone tickets arrive as symptoms: a phone placing calls nobody dialed, a phone that never rings, one location's calls ringing at another location, a phone sitting offline. The cause can be hardware, configuration on the phone itself, configuration in Genesys Cloud, or leftovers from a phone or user being moved between locations without every related object being updated. The investigation that started this work, a phone repeatedly redialing one external number overnight while the location was empty, took hours by hand. The goal was to make the next one take minutes.

## What I built

Four tools that split diagnosis from repair:

- **Phone forensics (read-only).** Identifies the device in Genesys Cloud, rules out software causes, then logs into the phone's own web utility over the corporate VPN to pull device info, application and boot logs, and the configuration export, and returns a HEALTHY, SUSPECT or FAULTY verdict with evidence.
- **Line audit (read-only).** For one location: every phone's line appearance and live SIP address of record, whether each phone's line label matches its user's directory extension, and the audit trail of who changed extensions or line appearances, with old and new values.
- **Identity audit and remediation.** Checks a user and phone against the location they should belong to (user name, email, extension, line label, station name, E911 site, group memberships) and produces a numbered fix plan with a ready-to-run command per fix.
- **Phone operations.** Reboots a location's phones after a health assessment, and resets a user's voicemail PIN by extension.

## How it works

**Forensics.** The script walks pointer -> user -> station -> phone to get the model, MAC, edge, registration state and the phone's LAN address from its live line status. It scans phone and base settings for hotline, off-hook auto-dial, speed-dial and forwarding keys, and greps for a suspected target number. It probes the phone's HTTPS port over the VPN, then logs in. The login method depends on firmware, not model: the script reads the phone's own login script to choose between Basic and SHA-256 Digest, because sending the wrong one is rejected even with the right password. It pulls logs and the effective configuration export using the anti-CSRF token as a query parameter, which those endpoints require. The diagnosis step looks for known signatures:

- A failing keypad shows as a continuous flood of key-bounce messages in the application log, where a healthy idle phone logs none. A shorting key re-triggers redial of the last number.
- A forward set from the handset's own Forward key exists only in the phone-local configuration export, never in the platform's user forwarding. In analytics, every inbound alert to that user ends almost instantly with a SIP 302.
- A phone that never provisions, rejects both factory and fleet passwords, and answers a plain UDP SIP OPTIONS (which platform-provisioned phones, TLS only, do not) is carrying configuration from outside the org and needs a factory reset on site.

**Line audit.** It resolves a location's sites, phones, stations, groups and members across their different naming prefixes, joins users to phones through their effective station, and flags two phones reporting the same SIP address of record. That duplicate registration leaves one phone offline, and a label comparison alone does not reveal it. The history view queries the platform audit service one entity type at a time in 30-day chunks with cursor paging.

**Identity remediation.** The headline defect is a station still in another location's ring group, which makes that location's inbound calls ring the wrong building. Fix actions are graded by risk (group changes are safe, renames and extension or line-label changes are medium, an E911 site change is high because the phone re-provisions). Each runs alone, previews first, re-fetches the object version before writing, reads the value back after, and never deletes a user, phone, site or group.

**Phone operations.** The reboot pipeline lists a site's phones with status, flags each as offline, unregistered, stale or provisioning error, optionally asks Claude to assess which need a reboot and why, and reboots only after a person confirms. The PIN reset dry-run shows the matched user's name and email before the write, then prints a ready-to-send email for the requester.

## Engineering decisions

- **Read-only diagnosis, separate write tools.** Forensics and the line audit never write to the phone or the platform. Fixes live in tools that dry-run and confirm. The handset-forward fix is documented as a write that needs approval, not folded into forensics.
- **Lockout-safe login.** The phone locks its admin account after repeated failures, so forensics tries exactly one password. The password comes from the environment and is never printed or logged.
- **All phones offline means do not reboot.** If every phone at a site is offline at once, the cause is almost always the network or the SIP trunk. The reboot pipeline flags that as critical and recommends zero reboots.
- **Prove physical location before trusting names.** A user named for one location whose station, site, ring group and network address all point to another is a mislabeled seat, not a foreign phone. Acting on the name would remove a working phone from its own ring group, so the contract requires confirming location from the phone's network address first, then renaming the user.
- **An error is never an empty result.** A single page of a tenant-wide list silently dropped locations, so every list call paginates. The platform ignores one site filter parameter and honors another, so the scripts filter server-side with the right one and keep a client-side check. An oversized audit query once returned an error that was read as zero events; any non-200 now exits with an error code.
- **Exit codes built for sweeps.** The audits return 0 for consistent, 1 for remediation needed and 2 for errors, with JSON output, so they can run across a batch of tickets.

## Stack

Python, Genesys Cloud API (users, stations, phones, groups, audits, analytics), Poly and Polycom VVX and Edge web configuration utility over VPN, SIP, Claude.

## Skills demonstrated

System design and architecture, LLMs (large language models), APIs and API integration, Debugging and root-cause analysis, Security and compliance, Measurable outcomes, Python

Platforms: Genesys Cloud, Poly and Polycom VVX, Claude

## Timeline

- 2026-07-08: Diagnose phantom auto-dialing from a failing keypad
- 2026-07-30: Trace wrong-location ringing to phone identity drift
- 2026-07-31: Fix group busy tone caused by wrong telephony site
- 2026-08-17: Diagnose single-phone ringing from SIP response codes
- 2026-08-24: Detect duplicate SIP registrations behind line mismatches
