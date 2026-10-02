# On-call escalation hotline

A 24/7 phone line that forwards security alerts to whoever is on call and everything else to the team lead, built as a proof of concept in dev and promoted to production under change control.

## Problem

The security operations team needed one number that an outside alert-monitoring vendor could call at any hour and reach the person on call, while any other caller reached the team lead. The team rotates on-call coverage themselves and had to be able to change it without a telephony ticket. The calls needed to ring personal cell phones, not desk phones.

## What I built

- An Architect inbound flow, kept as Archy YAML, with no schedule and no menu.
- Three data tables: the alert vendor's caller numbers, a roster mapping each team member to their cell, and a single on-call row with one checkbox per member.
- A dedicated division and a role scoped to it, so team members can edit the on-call table and nothing else.
- An idempotent Python build script with an environment switch and a dry-run default, used for both the dev proof of concept and the production build.
- A follow-up caller ID fix: a dedicated trunk carrying the hotline's own number, plus a number plan and outbound route that send forwarded calls to the team's cells over it, with snapshot, apply, rollback and verify scripts.

## How it works

hotline -> inbound flow -> is the caller an alert number? -> yes: first checked member's cell / no: team lead's cell -> transfer fails: spoken message and disconnect

- Calls from an alert number go to the cell of the first checked member in the on-call row. If nobody is checked, they go to the team lead.
- Calls from anyone else go to the team lead.
- Every call is tagged with a participant attribute naming who it went to, or marking it as an escalation or unresolved, so the routing can be audited later.
- Forwarded calls leave on a dedicated trunk because the outbound route matches on the dialed cell number, so the handset shows the hotline's number.

The dev proof of concept went through several revisions with the team lead: it started with ring groups, overflow and a voicemail greeting, and was simplified to direct cell forwarding with missed calls going to personal voicemail. After a demo with the team, I filed the change request, built production in the approved window with the same script, ran the production test cases with the team, and closed the change.

## Engineering decisions

- **One row, one checkbox per member, first checked wins.** Data-table lookups key only on the row key, so a single row read with a nested condition is the simplest model the team can edit in the UI. The team lead's column is last, so a forgotten checkbox for the lead never overrides someone else.
- **A dedicated division and scoped role** let the team manage on-call themselves without broader admin rights.
- **A new trunk instead of changing the shared one.** Forwarded calls first showed the company's main number because they left on the default site's first trunk, which nearly every site uses. Changing that trunk's caller ID settings would have affected the whole org, so the fix was a dedicated trunk and a narrow route keyed on the team's cell numbers.
- **The original caller cannot be shown.** The voice carrier treats presenting an outside caller's number on a forwarded leg as spoofing, so the hotline's own number is what the team sees.
- **Verify on the handset, with a reliable proxy.** Analytics shows the original caller on the outbound leg, which is not what the phone displays. The conversations API's external participant address matched what the handset showed, so I used it as the check alongside confirmation from the receiving phone.
- **Snapshot, dry run, rollback, diff.** Every production change captured a before snapshot, ran as a dry run first, had a one-flag rollback, and was verified by diffing the read-back against the intended state.

## Stack

Genesys Cloud CX (Architect, data tables, divisions and roles, trunks, number plans, outbound routes, conversations and audit APIs), Archy, Python.

## Skills demonstrated

Prototypes and proof of concept, Implementation, Production systems, Reliability, Security and compliance, Debugging and root-cause analysis, QA and UAT testing, Stakeholder management, End-to-end ownership, Python

Platforms: Genesys Cloud CX, Architect, Archy, Python

## Timeline

- 2026-10-02: Build on-call escalation line proof of concept in dev
