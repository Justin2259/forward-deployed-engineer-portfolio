# Automation platform foundation

The shared Azure runtime, identity, secrets, state, source control and deploy path behind the phone team's unattended automation.

## Problem

The phone team's automation had grown up on a self-hosted low-code workflow tool and an interim web app on proof-of-concept infrastructure. Leadership directed that it move in-house onto Azure, with all code in the company's Azure DevOps. The team needed one place where small programs could run on a schedule or on demand, get their credentials without anyone pasting secrets, remember what they had already processed, and be rebuilt and rolled back from source.

## What I built

A single resource group that hosts the team's workloads as containers: two always-on apps (a Microsoft Teams assistant and a user-provisioning API) and a set of Container Apps Jobs that run on a schedule or on demand. They share one container registry, one Key Vault, one storage account for runtime state, and one Azure AI Foundry account. I migrated the existing workflows into Container Apps Jobs, checking each against the old tool's output before cutover, and put all code in one Azure DevOps repository.

## How it works

repository -> image build in the registry -> job or app with its own managed identity -> Key Vault references resolved at start -> run -> state in blob storage, logs in Log Analytics

- **Compute.** Two Container Apps environments, each with its own Log Analytics workspace for console logs.
- **Identity and secrets.** Every app and job runs under its own system-assigned managed identity with a Key Vault policy limited to reading secrets. Secrets reach the container as Key Vault references resolved when it starts, so nothing is in code, images, templates or logs. A rotated secret reaches a job on its next execution and an app on its next revision.
- **State.** A shared Python library provides a small document store with a file backend and a blob backend, plus a lock built on an infinite blob lease. Fixed leases are capped at 60 seconds, shorter than a speech-to-text backoff, so the lock uses an infinite lease stamped with its acquisition time; a lease older than the job's replica timeout is treated as orphaned and broken, and a second run exits cleanly.
- **Repository.** One folder per workload plus a shared library for Genesys Cloud, email through Microsoft Graph, Foundry and state. Bicep under `infra/` declares the bot, the provisioning app and its digest job with their environments and log workspaces. The `.gitignore` excludes environment files and any test fixtures containing customer transcripts, and staged code is scanned for secret patterns before commit. The main branch requires reviewers.
- **Deploy.** `azd deploy` builds the two apps remotely in the registry and releases a new revision. Each job is built with `az acr build` and released with `az containerapp job create` or `update`. Image tags are date-stamped, and rollback is an update to the previous tag.
- **Operations.** Instead of a separate alerting stack, the busiest jobs each have a heartbeat companion job that emails if the primary stops, and daily report jobs send an email every day whether or not there is news, so a missing email is itself the signal.

## Engineering decisions

- **One identity per workload, read-only on the vault.** A compromised job can read its secrets but cannot write or delete any.
- **Always-on only where a workload answers requests.** The Teams assistant and the provisioning API run continuously; everything else is a job that starts, does its work and exits.
- **A shared state and lock library**, because scheduled Container Apps Jobs have no forbid-concurrent setting and several jobs would otherwise double-process.
- **Never run `azd provision` against the live group,** because it resets the live Teams bot image. Jobs are created and updated individually with the CLI.
- **Skills become jobs.** Workflows are first built as Claude Code skills (deterministic Python behind a written contract) and proven on a laptop, then promoted into containers once their output matches.

Planned, not yet built: an Azure DevOps pipeline with a workload-identity-federation service connection and a gated production stage (the YAML is scaffolded), image pulls by managed identity for every job, Bicep for every resource so the whole group can be rebuilt from source, Azure Monitor alerting, and a move to a VNet-integrated environment with private endpoints.

## Stack

Azure Container Apps and Container Apps Jobs, Azure Container Registry, Key Vault, Managed Identity, Azure Blob Storage, Log Analytics, Azure AI Foundry, Bicep, azd, Azure CLI, Azure DevOps Repos, Python, Claude Code.

## Skills demonstrated

System design and architecture, Solution architecture, Production systems, Reliability, End-to-end ownership, Prototypes and proof of concept, Implementation, QA and UAT testing, Security and compliance, Python

Platforms: Azure Container Apps, Azure Container Apps Jobs, Azure Container Registry, Key Vault, Managed Identity, Azure Blob Storage, Log Analytics, Azure AI Foundry, Bicep, azd, Azure CLI, Azure DevOps Repos, Claude Code

## Timeline

- 2025-07-29: Create shared API client for team automation scripts
- 2026-03-31: Bootstrap web tools platform with run logging and dashboard
- 2026-04-30: Retire serverless host and consolidate the tools platform
- 2026-05-12: Design single resource group architecture for AI workloads
- 2026-06-10: Inventory workflow estate and shrink bloated workflow database
- 2026-06-17: Seed platform repo with IaC, Key Vault module and job scaffold
- 2026-06-18: Merge Key Vault secret references into the platform deploy
- 2026-06-30: Audit cloud footprint and publish shared state reference
- 2026-07-08: Retire legacy self-hosted tools platform
- 2026-09-29: Add job lease lock, shared clients and infra wiring to repo
