# Automated IVR test caller

Places scripted test calls into a development copy of the company IVR, presses keys, records what plays, and scores each test case PASS, FAIL or REVIEW with audio and flow-trace evidence.

## Problem

The company's primary inbound IVR was being restructured, and every change had to be tested by phoning the development copy, pressing keys and listening. Doing that by hand is slow, and disagreements about what a caller actually heard were hard to settle without a recording. Many test cases also need the development org set up first, such as a data-table row or a schedule forced open, and then put back exactly as it was.

## What I built

An on-demand Azure Container Apps Job that contains a software SIP phone registered to the Genesys Cloud development org. A laptop client starts a run with a list of test cases and opens the results page when it finishes. I first built it on a laptop under WSL2, then moved it into Azure because corporate group policy kept breaking the media leg.

## How it works

test case -> dev setup (snapshot, apply) -> SIP call with DTMF and recording -> revert and verify -> local transcription -> analytics lookup and flow trace -> verdict -> HTML report in blob storage

- **Media.** baresip registers a generic SIP station with the development registrar over TLS with mandatory SRTP, dials the development IVR, sends RFC 2833 DTMF (each digit timestamped from answer), and records the far-end audio. The Python code drives baresip over its TCP control interface.
- **Pass 1, calls.** For each case the job applies any development-side setup, places the call, then reverts the setup in a `finally` block. Calls are placed back to back so the SIP connection never idles past the outbound NAT timeout.
- **Pass 2, evidence.** Because the analytics API lags a finished call by a few minutes, evidence is gathered after all calls are placed. The job checks the raw audio for silence, transcribes it locally with faster-whisper, fuzzy-matches each expected phrase, finds the conversation in Genesys analytics, pulls the Architect flow execution trace, and checks it against an expected-trace fixture.
- **Verdict.** FAIL on any failed check (silent recording, missed phrase, failed trace check, a digit sent too soon after answer). REVIEW when evidence is missing. PASS only with both audio and trace evidence. A case whose evidence step throws is marked REVIEW, never lost.
- **Delivery.** An HTML summary (per-case audio player, phrase scores, trace checks, transcript, interaction link) and a CSV in the QA team's template columns are uploaded to blob storage in an outer `finally`, so evidence survives a failure or timeout. The laptop client downloads and opens it.

## Engineering decisions

- **Development org only, enforced in code.** The job holds only development credentials, the Genesys client class is pinned to the development environment, and every write path asserts it.
- **Every setup change is reversible and verified.** Each kind of write must be approved for that run, each object is snapshotted before the edit and re-read after the revert, and a mismatch stops the run. SIGTERM is trapped so a stopped run still reverts and uploads its evidence.
- **Case files cannot dial anything else.** Every case is validated before the phone starts: an allowlist of development test numbers, a DTMF character set, a cap on waits, and safe names.
- **Local speech-to-text.** faster-whisper is baked into the image and runs offline on the container CPU, so no AI service is called and there is no per-call cost. The prompt is seeded with vocabulary, never with the expected sentence, because seeding the exact greeting made the model skip it.
- **Solving SIP interop in the container.** The registrar required a minimum registration interval and an echoed digest `opaque`, which needed a newer baresip build. A loopback-only dnsmasq resolver maps the registrar names to one address so the SIP ACK reuses the registration TLS connection. RTP is outbound-first, so the container needs no inbound ports.
- **Hardened container.** Non-root user, no ingress, egress only to the development org and the storage account, and the SIP password is written only to a per-run file with restricted permissions that is never uploaded.

The tool has already recorded real routing defects in the IVR under development, such as a missing retry prompt and a menu path that played an error and disconnected.

## Stack

Python, baresip, SIP over TLS, SRTP, RFC 2833 DTMF, dnsmasq, faster-whisper (CTranslate2), Genesys Cloud analytics and Architect APIs, Azure Container Apps Jobs, Azure Blob Storage, Key Vault, Managed Identity.

## Skills demonstrated

System design and architecture, Prompt engineering, Reliability, APIs and API integration, Prototypes and proof of concept, QA and UAT testing, Debugging and root-cause analysis, Security and compliance, Customer experience (CX), Python

Platforms: baresip, dnsmasq, faster-whisper, Genesys Cloud, Azure Container Apps Jobs, Azure Blob Storage, Key Vault, Managed Identity

## Timeline

- 2026-09-17: Build SIP caller bot with local Whisper and verdicts
- 2026-09-21: Move IVR caller bot into an on-demand container job
- 2026-09-29: Commit IVR caller bot job to the platform repo
- 2026-10-06: Design a conversational voice QA agent
- 2026-10-30: Update bot dev client; scope two-sided test harness
