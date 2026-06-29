# CRM-embedded softphone

Design and spike for putting contact-center call control inside the repair-management system that outbound agents already work in.

## Problem

Outbound contact-center agents work in the repair-management system (the company's CRM) but had to switch to a separate softphone to place and manage calls. The business mockup asked for call control inside the CRM: make, receive, transfer, consult, conference and end calls, click-to-dial on each phone field of a record, screen pop, and automatic wrap-up. Agents also needed their shift-start single sign-on to carry into the embedded phone without a second login.

## What I built

This project is at the design and spike stage.

- A technical evaluation of two approaches: the Genesys Cloud Embeddable Framework, and a custom client built on the WebRTC SDK, the platform API client and notifications.
- A local sandbox with a host page that loads both embeds, a PKCE pop-up sign-in, a call-control verification script and a script to create the PKCE OAuth client.
- A postMessage contract between the CRM and the embed, so the CRM team can wire its phone icons to dial without knowing Genesys internals.

## How it works

CRM page -> phone icon click -> postMessage contract -> embedded softphone -> Genesys Cloud call -> backend wrap-up

- The CRM sends dial and call-control requests through the postMessage contract; the embed places and manages the call.
- Sign-in uses a PKCE pop-up so the embed can reuse the agent's existing single sign-on session.
- Wrap-up is posted from the backend in either approach, and outbound calls are placed on behalf of a queue so they get wrap-up and after-call work.

## Engineering decisions

- **Embeddable Framework as the primary.** The mockup shows the softphone in a corner of the CRM with click-to-dial, which the framework provides natively. The custom SDK client is the documented fallback.
- **Spike both before committing**, against the real environment, so the choice rests on evidence rather than documentation.
- **Single login is a pop-up and first-party decision, not an SDK one.** Whether the embed can reuse the agent's session depends on how sign-in is hosted, so that was designed separately from the call-control choice.
- **Clear ownership split.** The CRM team wires its phone icons to the contract; I own the embed and the contract.
- **Gated live tests.** Creating the OAuth client, interactive sign-in and any real call need explicit approval and use a test number only; the sandbox places no autonomous live calls.

## Stack

Genesys Cloud Embeddable Framework, Genesys Cloud WebRTC SDK, Genesys Cloud Platform API and notifications, OAuth PKCE, postMessage, Python (FastAPI sandbox).

## Skills demonstrated

Solution architecture, Technical discovery and requirements, Prototypes and proof of concept, APIs and API integration, Systems integration, Contact center (CCaaS), Python

Platforms: Genesys Cloud CX, Embeddable Framework, WebRTC softphone, Python

## Timeline

- 2026-06-29: Scope embedded call control for a CRM outbound dialer
