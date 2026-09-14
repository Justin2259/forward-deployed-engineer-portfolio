# Call recording policy and compliance

Recording and retention for the IT service teams' calls, plus an audit of who viewed which recordings.

## Problem

The IT service desk and related IT teams needed their calls recorded and kept for a set period. Most of their calls never touch a queue: they are direct calls to a person or to a ring group, and many are internal. Early checks found no recordings for these calls even though recording was on at the trunk.

## What I built

- A retention policy covering inbound and outbound calls for the IT teams, with an archive step and a delete schedule.
- Work teams for the IT teams, used as the policy's target.
- Station-side recording turned on for each phone on those teams, set per phone.
- Role permission changes so the automation client could manage teams and retention policies, which the org's trimmed admin role could not.
- Separately, an audit of recording access: who opened which recordings, from the Genesys audit API.

## How it works

call -> trunk-side and station-side recording -> retention policy matches the user's work team -> retain, archive, then delete on schedule

- **Why recordings were missing.** The org has a catch-all policy that deletes recordings of any call that never reaches a queue. Those calls are recorded at the trunk and then purged, so the recordings API returns not found even though recording happened.
- **Retain policy.** A targeted retain policy wins over the catch-all, because the catch-all uses a plain delete action rather than an always-delete. The new policy keys on the IT work teams.
- **Internal calls.** Trunk-side recording only sees calls that cross the trunk, so user-to-user calls are never captured there. Station-side recording on each team phone captures them. External calls then produce two recordings (trunk and station) and internal calls produce one.
- **Access audit.** The audit pull counts distinct recordings opened per viewer, not raw read events, separates self-review from review of other people's calls, and attributes viewers from a partner organization through the trustee filter.

## Engineering decisions

- **Add a retain policy, leave the catch-all alone.** A targeted policy outranks the catch-all, so the change touches only the calls in scope and the rest of the org behaves as before.
- **Work teams as the target.** Recording policies cannot target a group, only users, work teams, queues, wrap-up codes, direction, time or duration. Work teams map cleanly because each user can belong to only one.
- **Station-side recording per phone, never on the shared phone base.** Most of the org's softphones inherit one shared base, so setting it there would have recorded far more people than intended.
- **Distinct recordings, not read events, in the access audit.** One recording generates several read events each time it is opened, replayed or scrubbed, so raw event counts overstate activity badly.
- **Self-review split out first.** An agent replaying their own call is not oversight of others, and leaving it in distorted every concentration signal.

One gap remains and is documented: new hires on one of the IT teams are provisioned onto the right work team automatically, while the others still need a manual step.

## Stack

Genesys Cloud CX (recording policies, retention, work teams, station line properties, recordings API, audit API, authorization roles), Python.

## Skills demonstrated

Security and compliance, Debugging and root-cause analysis, Technical discovery and requirements, Implementation, Production systems, APIs and API integration, Contact center (CCaaS), Documentation and playbooks, Python

Platforms: Genesys Cloud CX, Python

## Timeline

- 2026-09-11: Enable recording policy for a help desk work team
- 2026-09-14: Extend call recording to two more IT teams
