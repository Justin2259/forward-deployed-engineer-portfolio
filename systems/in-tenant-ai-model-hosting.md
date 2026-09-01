# In-tenant AI model hosting

Runs the Claude and Whisper models that the team's automation uses inside the company's own Azure tenant, so prompts, call audio and transcripts never go to a public API.

## Problem

Several workloads needed language and speech models: a Teams assistant for the phone team, an AI call grader, a weekly report narrative, and a voicemail transcriber. Much of what they process is customer call audio and internal call-out messages. Sending that to public vendor APIs was not acceptable, and each workload needed a consistent, keyless way to authenticate.

## What I built

One Azure AI Foundry account with separate model deployments: a production Claude deployment for the workloads, separate Claude deployments for interactive developer tooling, and a Whisper deployment for speech-to-text. I wrote the client code the jobs and the Teams bot use to call it, a scripted per-user access grant, and a usage report per deployment.

## How it works

workload managed identity -> Key Vault secret -> Entra token for the right audience -> Foundry deployment

- **Programs.** Each workload runs under its own managed identity, reads a service principal secret from Key Vault, exchanges it for a bearer token with the audience of the surface it is about to call, and presents that token to Foundry. No API key is used anywhere.
- **Claude.** Jobs post to the Anthropic-compatible Messages endpoint on Foundry with plain HTTP and the deployment name as the model. The Teams bot uses an async twin with `azure-identity` and prompt caching, with a cache marker on the last system block so the tool definitions are cached too. Usage counters, including cache reads and writes, feed the grader's cost ledger.
- **Whisper.** The transcriber and the grader post multipart audio to the OpenAI-compatible transcription endpoint on the same account. The grader asks for segment timestamps so it can realign text onto speaker labels, paces requests across worker threads, and collapses repetition loops. Both back off on throttling.
- **People.** A developer's Claude Code session can route to the developer deployments with the person's own Entra login. A skill grants the invoke-only role and hands over a setup script. The setting is machine-scoped on purpose, so a checked-in workspace file cannot redirect which account a tool authenticates as.
- **Usage.** A script reads token volume per deployment from Azure Monitor and separates interactive desk usage from production load.

## Engineering decisions

- **Foundry over public APIs** because data stays in the tenant and is not used for training.
- **Two auth audiences, pinned in code.** The Claude surface and the Whisper surface need different token audiences, and the wrong one on the Claude surface returns a silent empty-body 403. Both clients pin their audience and warn against "correcting" it, after a well-meant unification once broke one of them.
- **No vendor SDK in the jobs.** The Anthropic SDK assumes API-key auth, so the jobs use plain `requests` with bearer tokens.
- **New models ship as new deployments.** The production deployment is never edited; a consumer moves by changing one environment variable, which is also the rollback.
- **A simpler architecture won.** The first design was a Foundry-hosted agent exposing tools through the Responses API. Once the Teams bot called the model directly with its own tool catalog, the hosted agent was a second copy of the same tools, so I retired it and kept the code for reference.
- **Deploying Anthropic models needed a workaround.** The CLI has no flag for the required Marketplace attestation, so deployments go through a raw ARM PUT on a preview API version that accepts the field.

## Stack

Azure AI Foundry, Claude (Anthropic Messages API), Whisper, Microsoft Entra ID, Key Vault, Managed Identity, Azure Monitor, Python (`requests`, `azure-identity`), Claude Code.

## Skills demonstrated

System design and architecture, Solution architecture, LLMs (large language models), AI agents, Function calling and tool use, Production systems, APIs and API integration, Prototypes and proof of concept, Debugging and root-cause analysis, Security and compliance, Python

Platforms: Azure AI Foundry, Claude, Whisper, Microsoft Entra ID, Key Vault, Managed Identity, Azure Monitor, Claude Code

## Timeline

- 2026-07-15: Deploy in-tenant Claude models and route coding assistant
- 2026-08-10: Deploy shared speech-to-text model in the tenant
- 2026-09-01: Raise speech-to-text quota and redeploy at higher capacity
