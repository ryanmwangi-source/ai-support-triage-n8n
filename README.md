# AI-Powered Customer Support Triage (n8n)

A production-hardened, AI-driven customer-support ticket triage system built on
[n8n](https://n8n.io). It automatically reads incoming support tickets, uses AI to
clean and standardise them, scores their urgency, drafts a reply, logs everything for
audit, and escalates the genuinely urgent cases to a human — around the clock.

This repository is the final capstone deliverable for the n8n Workflow Automation
course. It takes a fragile Week-8 prototype and re-engineers it to a production,
sellable standard across five dimensions: **structured AI output, fault tolerance,
model fallbacks, logging & governance, and overall robustness.**

---

## What it does

```
Webhook ─▶ Read tickets ─▶ Filter (New) ─▶ AI Janitor (clean) ─▶ AI Triage Officer (prioritise + draft)
        ─▶ Assemble record ─▶ Log EVERY ticket ─▶ Is Urgent?
                                                   ├─ High ─▶ Human Response WF (alert + customer draft)
                                                   └─ Med/Low ─▶ logged only
Any unhandled failure anywhere ─▶ Error Handling WF (log to ErrorLog + email alert)
```

## Architecture — three focused workflows

| Workflow | File | Role | Trigger |
|---|---|---|---|
| **1. Data Processing** | `workflows/01-data-processing.json` | Reads tickets, runs the 2-stage AI pipeline, logs everything, decides urgency. | Webhook `POST /new-ticket` |
| **2. Error Handling** | `workflows/02-error-handling.json` | Catches any failure, logs it, classifies severity, alerts. | Error Trigger (global) |
| **3. Human Response / Email** | `workflows/03-human-response-email.json` | Sends the urgent internal alert and drafts the customer reply. | Execute Workflow (called by WF1) |

The two AI agents live **inside Workflow 1** (cleaning + triage are steps in data
processing, not a separate workflow). Workflow 1 calls Workflow 3 for High-urgency
tickets; Workflows 1 and 3 both fall back to Workflow 2 on any unhandled error.

## Repository structure

```
.
├── README.md
├── ERROR_HANDLING.md                              # Standalone error-handling documentation
├── workflows/
│   ├── 01-data-processing.json
│   ├── 02-error-handling.json
│   └── 03-human-response-email.json
├── screenshots/                                   # Execution evidence + node configs
├── Triage Bot - Project Documentation For Week 10.docx
├── Triage Bot - Presentation Speech Guide.docx
└── Triage Bot - Presentation.pptx
```

## Tech stack & integrations

- **n8n** (self-hosted, v2.8.x) — workflow engine
- **Groq API** — `llama-3.3-70b-versatile` (primary LLM for both agents)
- **Mistral AI** — `mistral-large-latest` (fallback LLM for both agents)
- **Google Sheets** (OAuth2) — data store (input, audit log, error log)
- **Gmail** (OAuth2) — alerts + customer-reply drafts
- **LangChain Agent + Structured Output Parser** (n8n nodes)

## Environment requirements

- A running n8n instance (self-hosted or cloud), Node.js 20+ if self-hosting.
- Google account with a Google Sheet (three tabs — see Setup).
- API keys: **Groq**, **Mistral**. OAuth credentials: **Google Sheets**, **Gmail**.
- No secrets are stored in this repo — n8n keeps credentials separately by ID.

## Setup & configuration

### 1. Google Sheet
Create one spreadsheet with **three tabs** (exact headers matter):

| Tab | Columns |
|---|---|
| `SupportTickets` | TicketID, CustomerEmail, Issue, Category, Status |
| `TriagedTickets` | TicketID, CleanCategory, Urgency, SuggestedResponse, IsDuplicate, ModelUsed, ProcessedAt |
| `ErrorLog` | Timestamp, Workflow, FailedNode, ErrorMessage, Severity, ExecutionUrl |

### 2. Import the workflows
In n8n, **create a new blank workflow and import one JSON file into it** — repeat for
each of the three files (one file per blank workflow).

### 3. Connect credentials
On the affected nodes, select your credentials: Google Sheets OAuth2, Gmail OAuth2,
Groq API, Mistral Cloud API.

### 4. Wire the workflows together
- In **Workflow 1**, open **"Escalate to Human Response WF"** and set its **Workflow**
  to **Triage Bot - 3. Human Response / Email**.
- In **Workflow 1** and **Workflow 3**, open **⋯ → Settings → Error Workflow** and set
  it to **Triage Bot - 2. Error Handling**.

### 5. Publish (order matters)
Publish **Workflow 3 first**, then Workflow 2, then Workflow 1 (a parent can only be
published after its sub-workflow is published).

## Usage

Trigger a processing run by sending a POST request to the webhook:

```bash
curl -X POST "https://<your-n8n-host>/webhook/new-ticket"
```

The body is ignored — the webhook is the "process now" signal; the workflow reads the
tickets from the sheet itself. Keep test batches small (3–5 rows) to stay within free
API rate limits.

## Deployment

1. Provision n8n (Docker or n8n Cloud).
2. Add the four credentials.
3. Import + wire + publish the three workflows (steps above).
4. Point your ticket source at the production webhook URL.

## Reliability features

- **Structured AI output** — JSON-schema output parsers (enum-locked) on both agents.
- **Fault tolerance** — retry-on-fail (3×) on agents + Sheets nodes; continue-on-error.
- **Model fallback** — Groq → Mistral on both agents (cross-provider).
- **Logging & governance** — every ticket logged with model + timestamp; full ErrorLog.
- **Human-in-the-loop** — customer replies are Gmail drafts, never auto-sent.

See **[ERROR_HANDLING.md](ERROR_HANDLING.md)** for the full error-handling design.

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| Google Sheets append writes nothing; "Column names were updated after the node's setup" | The node's cached schema no longer matches the sheet. Open the node → re-pick the Sheet / refresh the columns list. |
| Sheet tab won't resolve after rename | Re-select the tab in the node's Sheet dropdown so n8n re-reads its gid. |
| Error Workflow can't be selected in Settings | The target workflow must be **published** first. |
| Triage agent always fails over to Mistral | The Groq Triage model must be a chat model (`llama-3.3-70b-versatile`), not an audio model. |
| AI agents error with 429 / rate limit | Free-tier burst limit (Groq ~30 req/min, ~1,000 req/day). Wait ~60s; use small batches. |
| ErrorLog empty after forcing an error | The global Error Workflow fires on **production** runs, not manual editor runs. Test by executing Workflow 2 directly, or trigger via the production webhook. |

## Author

**Ryan Mwangi** — n8n Workflow Automation, Final Capstone (2026).
