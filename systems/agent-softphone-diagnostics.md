# Agent softphone sign-out diagnostics

A repeatable method for "the softphone keeps logging me out" tickets that separates real sign-outs from dropped browser connections, using the audit trail, presence history and a peer control group.

## Problem

Contact-center agents on the Genesys Cloud browser softphone reported being logged out during their shifts, and later a group of agents reported it after the company moved to a managed enterprise browser. The audit trail alone looked clean, with one sign-in per shift, which would have closed the tickets as "no issue". The agents were still losing calls because they were no longer on queue.

## What I built

- A read-only Python method that pulls two sources per agent and lines them up: authentication and presence events from the audit API, and presence segments from the analytics user details API.
- A peer control: the same pull for the agent's queue-mates on the same days, to tell an endpoint problem from a platform problem.
- A fleet comparison and a before/after harness with a frozen baseline file that refuses to be overwritten, plus per-agent timeline and sign-ins-per-day tools for single cases.

## How it works

audit sign-in events + presence segments + queue-mate control -> classify each drop -> endpoint or platform

- **Real sign-ins.** An authenticate event paired with an access-token creation is a real login. One pair per shift and no logout or token-deletion events means the session token is not being killed.
- **Drops.** An on-queue segment followed by an offline segment during working hours that comes back within a few hours, with no new sign-in, means the client disconnected and reconnected on a still-valid token. Agents land back on Available and have to re-select On Queue, which they describe as being logged out.
- **Who changed presence.** Presence audit events split two ways: on-queue to offline with an empty actor means the platform saw the client connection die; on-queue to Available with the agent's own user and IP means their client, or a second stale tab, published Available.
- **Org settings ruled out.** Idle token timeout and universal logout were checked and were not the cause.
- **Control.** If queue-mates show a few drops and the reporting agent shows many, the cause is the agent's endpoint.

The browser-side fix (keep the Genesys web app's tab always active and turn off memory saving) helped the first agent noticeably. Another agent kept dropping after the same change and showed a different pattern, a full new sign-in some minutes after each drop, which means the page itself was being unloaded. Genesys publishes a supported browser list and a recommendation to disable browser sleep, and the managed browser is not on that list, so the interim workaround was a supported browser while the managed-browser side was investigated.

## Engineering decisions

- **Two sources, always.** The audit trail answers "was the session killed"; the presence timeline is where the symptom shows. Either alone gives the wrong answer.
- **A peer control group** turns "the platform is broken" versus "this workstation is broken" into a measurement.
- **Page through every result.** The analytics user details query pages by segments, not users, so a multi-user query returns only a few users per page. Merging all pages by user fixed an earlier undercount in my own fleet table.
- **Freeze the baseline.** The before/after harness writes its baseline once and refuses to overwrite it, so a fix is judged against each agent's own pre-fix rate.
- **Ask for a screenshot early.** "Logged out" from an agent usually means the On Queue toggle moved, which changes which data to pull.

## Stack

Python, Genesys Cloud audit API, Genesys Cloud analytics user details API, Genesys Cloud routing and users APIs.

## Skills demonstrated

Debugging and root-cause analysis, APIs and API integration, Reliability, Contact center (CCaaS), Technical discovery and requirements, Documentation and playbooks, Stakeholder management, Python

Platforms: Genesys Cloud CX, Python

## Timeline

- 2026-09-15: Correlate login audit with presence to explain sign-outs
- 2026-09-16: Trace agent off-queue flips to sleeping browser tabs
