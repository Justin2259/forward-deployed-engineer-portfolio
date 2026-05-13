# Telephony operations assistant in Microsoft Teams

A conversational AI bot in Microsoft Teams that lets the phone team ask about phone health, individual calls and call reports in plain language, and reroute a branch location's calls during an outage, without opening the phone system's admin screens.

## Problem

Answering questions like "is this branch's phone down?" or "what happened on this call?" meant logging into the Genesys Cloud admin console and clicking through several screens. Rerouting a branch's inbound calls to the central contact center during an outage was another manual admin task.

## What I built

An always-on bot service in Azure that the phone team chats with in Microsoft Teams. It answers common questions directly from Genesys data and hands everything else to Claude, which picks from a catalog of Genesys lookup tools and writes the reply. The bot can make exactly one change: flip the outage flag that reroutes a branch's calls to the central contact center and back. That change is always previewed and waits for a typed "yes" before it runs.

## How it works

Teams -> Azure Bot Service -> bot container -> allowlist check -> fast path or Claude tool loop -> Genesys Cloud -> reply in Teams

- **Gate.** Every message is checked against an allowlist of Entra ID object IDs. Non-Teams channels carry no Entra identity, so they are refused.
- **Fast paths.** Regular expressions catch the most common requests (a pasted call ID or Genesys link, a "phones down" query, a dialed-number report, a group call report) and call Genesys directly. A second Claude call turns the raw result into readable text, with deterministic formatters as the fallback.
- **Claude tool loop.** Anything else goes to Claude with a system prompt, the tool schemas and the recent conversation. While Claude asks for tools, the bot runs them in a thread pool and feeds the JSON back, up to a fixed iteration cap. Tools cover phone health per location or across the fleet, a single call's full story (IVR path and SIP signalling), calls to or from a number, and call reports.
- **Staged writes.** The one write tool is never executed by the model. The bot intercepts it, stages it with a short expiry, and tells Claude to show the user a preview. Only a typed "yes" runs it. A confirmed change that actually altered state sends an alert email from a shared support mailbox through Microsoft Graph, which also serves as an audit trail.
- **Hosting.** Azure Container Apps with a single replica, secrets as Key Vault references resolved by a system-assigned managed identity, image pulled from Azure Container Registry by that identity, logs in Log Analytics.

## Engineering decisions

- **Model calls stay in the tenant.** Claude runs on an Azure AI Foundry deployment through its Anthropic-compatible endpoint, so call data sent to the model never leaves the Microsoft tenant.
- **The model can propose a write but not perform it.** Write tools are intercepted in code and require a human "yes", so a model mistake cannot reroute calls.
- **Routing requests skip the fast paths.** Any message that looks like a routing intent is forced through the Claude loop and the confirm gate, so a write never takes a shortcut.
- **Fast paths for the frequent questions.** Common lookups call Genesys directly instead of waiting for the model to choose a tool. A bare location code deliberately has no fast path, so the bot asks what the user wants instead of guessing.
- **Prompt caching** on the system prompt and tool list, which are the same on every turn.
- **Single replica on purpose.** Conversation memory and staged writes live in process, so the app is pinned to one replica.

## Stack

Python (aiohttp, Bot Framework SDK), Microsoft Teams, Azure Bot Service, Azure Container Apps, Azure AI Foundry (Claude), Genesys Cloud API, Microsoft Graph, Entra ID, Key Vault, Managed Identity, Azure Storage, Bicep.

## Skills demonstrated

Solution architecture, LLMs (large language models), Generative AI, AI agents, Agentic AI, Conversational AI, Function calling and tool use, Production systems, APIs and API integration, Security and compliance, Contact center (CCaaS), Python

Platforms: Microsoft Teams, Azure Bot Service, Azure Container Apps, Azure AI Foundry, Claude, Genesys Cloud, Microsoft Graph, Entra ID, Key Vault, Managed Identity, Azure Storage, Bicep

## Timeline

- 2026-05-12: Add hosted agent and workflow for phone lookups
- 2026-05-13: Ship hosted assistant agent and swap its model
