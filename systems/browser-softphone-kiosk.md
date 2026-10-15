# Browser softphone demo kiosk

A touch-screen web page that acted as the phone in an event booth, letting visitors call the AI voice agent with one tap.

## Problem

The company wanted visitors at an internal event to talk to its AI voice agent from a phone booth with a touch screen. Genesys Cloud has no unauthenticated click-to-call, so the page had to sign in as a real user and place calls through a WebRTC softphone. The page was hosted on the public website by the web team, who owned the CMS, styles and security headers, and the booth PC was run by someone else. It had to work unattended for several days, with no one from telephony at the booth.

## What I built

- The kiosk page logic: two buttons ("I have filed a claim" and "I'm a new customer"), each dialing its own demo line, with live call status and reliable hang-up.
- The demo lines behind it: a self-contained inbound flow that presents a scenario-specific caller number, a bridge queue, and a dedicated trunk to the vendor's test agent, which reads only the caller number from the SIP UUI header and stages the matching customer on its side.
- A build script that splices my JavaScript into the web team's page and produces the body-only file they publish, plus a script that adds or removes the demo routing on a user's site.
- An event runbook: revoke the kiosk user's tokens the night before, sign in fresh at the booth, and confirm in the audit trail that it is the only sign-in.

## How it works

tap -> WebRTC SDK places the call -> demo line inbound flow sets caller ID -> bridge queue -> SIP transfer with UUI -> AI voice agent

- **Sign-in.** If no token is stored, a sign-in card appears over the attract screen with the provider pinned explicitly. The OAuth implicit-grant token is kept in local storage and scrubbed from the URL. The page URL is on the client's exact redirect allowlist.
- **Microphone.** The page opens the microphone at load and keeps it open, hands that stream to the SDK as the default, reopens it if the track ends or the device changes, and re-checks it before each dial.
- **Call status.** The page polls the conversations API for the call it created and moves from "Connecting" to "picking up" (any far-end participant connected) to "You're talking to" the agent (customer participant connected). The SDK's own update event cannot see the far end.
- **Hang-up.** Done, Back, Escape, idle timers, the agent hanging up and closing the tab all disconnect the page's own participant through the API, and the result is checked in Genesys by disconnect type rather than by the page moving on.
- **Sign-out.** A small status card revokes the token and ends the Genesys login session, so the next sign-in does not silently reuse a single sign-on session.

## Engineering decisions

- **Keep the microphone open for the whole session.** The booth's camera microphone stalled after idle periods, so calls placed after a quiet spell had no audio until the browser leg timed out.
- **Disable auto-answer.** The SDK auto-answers outbound calls on every browser signed in as the same user, which caused an idle test kiosk to join calls placed elsewhere. The page now accepts only the session it created and ignores the rest.
- **One station, one browser.** A WebRTC station locks to one client, so persistent connection is off on the kiosk line and tokens are revoked before the event.
- **Status from the API, no fallback timers.** An earlier timer-based fallback announced "picking up" on calls that had actually failed, so status now comes only from the conversation's real participant states.
- **Route on the user's site.** A user-originated call resolves the in-queue flow's transfer against the user's own site, not the org default, so testers on other sites needed the demo route added to theirs.
- **Body-only deliverable with a script hash.** The web team publishes only the body and owns headers, so every change shipped as a body file plus the inline-script hash for their content security policy.

The booth ran across the full event with no failed calls. The one gap was the sign-in token expiring partway through, which needed a fresh sign-in; for a multi-day event the fix is an automatic re-sign-in or a fresh sign-in each morning.

## Stack

JavaScript, Genesys Cloud WebRTC SDK, Genesys Cloud Platform API (conversations, tokens, audit), OAuth implicit grant, Genesys Cloud Architect, SIP with UUI headers, Python.

## Skills demonstrated

AI voice agents, Prototypes and proof of concept, APIs and API integration, Systems integration, Reliability, Debugging and root-cause analysis, Security and compliance, Stakeholder management, Customer experience (CX), End-to-end ownership

Platforms: Genesys Cloud CX, WebRTC softphone, Architect, SIP, Python

## Timeline

- 2026-10-07: Build browser softphone demo page and web phone user
- 2026-10-09: Provision web phone account; watch live test calls
- 2026-10-14: Ship softphone page with hangup handling
- 2026-10-15: Open mic at page load to cut first-call lag
