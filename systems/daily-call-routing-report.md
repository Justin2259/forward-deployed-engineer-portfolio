# Daily call-routing diversion report

Emails contact-center leadership every morning with the list of branch locations whose calls are currently diverted to the central contact center.

## Problem

When a branch location cannot take calls, the phone team sets an outage flag in a Genesys Cloud data table, and the IVR sends that location's callers to the central contact center. Those callers land on contact-center agents, so leadership and workforce planning need to know each morning which locations are diverted. The old version of this report, a low-code workflow, stayed silent on days with nothing to report and stopped reading the table partway through on large tables.

## What I built

A scheduled Azure Container Apps Job that reads the whole routing table every morning, keeps the rows with the outage flag on, drops closed locations, and emails an HTML report from a shared support mailbox through Microsoft Graph. On a day with no diverted locations it still sends an "all clear".

## How it works

Genesys data table -> job -> filter and format -> email via Graph

- **Fetch.** One paged read of the data table rows, following the server's page count with no cap.
- **Build.** A pure function, testable offline: accepts the flag as a boolean or the string forms the table stores, extracts the location code with a regular expression, removes closed locations, sorts by name and formats the phone number.
- **Render and send.** A shared HTML card helper: a table of diverted locations, or a green all-clear callout. Sent with "save to Sent Items" on, so the mailbox's Sent Items is the archive.
- **Schedule.** Azure cron is UTC and ignores daylight saving, so the job fires at both possible UTC hours and the code runs only when the local Central hour matches. The other firing exits cleanly as "skipped".

## Engineering decisions

- **Always send.** The all-clear email doubles as the heartbeat: a missing email means the job failed, not "nothing to report".
- **No row cap.** Paging follows the server's page count, fixing the old version's truncation.
- **Parity mode** sends every email to one validation recipient with a prefixed subject, for validation before cutover. It also works as a soft off switch.
- **Fail loudly.** The mail helper returns a result instead of raising, and the job raises on failure so the execution shows Failed.
- **Read-only against the phone system**, with its own managed identity limited to reading its secrets from Key Vault.

## Stack

Python, Genesys Cloud Architect data tables API, Azure Container Apps Jobs, Microsoft Graph mail, Key Vault, Managed Identity.

## Skills demonstrated

Production systems, Reliability, APIs and API integration, QA and UAT testing, Debugging and root-cause analysis, Security and compliance, Contact center (CCaaS), Python

Platforms: Genesys Cloud, Azure Container Apps Jobs, Microsoft Graph, Key Vault, Managed Identity

## Timeline

- 2026-06-11: Pilot routing report as a scheduled cloud job
- 2026-06-12: Promote routing report to production after stability test
