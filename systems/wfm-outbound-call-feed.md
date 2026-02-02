# Outbound call feed to workforce management

Sends every outbound call from the central contact center to the workforce-management system so it can forecast outbound work and build agent schedules.

## Problem

The workforce-management system needs a record of each outbound call: when it started, whether it connected, and how long the agent spent on it. That feed ran as a workflow in a self-hosted low-code automation tool the team was retiring. It also dropped calls that ended after its last run of the day.

## What I built

Two scheduled Azure Container Apps Jobs from one image. The sync job runs every fifteen minutes and re-sends the whole current day. A reconciliation job runs once just after midnight Central and re-sends the full previous day, which closes the end-of-day gap the old workflow had. I ported the transform so the numbers the vendor sees did not change.

## How it works

Genesys Cloud analytics -> sync job -> transform -> vendor contact-events API

- **Fetch.** One paged Genesys analytics conversation-details query for a single Central day, filtered to the outbound queue.
- **Transform.** A pure module with no I/O turns each conversation into a contact event: unique ID (the Genesys conversation ID), timestamp, offered, handled and handle seconds. It reproduces the old workflow's rules exactly, including JavaScript `Math.round` half-up rounding and integer output when whole, so the JSON matches the original's serialization. An empty day still posts one empty batch.
- **Deliver.** Batches are posted to the vendor's API with a bearer token from Key Vault, with a short timeout and a few retries. A failed batch raises, so the execution shows Failed.
- **Reconciliation.** The same image with a flag set. Azure cron is UTC and ignores daylight saving, so the job fires at both possible UTC hours and the code proceeds only when the local Central hour matches, exiting cleanly as "skipped" on the other.

## Engineering decisions

- **Stateless full-day resend.** No cursor, lock or state store. Every event carries the conversation ID and the vendor upserts on it, so resending the whole day every run is safe and a failed run loses nothing once a later run succeeds.
- **Bug-for-bug parity with the old workflow.** I verified it by running the original JavaScript in a V8 engine against real multi-day Genesys data and comparing outputs. The module documents each quirk and says not to "fix" it.
- **Shadow mode is the default in code**, so a job definition that forgets the flag computes and logs but never posts.
- **A dedicated Genesys OAuth client** for this feed, with the shared reporting client only as a fallback, and the run summary logs which client authenticated.
- **Retry limit zero in Azure.** The next quarter-hour run is the retry.
- **Minimal outbound payload.** Only call metadata leaves the tenant: no phone numbers, names, audio or transcripts.

## Stack

Python, Genesys Cloud Analytics API, Azure Container Apps Jobs, Key Vault, Managed Identity, Azure Container Registry, REST integration with a workforce-management system.

## Skills demonstrated

Production systems, Reliability, APIs and API integration, Systems integration, QA and UAT testing, Security and compliance, Contact center (CCaaS), Python

Platforms: Genesys Cloud, Azure Container Apps Jobs, Key Vault, Managed Identity, Azure Container Registry, workforce-management system

## Timeline

- 2026-01-14: Create least-privilege API client for outbound feed
- 2026-02-02: Create production API client for outbound feed
