# AI mailbox triage with human approval in Teams

Watches a shared support mailbox for requests to reroute a branch location's calls during an outage, proposes the change to the phone team in Teams, and carries it out only after a person clicks Approve.

## Problem

When a branch location loses phones, power or internet, someone emails a shared support mailbox asking for that location's calls to go to the central contact center, then later asks for them to come back. Someone on the phone team had to notice the email, work out which location and which direction, make the change in Genesys Cloud, and reply to the thread.

## What I built

A background monitor that runs inside the Teams operations assistant bot. It reads the mailbox through Microsoft Graph, filters out automated noise with cheap rules, and asks Claude whether each remaining email is a routing request, for which location, and in which direction. If it is, every phone team member gets an Adaptive Card in Teams with Approve and Decline buttons and a summary of the email thread. The first click decides for everyone. On Approve, the bot flips the location's outage flag in a Genesys data table that the IVR reads on every call, then replies all on the original email thread to confirm.

## How it works

mailbox -> Graph webhook (plus poll backstop) -> noise filter -> Claude classifier -> incident -> Adaptive Cards in Teams -> Approve -> Genesys data table write -> reply-all on the thread

- **Ingest.** A Graph change-notification subscription posts to the bot's webhook when a new message arrives. The handler checks a shared secret, acknowledges inside Graph's time limit, and fetches the message by ID in the background. A periodic poll re-reads recent mail so a dropped webhook is still caught. The subscription is renewed before Graph's expiry.
- **Classify.** A keyword and sender filter drops automated mail. A human reply into an automated thread is deliberately kept. Claude returns one JSON verdict: is it a request, direction, location code, requester, a short quote and a reconstructed thread summary. The code then hardens the verdict (normalizes the location code, whitelists the direction, requires a location code). Any doubt, parse error or model error means silence.
- **Propose.** One incident per location, direction and subject. The bot skips locations already in the requested state, then sends the card as a proactive 1:1 Teams message to each recipient using a conversation reference stored the last time that person messaged the bot.
- **Decide and act.** The first responder wins, and the other cards update in place to show who handled it. The Genesys write is idempotent, and the reply email is sent only if the row actually changed.
- **State.** Small JSON documents in Azure Blob Storage: the subscription, a kill switch, incidents (pruned after a retention window), stored conversation references, and a seen-message list shared by the webhook and the poll.

## Engineering decisions

- **Mailbox-scoped access.** The service principal's Graph mail permissions are bound with Exchange RBAC for Applications to the one shared mailbox. I removed the tenant-wide grants only after a positive test on that mailbox and denied control tests on other mailboxes.
- **Human in the loop for every write.** Nothing moves without an Approve click from an allowlisted person. A shadow mode posts cards but skips the write, which I used for rollout.
- **Idempotent write, conditional email.** A double click or a repeat request cannot double-route or double-email.
- **Webhook plus poll.** Webhooks are fast but can be dropped. The poll and the shared seen-list make delivery reliable without duplicates.
- **Message content is never trusted from the notification.** It is always fetched from Graph by ID, and the webhook accepts only the validation handshake and notifications carrying the expected client state.
- **Operator kill switch.** A script disables the monitor and deletes the Graph subscription without redeploying, and the monitor will not recreate the subscription while the switch is off.

## Stack

Python (aiohttp, Bot Framework SDK), Microsoft Graph (change notifications, mail), Exchange RBAC for Applications, Azure AI Foundry (Claude), Microsoft Teams Adaptive Cards, Genesys Cloud API, Azure Container Apps, Azure Blob Storage, Key Vault, Managed Identity.

## Skills demonstrated

Solution architecture, LLMs (large language models), Generative AI, Production systems, Reliability, APIs and API integration, Systems integration, QA and UAT testing, Security and compliance, Contact center (CCaaS), Python

Platforms: Microsoft Graph, Exchange RBAC for Applications, Azure AI Foundry, Claude, Microsoft Teams Adaptive Cards, Genesys Cloud, Azure Container Apps, Azure Blob Storage, Key Vault, Managed Identity

## Timeline

- 2026-06-23: Mine routing requests and define mailbox triage contract
