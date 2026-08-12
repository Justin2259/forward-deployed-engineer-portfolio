# Attendance line voicemail transcriber

Turns call-in absence voicemails into an email with a transcript and the audio, delivered to every supervisor and staffing planner who needs it.

## Problem

Company policy requires contact-center staff who are sick or running late to call in and say so out loud rather than send an email. Before this system they called their own supervisor, who was supposed to relay the message to a Teams channel for workforce management and leadership. With supervisors and staff spread across time zones, that relay often did not happen, and nobody knew who had reported in.

## What I built

A dedicated toll-free attendance line in Genesys Cloud that answers around the clock and records a voicemail, plus two Azure Container Apps Jobs: a transcriber that turns each new voicemail into one email carrying the transcript and the recording, and an hourly heartbeat that alerts me if the transcriber stops or transcription keeps failing.

## How it works

caller -> IVR greeting -> group voicemail -> notification email in a shared mailbox -> Graph read -> Whisper -> one email to the distribution list

- **Genesys side.** The IVR plays a short greeting with no schedule gating and transfers to a group voicemail whose only member is an intake user tied to a shared team mailbox. Genesys's own voicemail notification email, with the recording attached, is how the audio leaves Genesys. The job never calls Genesys.
- **Read.** Every two minutes the transcriber lists the shared mailbox's Inbox through Microsoft Graph with a server-side filter on received time and sender, follows paging, refuses partial results, and narrows to the attendance line's subject marker so other voicemail boxes are ignored. It never deletes, moves or marks messages read.
- **Transcribe.** Each recording goes to Whisper on Azure AI Foundry, with the prompt seeded with staff surnames and two seconds of silence padding. Calls are paced to the deployment's rate limit and back off on throttling.
- **Send.** One email goes out from the shared mailbox with the transcript, the calling number, the received time in the local business time zone, and the original audio attached when it fits.
- **State.** A small JSON document in blob storage records processed message IDs, timestamps and failure counters. No audio, transcript or caller identity is stored in Azure; the email is the record.
- **Heartbeat.** A second job, on the same image with a command override, reads the transcriber's state without writing it and emails me when the last successful run is too old or Whisper has failed several times in a row, throttled to avoid alert storms.

## Engineering decisions

- **Transcript only, no extraction and no LLM.** Call-outs often contain medical detail, so the job deliberately does not pull out a name, reason or status. Leaving out structured fields is also the control that keeps health information out of anything structured.
- **A singleton lock with staleness recovery.** Scheduled Container Apps Jobs have no forbid-concurrent setting, and the state save is a whole-document overwrite, so two overlapping runs would double-send. The job takes an infinite blob lease, a second run exits cleanly as a no-op, and a lease older than the replica timeout is treated as orphaned and broken.
- **Degraded is not fatal.** A Whisper failure, empty transcript, missing or non-audio attachment, or oversized recording still sends the email with a banner naming what went wrong, so no absence is ever lost. Only a failed send fails the execution, which makes the execution status the monitoring signal. State is saved after every send so a restart cannot resend.
- **Whisper tuned for phone audio.** On narrowband telephony audio, seeding the prompt with surnames is what got names right and stopped Whisper dropping the caller's opening words.
- **Least privilege by reuse.** The Graph credential's mail permissions are scoped by Exchange Online RBAC for Applications to the one shared mailbox. Untrusted input is handled as such: Inbox only, attachments accepted only if the bytes are a real RIFF/WAVE file, a fixed outbound filename, and HTML-escaped transcripts.
- **Safe rollout switches.** A shadow mode does everything except send, and a parity mode sends only to me with a validation subject prefix. An offline self-test covers the duplicate-send guards, prompt truncation, daylight-saving rendering, the dedupe key and the lock skip path.

## Stack

Python, Genesys Cloud (IVR, group voicemail), Microsoft Graph mail API, Exchange Online RBAC for Applications, Azure AI Foundry (Whisper), Azure Container Apps Jobs, Azure Blob Storage, Key Vault, Managed Identity.

## Skills demonstrated

Solution architecture, Prompt engineering, Production systems, Reliability, APIs and API integration, Systems integration, End-to-end ownership, QA and UAT testing, Security and compliance, Contact center (CCaaS), Python

Platforms: Genesys Cloud, Microsoft Graph, Exchange Online RBAC for Applications, Azure AI Foundry, Whisper, Azure Container Apps Jobs, Azure Blob Storage, Key Vault, Managed Identity

## Timeline

- 2026-08-10: Build voicemail line flow and transcription prototype
- 2026-08-11: Add transcription container job for group voicemail
- 2026-08-12: Harden transcriber with singleton lock and local time
