# AI call quality grader

Scores recorded contact-center calls against the QA team's scorecard automatically with an LLM, so quality reviewers can cover far more calls per agent than manual listening allows.

## Problem

The central contact center's quality team scored calls by listening to recordings and answering a scorecard by hand. That limited how many calls per agent anyone could review. Any automated replacement had to keep call audio and transcripts inside the company's cloud tenant, write into the workbook the reviewers already used so their process did not change, and account for its own cost, because the department that used it was charged for each run.

## What I built

An on-demand Azure Container Apps Job that selects a sample of recorded calls for every agent in a date range, transcribes them, adds appointment context from the scheduling system, asks Claude to answer every scorecard question with a reason, and appends the scores to the QA team's Excel workbook on SharePoint. Around it I built a launcher that sizes and starts a run, a per-run cost ledger, a versioned rubric store, and an outcome email for every execution.

## How it works

recording -> transcription -> appointment context -> Claude scoring -> deterministic corrections -> QA workbook -> outcome email and cost ledger

- **Launch.** A small launcher takes an inclusive date range and a per-agent cap, validates them, reports the cost ceiling, checks the deployed image version, and starts one execution through the Container Apps REST API with per-execution overrides. It never edits the job template. A free "select only" mode picks the calls and reports counts and an estimate without grading anything.
- **Selection.** The job queries the Genesys Cloud analytics API for the period's calls on the scheduling queues, filtered to the wrap-up codes QA reviews, and samples up to the cap per agent. A blob-lease lock allows only one execution at a time.
- **Transcript.** Each recording goes to Whisper on Azure AI Foundry. The Whisper words are aligned onto the speaker labels from the Genesys transcript, producing a transcript with accurate words and correct speaker turns. Fallbacks cover calls where audio is missing.
- **Context.** A read-only call to the scheduling system's API returns whether an appointment was booked, when, and by whom.
- **Scoring.** Claude, deployed in the tenant, receives the transcript, call metadata and rubric at temperature 0 and returns each answer with its applicability and a reason through a single forced tool call. Incomplete output is retried. Deterministic code then corrects cases the rubric text cannot express, such as outbound calls and transfers.
- **Delivery.** Scores are appended to a table in the workbook through the Microsoft Graph Excel API, idempotent on conversation ID. If the workbook is locked, the load is deferred and retried, with the run's results file kept in blob storage. Every execution writes a cost record (token and audio usage) to a ledger and emails its outcome, counts and cost.
- **Calibration.** The job also reads the reviewers' calibration columns from the workbook, so a rubric change can be measured against human rulings before it is promoted.

## Engineering decisions

- **Both models run inside the tenant on Azure AI Foundry** instead of public APIs, so call audio and transcripts never leave the company's cloud boundary. I evaluated Copilot Studio as an alternative backend; Foundry was chosen.
- **Write to the workbook the reviewers already use**, so adopting the grader did not change the QA team's process. The writer is a dedicated app registration with Sites.Selected access to that one SharePoint site, and the Genesys client is view-only.
- **Model judgment plus deterministic post-processing.** Temperature 0, a forced tool call for structured output, and code-level corrections for cases the rubric prose cannot capture keep the scores repeatable.
- **The rubric is versioned data, not code.** A promote script publishes a rubric version to a blob document that production runs read, so rules change and roll back without an image rebuild.
- **Cost is measured per run** in a ledger, because the consuming department is charged per run. The launcher shows a ceiling before any paid run.
- **Re-runs are safe.** Already-graded calls are cached and rows are keyed on conversation ID, so a failed or timed-out run resumes, and re-running a window at a higher cap only adds new calls. A weekly schedule is implemented but not enabled (planned), pending a results destination sized for that volume.

## Stack

Python, Azure Container Apps Jobs, Azure AI Foundry (Claude, Whisper), Genesys Cloud analytics and recordings APIs, Microsoft Graph Excel API, SharePoint, Azure Blob Storage, Key Vault, Managed Identity.

## Skills demonstrated

Solution architecture, LLMs (large language models), Prompt engineering, Function calling and tool use, Evaluation frameworks (evals), Production systems, At scale, Systems integration, End-to-end ownership, Security and compliance, Contact center (CCaaS), Python

Platforms: Azure Container Apps Jobs, Azure AI Foundry, Claude, Whisper, Genesys Cloud, Microsoft Graph, SharePoint, Azure Blob Storage, Key Vault, Managed Identity

## Timeline

- 2026-07-27: Prototype call grading from transcripts and structured data
- 2026-07-29: Run transcription bake-off and wire scheduling API data
- 2026-07-30: Align word timestamps to speaker turns; add live grading tool
- 2026-08-03: Ingest reviewer exports and compare AI and human grades
- 2026-08-06: Lock grading ruleset and wire nearest-location check
- 2026-08-07: Apply reviewer final-sweep edits to grading rules
- 2026-08-13: Create grader API client and file cloud security review
- 2026-08-25: Port grader to a laptop-free scheduled cloud run
- 2026-08-26: Deploy and validate live grading pipeline in the cloud
- 2026-09-01: Align full-Whisper transcripts and re-validate grading
- 2026-09-03: Measure rule changes against re-adjudicated reviewer truth
- 2026-09-08: Preflight dashboard integration from reviewer sync notes
- 2026-09-09: Verify dashboard write phases and deploy new image
- 2026-09-11: Prototype detection of transfers without a booking attempt
- 2026-09-16: Build side-by-side grader on a low-code AI platform
- 2026-09-17: Compare grader backends on cost, quality and transcription
- 2026-09-23: Recalibrate grading rules from reviewer dashboard feedback
- 2026-09-24: Grade the survey checkpoint and restore dashboard formulas
