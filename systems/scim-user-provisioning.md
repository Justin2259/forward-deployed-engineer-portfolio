# HR-driven phone system user provisioning

An HTTP API that creates, updates, deactivates and restores employees' Genesys Cloud accounts from HR lifecycle events, including their softphone, extension, roles and queue membership.

## Problem

New corporate employees, department moves and departures were handled by hand in the phone system: create the user, build a phone, pick a free extension from the department's range, assign roles, add call-center agents to queues, and send the person their phone details.

## What I built

A SCIM-style provisioning service that an integration platform calls when the HR system raises a join, change or leave event. For a new employee it creates the Genesys user, creates a WebRTC softphone and sets it as the default station, assigns the lowest free extension in the department's range, assigns roles, sets a starting voicemail PIN, applies department extras (queues and auto-answer for contact-center agents, groups and work teams for other departments), and sends a welcome email. A companion scheduled job emails a daily digest of what was created, failed or flagged for review.

The service is built, deployed and was validated end to end with real accounts through the integration platform. Production use is waiting on field mappings on the integration platform side.

## How it works

HR system -> integration platform -> provisioning API -> Genesys Cloud, with an activity log in storage -> daily digest job -> email

- **API.** Routes for create, update, terminate and re-enable, plus an unauthenticated health check. Requests carry a batch of records. Every route except health requires a bearer token checked against a Key Vault secret. Adding `dryRun=true` returns the same response without changing anything.
- **Department recipes.** A config module maps each HR record to a department from office location, company name, title and email domain, and holds each department's sites, roles, groups, queues and extension range.
- **Create.** Per record: pick the recipe, check the employee ID and email are not already in Genesys, allocate an extension, create the user, then roles, phone, PIN, department extras and welcome email. Each step is recorded, and one record's failure does not stop the batch.
- **Update.** Finds the user by employee ID, then email, updates name, title and email, and renames the phone. An office change that maps to a different department reconciles site, roles, groups, queues and extension, and sends a change notice.
- **Terminate and re-enable.** Terminate releases the extension and soft-deletes the user. Because Genesys keeps the email reserved, a returning employee is restored and then provisioned again.
- **Activity log and digest.** Each batch's per-record results go to a short-retention JSON log in Azure Storage. The digest job reads the last day of it and emails a summary. It holds no Genesys credential.

## Engineering decisions

- **One batch at a time on a single replica**, which is what makes "lowest free extension" allocation safe without a distributed lock.
- **Genesys is the system of record.** The service keeps no copy of the directory and reads current state on every batch.
- **A dedicated Genesys OAuth client** for directory writes, separate from the reporting jobs, so its permissions and rate limits are isolated.
- **Branch staff are out of scope by two independent checks** (office code and job title), so a broad HR feed cannot create accounts this service should not own.
- **Dry run on every write route**, so a caller can test a request and see the exact response with no side effects.
- **Its own Container Apps environment and Log Analytics workspace**, so this workload deploys independently of the others.

## Stack

Python (aiohttp), SCIM-style REST API, Genesys Cloud API, Azure Container Apps and Container Apps Jobs, Azure Storage, Key Vault, Managed Identity, Microsoft Graph mail, Bicep.

## Skills demonstrated

System design and architecture, APIs and API integration, Systems integration, End-to-end ownership, Implementation, QA and UAT testing, Security and compliance, Contact center (CCaaS), Python

Platforms: Genesys Cloud, Azure Container Apps, Azure Container Apps Jobs, Azure Storage, Key Vault, Managed Identity, Microsoft Graph, Bicep

## Timeline

- 2026-06-18: Design provisioning service and SOP-driven user lifecycle
- 2026-06-22: Deploy provisioning service and load secrets to vault
- 2026-06-23: Validate department-aware create, edit and terminate
- 2026-06-29: Pass full lifecycle test plan and switch off shadow mode
- 2026-06-30: Write provisioning session handoff and status audit
- 2026-07-13: Write real-account test cases for provisioning
