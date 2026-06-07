# Error Handling Documentation

This document details every error-detection, recovery, fallback, and alerting mechanism
in the AI-Powered Customer Support Triage system. The design follows defence-in-depth:
**prevent → absorb → isolate → catch → notify.**

---

## 1. Error detection techniques

| Layer | Technique | Where |
|---|---|---|
| **Input validation** | A Filter node passes only `Status = New`, so malformed/processed rows never enter the AI pipeline. | WF1 · "Keep Only New Tickets" |
| **Schema validation** | Structured Output Parsers reject AI output that doesn't match the JSON schema (enum-locked category & urgency). | WF1 · "Schema: Clean Category", "Schema: Urgency & Reply" |
| **Node-level failure capture** | Nodes with `Continue on error` expose a dedicated **error output** that detects per-item failures. | WF1 · Read, Log, Escalate |
| **Global failure capture** | An **Error Trigger** node fires automatically on any unhandled execution failure. | WF2 · "On Workflow Error" |

## 2. Error recovery mechanisms (retry)

All transient-failure-prone nodes are configured with automatic retry:

- **Retry On Fail = enabled**, **Max Tries = 3**, **Wait Between Tries = 2000 ms**.
- Applied to: both **AI agents** (Janitor, Triage Officer), **all Google Sheets** nodes
  (Read, Log, Append to ErrorLog), and the **Gmail** alert/draft nodes.

This absorbs momentary API/network blips (timeouts, brief 5xx, transient rate spikes)
without any human intervention — the node simply waits and retries.

## 3. Fallback processes (model fallback)

Each AI agent is wired to two language-model nodes:

- **Primary:** Groq `llama-3.3-70b-versatile`
- **Fallback:** Mistral `mistral-large-latest` (the agent's `needsFallback` slot, index 1)

If the primary model errors (provider outage, auth failure, sustained rate-limit), the
agent automatically retries the request against the fallback model. Because Groq and
Mistral are independent providers, a single provider outage cannot stop the system.

> **Known limit:** if *both* providers fail simultaneously, the agent exhausts its
> retries and the failure escalates to the Error Handling workflow (Section 5) — it
> fails loudly and is logged, never silently.

## 4. Continue-on-error implementations

`Continue (using error output)` routes a failed item to a separate branch instead of
aborting the whole run. This guarantees one bad ticket never kills the batch.

| Node | Workflow | On-error behaviour |
|---|---|---|
| Read SupportTickets | WF1 | Failure → "Read Failed (caught)" branch |
| Log ALL Tickets to TriagedTickets | WF1 | Failure → "Log Failed (caught)" branch |
| Escalate to Human Response WF | WF1 | Sub-workflow failure is caught; batch continues |
| Customer Reply (draft for review) | WF3 | `Continue` — a draft failure never blocks the alert |
| Append to ErrorLog | WF2 | `Continue` — logging failure never blocks the alert |

## 5. Notification & alerting strategy

The **Error Handling workflow (WF2)** is registered as the global **Error Workflow** for
WF1 and WF3. On any unhandled failure it runs this chain:

1. **On Workflow Error** (Error Trigger) — receives full failure context.
2. **Extract Error Details** (Set) — captures `Timestamp`, `Workflow`, `FailedNode`,
   `ErrorMessage`, `ExecutionUrl`, and a computed `Severity`.
3. **Append to ErrorLog** (Google Sheets) — writes a permanent audit row.
4. **Critical? (escalate)** (IF) — branches on severity.
5. **Gmail alert** — sends the appropriate email.

Two notification tiers:

- **Escalation Alert (urgent)** — subject `[ESCALATION] …`, for HIGH-severity errors.
- **Standard Alert** — subject `[Triage Bot] Handled error …`, for everything else.

## 6. Retry & escalation logic (severity classification)

Severity is derived automatically from the error message:

```
Severity = HIGH   if message matches /auth|credential|quota|rate limit|forbidden|401|403|429/
Severity = STANDARD  otherwise
```

- **HIGH** (authentication, quota, or rate-limit problems) → urgent escalation email,
  because these usually need a human to rotate a key, top up credit, or wait out a limit.
- **STANDARD** (transient/data issues) → logged + a standard notification; no immediate
  action required.

Every error — regardless of severity — is **always logged** to ErrorLog with a direct
link to the failed execution, so any incident is fully traceable and re-runnable.

---

## Summary: how a failure flows through the system

```
A node fails
   │
   ├─ transient?  ── Retry (3×, 2s) ──▶ success → continue
   │
   ├─ primary model down? ── Fallback (Groq → Mistral) ──▶ success → continue
   │
   ├─ one bad item? ── Continue-on-error ──▶ caught branch, batch keeps running
   │
   └─ unhandled?  ── Error Trigger (WF2) ──▶ log to ErrorLog ──▶ classify severity
                                                              ──▶ HIGH: escalate / else: notify
```
