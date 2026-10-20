# Robocall evidence collection

Finds every call from an automated voice-bot campaign that hit branch location lines in the last day, confirms each one from its transcript, and writes the evidence file the platform's support team asked for.

## Problem

An automated voice bot was calling branch location lines asking for the person who handles the business's online payments, rotating its caller IDs. The case went to Genesys Cloud support through a contact-center services vendor, and support only reviews interactions from the last 24 hours. Each request meant finding the bot's calls among ordinary customer calls and sending them the same day.

## What I built

A read-only script that searches call transcripts for the bot's words, confirms each candidate against the bot's opening line, resolves the location and call details, and saves one CSV row per confirmed call in the format support uses. The search words and the confirming phrase are parameters, so the same tool works for a different bot.

## How it works

1. Search the platform's speech and text analytics transcripts over the look-back window. Every search word must appear in the transcript.
2. For each candidate conversation, fetch the transcript and keep it only if it contains the bot's phrase.
3. Pull the conversation's analytics details for start time, caller number, dialed number and duration, and resolve the location from the dialed number through the location data table.
4. Count every transcript indexed in the same window as a control.
5. Write the CSV (oldest first, times in UTC and Central) to the user's Downloads folder and print a summary: confirmed count, first and last call times, any excluded or errored candidates, and when the oldest call will leave the 24-hour window.

## Engineering decisions

- **Two-stage match.** The transcript search only matches single words, and a multi-word value returns nothing. So the search is a broad net of individual words, and the exact phrase check on the transcript is what confirms a call. Word choice matters: one plausible pairing also pulled in ordinary customer speech, so the defaults are words specific to the bot's line.
- **Prove a zero.** The control count of all transcripts in the window shows whether "0 calls" means the bot did not call or transcription and permissions are broken. A zero control during business hours is reported as a fault, not as no calls.
- **Read-only, with a manual spot check.** The script writes nothing to the platform. The validation step is opening one row's conversation and confirming the caller ID, the location and the bot's line by hand.
- **Platform transcripts are for search only.** If anyone needs quoted call text, the recording is transcribed with Whisper rather than quoting the platform's speech-to-text.
- **Fail loudly.** A failed search or auth call exits with an error code instead of producing an empty file.

## Stack

Python, Genesys Cloud speech and text analytics API (transcript search, transcript URLs), Genesys Cloud analytics and data tables APIs, CSV, Whisper.

## Skills demonstrated

System design and architecture, Reliability, APIs and API integration, QA and UAT testing, Debugging and root-cause analysis, Python

Platforms: Genesys Cloud, Whisper

## Timeline

- 2026-10-06: Identify nationwide robocall pattern and draft carrier ask
- 2026-10-12: Select additional robocall examples for traceback
- 2026-10-13: Analyze robocall caller-ID patterns across samples
- 2026-10-16: Break down robocall volume and file carrier fraud reports
- 2026-10-20: Add skill to pull the last 24h of matching robocalls
