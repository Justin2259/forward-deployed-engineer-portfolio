# Branch phone fleet health monitor

Polls the phone system for every desk phone's registration status, emails the team when a branch location goes completely dark or its phones start flapping, and sends a daily summary.

## Problem

A branch location whose phones have all gone offline (usually power, internet or hardware) or whose phones keep re-registering together (usually an internet link failing over and back) is invisible until a customer or employee calls to complain. Two older pollers in a low-code automation tool covered this, but they could die silently, which looked exactly like "all healthy". They also queried the phone system twice and leaked state for locations that no longer existed.

## What I built

One scheduled Azure Container Apps Job that replaces both pollers, plus a separate heartbeat job that watches the monitor itself. The monitor runs every ten minutes, detects outages and flapping from one snapshot, and sends color-coded HTML emails from a shared support mailbox through Microsoft Graph: red for an outage, amber for flapping, and a daily summary that goes out even when all is clear.

## How it works

Genesys Cloud (sites and phones with status) -> monitor job -> detectors -> email via Graph, with state in Blob Storage. Heartbeat job -> reads state -> emails if the monitor is stale.

- **Fetch once.** Each run pages the Genesys sites and phones endpoints (with status expanded) and normalizes the result. One snapshot feeds both detectors.
- **Outage detector.** A site is down when every phone is offline. It joins the alert list after a sustained down period. Recoveries are closed with a duration, the day rolls over at Central midnight, and sites removed from Genesys are pruned.
- **Flapping detector.** For a site with at least two operational phones, all of them re-registering within a short window of each other counts as one event. Several events inside a rolling hour, with a cooldown expired, makes the site an alert.
- **Send rules.** The outage email goes out on the top-of-the-hour run while any site is down. Flapping alerts go out immediately. The daily summary goes out once in the late-afternoon hour.
- **State.** One JSON document in Azure Blob Storage tracks site state, today's outages and flapping log, cooldown stamps and the last successful run.
- **Heartbeat.** A second job, same image with a different command, checks the last successful run time every thirty minutes and sends one "monitor not running" alert, throttled so it does not repeat.

## Engineering decisions

- **Stamp after send.** Cooldowns and "already sent" markers are written only after the email succeeds. The old pollers stamped at detection and burned the cooldown when an email failed. This was the one intentional change from the original logic.
- **Thresholds are constants in code, not environment variables**, so a mis-set variable cannot silence or over-fire the detectors. Recipients are environment variables so they change without a rebuild.
- **Pure detector functions with injected time**, covered by an offline self-test that pins threshold edges, day rollover, pruning, recovery and the cooldown-on-failed-send behavior.
- **A heartbeat job and an always-sent daily summary**, because silence from a monitor should never be read as "healthy".
- **Failures fail loudly.** Any fetch or email failure raises, so the execution shows Failed and state is not advanced. The next scheduled run is the retry.
- **Shadow mode** runs detection and persists state without email, so a shadow deploy previews production state exactly.

## Stack

Python, Genesys Cloud Telephony API, Azure Container Apps Jobs, Azure Blob Storage, Microsoft Graph mail, Key Vault, Managed Identity.

## Skills demonstrated

System design and architecture, Production systems, Reliability, APIs and API integration, QA and UAT testing, Python

Platforms: Genesys Cloud, Azure Container Apps Jobs, Azure Blob Storage, Microsoft Graph, Key Vault, Managed Identity

## Timeline

- 2026-06-15: Consolidate outage and SIP-flap alerts into one job
