# Justin Larpenteur: Forward Deployed Engineer, Applied AI

**Forward Deployed and Applied AI Engineer with a contact-center backbone.** I build production LLM systems, AI agents and AI voice agent integrations on enterprise contact-center platforms (Genesys Cloud CX), Azure AI Foundry and Claude, and I build with Claude Code every day. The work runs from technical discovery and system design through implementation, testing and operating it in production.

[LinkedIn](https://www.linkedin.com/in/justinlarpenteur) | [Vonimate](https://vonimate.com)

## What this repository is

A record of the systems I designed and built as the lead telephony and AI automation engineer for a multi-site enterprise contact center. Each page covers the problem, what I built, how it works, the engineering decisions and the skills it demonstrates. Employer data, names, identifiers and configuration are left out on purpose; the pages describe architecture and method only.

History note: this repository's history was rebuilt from my work log, with each commit dated to when the work happened.

## Highlights

- **12 cloud jobs, 2 always-on services and 42 agent skills** built and run in production on Azure and Claude, each with dry-run gates, measured accuracy and change control.
- **LLM call-quality grader with an evaluation framework:** agreement with human QA reviewers raised from 78.9% to 90.2% after recalibration, with temperature 0 scoring, forced tool calls and a regression gate on every rubric change. [Read more](systems/ai-call-quality-grader.md)
- **Migrations proven before cutover:** byte-identical parity with the legacy workflow output before every production switch. [Read more](systems/wfm-outbound-call-feed.md)
- **An AI-assisted engineering method** where every agent skill pairs a contract the LLM reads with deterministic scripts, because per-step accuracy compounds. [Read more](systems/ai-assisted-engineering-method.md)

## Skills and where they are shown

| Skill | Shown in |
|---|---|
| LLMs, generative AI | [AI call quality grader](systems/ai-call-quality-grader.md), [AI mailbox triage](systems/ai-mailbox-triage.md), [Teams operations assistant](systems/teams-ops-assistant-bot.md), [Help desk report with AI narrative](systems/help-desk-call-report.md) |
| AI agents, agentic AI, conversational AI | [Teams operations assistant](systems/teams-ops-assistant-bot.md), [AI-assisted engineering method](systems/ai-assisted-engineering-method.md), [In-tenant AI model hosting](systems/in-tenant-ai-model-hosting.md) |
| AI voice agents, contact center (CCaaS), IVR | [IVR routing to an AI voice agent](systems/ivr-ai-voice-routing.md), [Inbound call and voice-agent reporting](systems/inbound-call-reporting.md), [Automated IVR test caller](systems/automated-ivr-test-caller.md), [Call investigation and routing validation](systems/call-investigation-and-routing-validation.md), [Location telephony build-out](systems/location-telephony-buildout.md) |
| Evaluation frameworks (evals), prompt engineering | [AI call quality grader](systems/ai-call-quality-grader.md), [Automated IVR test caller](systems/automated-ivr-test-caller.md), [Voicemail transcription pipeline](systems/voicemail-transcription-pipeline.md) |
| Function calling and tool use | [AI call quality grader](systems/ai-call-quality-grader.md), [Teams operations assistant](systems/teams-ops-assistant-bot.md), [In-tenant AI model hosting](systems/in-tenant-ai-model-hosting.md) |
| System design and architecture, solution architecture | [Automation platform foundation](systems/automation-platform-foundation.md), [In-tenant AI model hosting](systems/in-tenant-ai-model-hosting.md), [AI mailbox triage](systems/ai-mailbox-triage.md), [HR-driven user provisioning](systems/scim-user-provisioning.md) |
| Production systems, reliability | [Phone fleet health monitor](systems/phone-fleet-health-monitor.md), [Daily call-routing report](systems/daily-call-routing-report.md), [Voicemail transcription pipeline](systems/voicemail-transcription-pipeline.md), [Workforce-management call feed](systems/wfm-outbound-call-feed.md) |
| At scale | [AI call quality grader](systems/ai-call-quality-grader.md), [Location telephony build-out](systems/location-telephony-buildout.md), [Outbound call analytics](systems/outbound-call-analytics.md) |
| APIs, systems integration | [HR-driven user provisioning](systems/scim-user-provisioning.md), [Monthly inbound call report](systems/monthly-inbound-call-report.md), [Branded caller ID onboarding](systems/branded-caller-id-onboarding.md), [Workforce-management call feed](systems/wfm-outbound-call-feed.md) |
| Technical discovery and requirements, stakeholder management | [Contact-center interval dataset](systems/contact-center-interval-dataset.md), [Outbound call analytics](systems/outbound-call-analytics.md), [Team workspace rollout](systems/team-workspace-rollout.md) |
| End-to-end ownership, implementation | [Location telephony build-out](systems/location-telephony-buildout.md), [Contact-center interval dataset](systems/contact-center-interval-dataset.md), [Automation platform foundation](systems/automation-platform-foundation.md) |
| Prototypes and proof of concept | [Automated IVR test caller](systems/automated-ivr-test-caller.md), [In-tenant AI model hosting](systems/in-tenant-ai-model-hosting.md) |
| QA and UAT testing | [Automated IVR test caller](systems/automated-ivr-test-caller.md), [Branded caller ID onboarding](systems/branded-caller-id-onboarding.md), [Routing change validation](systems/call-investigation-and-routing-validation.md) |
| Debugging and root-cause analysis | [Desk phone diagnostics](systems/desk-phone-diagnostics.md), [Call investigation](systems/call-investigation-and-routing-validation.md), [Robocall evidence collection](systems/robocall-evidence-collection.md) |
| Security and compliance | [ServiceNow ticket automation](systems/servicenow-ticket-automation.md), [AI mailbox triage](systems/ai-mailbox-triage.md), [In-tenant AI model hosting](systems/in-tenant-ai-model-hosting.md) |
| Customer experience (CX), measurable outcomes | [Branded caller ID onboarding](systems/branded-caller-id-onboarding.md), [Inbound call and voice-agent reporting](systems/inbound-call-reporting.md), [Desk phone diagnostics](systems/desk-phone-diagnostics.md) |
| Documentation and playbooks | [AI-assisted engineering method](systems/ai-assisted-engineering-method.md), [Engineering workflow tooling](systems/engineering-workflow-tooling.md) |
| Python, AWS | Every page uses Python; AWS in [Monthly inbound call report](systems/monthly-inbound-call-report.md) and [Inbound call reporting](systems/inbound-call-reporting.md) |

**Platforms:** Claude and the Anthropic API, Claude Code, Azure AI Foundry, Whisper, Genesys Cloud CX, Azure (Container Apps, Container Apps Jobs, Functions, Bot Service, Key Vault, Managed Identity, Storage, Monitor), Microsoft Graph, Microsoft Teams, Entra ID, ServiceNow, Azure DevOps, AWS, SIP.

## Systems

### AI and LLM systems
- [IVR routing to an AI voice agent](systems/ivr-ai-voice-routing.md): eligibility checks, SIP hand-off and hand-back with UUI headers, second-language routing and tiered IVR testing.
- [AI voice agent demo kiosk](systems/browser-softphone-kiosk.md): a browser softphone page that let event visitors call the AI voice agent.
- [AI call quality grader](systems/ai-call-quality-grader.md): LLM scoring of recorded calls against a QA rubric, with Whisper transcription and a cost ledger.
- [Automated IVR test caller](systems/automated-ivr-test-caller.md): a SIP test caller that dials the IVR, transcribes what it hears and grades each test case.
- [Telephony operations assistant in Microsoft Teams](systems/teams-ops-assistant-bot.md): a conversational AI bot with tool use and one confirmed write action.
- [AI mailbox triage with human approval](systems/ai-mailbox-triage.md): an LLM classifies requests, a person approves in Teams, the system makes the change.
- [Voicemail transcription pipeline](systems/voicemail-transcription-pipeline.md): voicemail to Whisper transcript to one email, with a heartbeat monitor.
- [IT service desk call report with AI narrative](systems/help-desk-call-report.md): code computes the numbers, the LLM writes the narrative.
- [In-tenant AI model hosting](systems/in-tenant-ai-model-hosting.md): Claude and Whisper deployed inside the company cloud tenant.

### Contact-center engineering (Genesys Cloud CX)
- [Location telephony build-out pipeline](systems/location-telephony-buildout.md)
- [Desk phone diagnostics and remediation](systems/desk-phone-diagnostics.md)
- [Call investigation and routing change validation](systems/call-investigation-and-routing-validation.md)
- [Branch phone fleet health monitor](systems/phone-fleet-health-monitor.md)
- [HR-driven phone system user provisioning (SCIM)](systems/scim-user-provisioning.md)
- [Branded caller ID onboarding](systems/branded-caller-id-onboarding.md)
- [Robocall evidence collection](systems/robocall-evidence-collection.md)
- [On-call escalation hotline](systems/on-call-escalation-hotline.md)
- [Call recording compliance](systems/call-recording-compliance.md)
- [Agent softphone sign-out diagnostics](systems/agent-softphone-diagnostics.md)
- [Softphone embedded in the CRM](systems/crm-embedded-softphone.md)

### Contact-center data and reporting
- [Contact-center interval forecasting dataset](systems/contact-center-interval-dataset.md)
- [Monthly inbound call report](systems/monthly-inbound-call-report.md)
- [Inbound call and AI voice-agent reporting](systems/inbound-call-reporting.md)
- [Outbound call analytics](systems/outbound-call-analytics.md)
- [Daily call-routing diversion report](systems/daily-call-routing-report.md)
- [Outbound call feed to workforce management](systems/wfm-outbound-call-feed.md)

### Platform and engineering method
- [Automation platform foundation](systems/automation-platform-foundation.md)
- [AI-assisted engineering method](systems/ai-assisted-engineering-method.md)
- [Engineering workflow tooling](systems/engineering-workflow-tooling.md)
- [ServiceNow ticket automation](systems/servicenow-ticket-automation.md)
- [Team workspace rollout](systems/team-workspace-rollout.md)
