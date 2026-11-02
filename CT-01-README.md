# CT-01 Main - Event Intake & Orchestration

## A. Identity
| | |
|---|---|
| Name / ID | `CT-01 Main - Event Intake & Orchestration` / `wdj1MhTpYhbnDcvZ` |
| Purpose | Single entry point: validates, de-duplicates, assesses, scores, approves, dispatches |
| Status | IMPLEMENTED; CONFIGURATION REQUIRED; no execution evidence; `active: true`; no `errorWorkflow` in export |

## B. Beginner overview
Think of a reception desk. An event arrives, the clerk checks the form, stamps it, files a folder (the incident), asks an AI assistant for an opinion, a rulebook (code) decides how urgent it is and whether a manager must sign off, then a Slack message goes out. If Slack is down, the folder goes to a "try again later" tray that CT-05 watches.

## C. Trigger
`Webhook - Receive Ops Event`: `POST /ops-events`, `responseMode: responseNode`. **No authentication parameter is set** although the sticky note says "Authenticate" (F-03). Production URL form: `<n8n-host>/webhook/ops-events`.

## D. Input (example values are illustrative)
```json
{
  "event_id": "EVT-DEMO-1001",
  "timestamp": "2026-10-10T08:15:00Z",
  "department": "Logistics",
  "event_type": "shipment delay",
  "severity": "HIGH",
  "entity_id": "ORD-55412",
  "payload": { "delay_days": 5, "customer_email": "jane.doe@example.com" }
}
```
Rules (Code - Validate Event): `event_id` 3-64 chars `[A-Za-z0-9_-]`; `timestamp` ISO date, not > 5 min in the future; `department` one of Sales, Logistics, Finance, Customer Service, Inventory, Procurement (case-insensitive); `event_type` normalised to UPPER_SNAKE (`/^[A-Z][A-Z0-9_]{2,63}$/`); `severity` LOW/MEDIUM/HIGH/CRITICAL; `entity_id` non-empty; `payload` must be an object (defaults `{}`). `customer_email` is optional.

## E. Node-by-node
| Node | Type | Logic | Next / failure |
|---|---|---|---|
| Webhook - Receive Ops Event | webhook | Receives body at `$json.body` | Validate |
| Code - Validate Event | code | Builds `{is_valid, errors[], raw_string, normalized{...}}`; invalid `event_id` becomes `INVALID-<execution id>` | Hash |
| Crypto - Hash Payload | crypto SHA256 | Hashes `raw_string` into `payload_hash` | Check Duplicate |
| Database - Check Duplicate | supabase getAll `incidents`, `event_id eq`, limit 1, `alwaysOutputData` | Empty item when none | IF Duplicate |
| IF - Duplicate Event? | if | `!!$json.id` | true: Respond 200; false: Build Row |
| Respond - Duplicate (200) | respondToWebhook | `{status:'duplicate', incident_no, state}` | end |
| Set - Build Incident Row | set raw | `{...normalized, payload_hash, execution_id:$execution.id, state:'RECEIVED'}` | Create Incident |
| Database - Create Incident | supabase insert (autoMap) | Insert row; unique violation errors the run | Event Received Log |
| Set - Prepare Event Received Log / Sub-workflow - Log Event Received (CT-02) | set / executeWorkflow | RECEIVED->RECEIVED audit `EVENT_RECEIVED` | Validating |
| Set - Prepare VALIDATING Update / Sub-workflow - Set VALIDATING (CT-02) | | `VALIDATION_STARTED` | IF Valid |
| IF - Event Valid? | if | `Validate Event.is_valid` | true: 202; false: 400 |
| Respond - Invalid (400) | respond | `{status:'rejected', incident_no, errors}` | Prepare REJECTED |
| Set - Prepare REJECTED Update / Sub-workflow - Set REJECTED (CT-02) | | `VALIDATION_FAILED`, WARNING, patch `rejection_reason: INVALID_SCHEMA` | end |
| Respond - Accepted (202) | respond | `{status:'accepted', incident_no, event_id, state:'VALIDATING'}`; flow continues after the HTTP reply | Prepare AI_PROCESSING |
| Set - Prepare AI_PROCESSING Update / Sub-workflow - Set AI_PROCESSING (CT-02) | | `AI_STARTED` | IF Email |
| IF - Customer Email Present? | if | regex on `normalized.payload.customer_email` | true: HubSpot; false: Build AI Context |
| HubSpot - Find Contact | hubspot v2 search (appToken) | `email EQ <lowercased>`, limit 2, retry 3x/2 s, `continueRegularOutput`, `alwaysOutputData` | Build AI Context |
| Code - Build AI Context | code | `crm_status` NO_EMAIL / FOUND / MULTIPLE_MATCHES / NOT_FOUND / LOOKUP_FAILED; `contact`; `session_key` (`ct-hs-<id>` or `ct-ent-<dept>-<entity>`) | AI Agent |
| AI - Assess Impact | AI Agent v2.2 | Prompt = normalised event + CRM context; system message "Brainpower"; retry 2x/1.5 s; error output enabled | success: Compute; error: Fallback Audit |
| Google Gemini Chat Model | LM (temp 0.2) | model name not set in parameters (UNVERIFIED default) | ai_languageModel |
| Postgres Chat Memory | memory | session key from Build AI Context | ai_memory |
| Structured Output Parser | parser | manual JSON schema (6 required fields) | ai_outputParser |
| Code - Build AI Fallback Audit / Sub-workflow - Log AI Fallback (CT-02) | | `AI_FALLBACK` WARNING audit, state stays AI_PROCESSING | Compute |
| Code - Compute Priority & Routing | code | see F. | Save Classification |
| Database - Save Classification | supabase update | writes `ai_*`, `risk_score`, `priority`, `requires_approval`, `notify_departments`, `sla_due` | Prepare CLASSIFIED |
| Set - Prepare CLASSIFIED Update / Sub-workflow - Set CLASSIFIED (CT-02) | | metadata carries priority, risk, crm status | IF Needs Approval |
| IF - Needs Approval? | if | `requires_approval` | true: Awaiting; false: Processing |
| Set - Prepare AWAITING_APPROVAL Update / Sub-workflow - Set AWAITING_APPROVAL (CT-02) | | `APPROVAL_REQUESTED` | Create Approval |
| Database - Create Approval Record | supabase insert `approvals` | status PENDING, `approver_email` literal placeholder, reason | Gmail |
| Gmail - Request Approval | gmail sendAndWait | double approval, "Reject" label, wait 45 min, `continueRegularOutput` | Read Decision |
| Code - Read Approval Decision | code | `data.approved` true/false/else -> APPROVED/REJECTED/TIMEOUT; builds state update | Record Decision |
| Database - Record Decision | supabase update `approvals` by id | `status`, `decided_at` | Prepare Decision |
| Set - Prepare Decision Update / Sub-workflow - Set Decision State (CT-02) | | state APPROVED, REJECTED or ESCALATED | Switch |
| Switch - Route Decision | switch | outputs Approved / Rejected / fallback "Timeout" | see below |
| Slack - Notify Rejection / Slack - Approval Timeout Alert | slack | hard-coded channel; no retry | end |
| Set - Prepare PROCESSING Update / Sub-workflow - Set PROCESSING (CT-02) | | previous state APPROVED or CLASSIFIED | Prepare Dispatch |
| Set - Prepare Dispatch | set | `{incident: dispatch_incident, attempt:1}` | CT-03 |
| Sub-workflow - Dispatch Action (CT-03) | executeWorkflow, error output | success output 0, failure output 1 | Completed / Retry |
| Set - Prepare COMPLETED Update / Sub-workflow - Set COMPLETED (CT-02) | | `ACTION_DISPATCHED` | end |
| Code - Build Retry Update / Sub-workflow - Set RETRYING (CT-02) | | `DISPATCH_FAILED`, `retry_count:1`, `next_retry_at: now+1 min` (demo constant) | end |

## F. Decision logic
- **Duplicate:** any existing row with that `event_id`.
- **Valid:** all validation rules pass. Invalid events are still stored and REJECTED.
- **AI fallback:** agent error output. Default: category OTHER, impact 5, confidence 0, no departments.
- **Risk:** `min(100, round((SEV_W*15 + impact*4) * DEPT_W))`; `SEV_W` LOW1 MEDIUM2 HIGH3 CRITICAL4; `DEPT_W` Finance 1.3, Logistics 1.2, Sales 1.1, Inventory 1.1, Procurement 1.0, Customer Service 1.0.
- **Priority:** >=80 P1, >=60 P2, >=35 P3, else P4. **SLA:** P1 30, P2 120, P3 480, P4 1440 minutes.
- **Approval required** if any: severity HIGH/CRITICAL; risk >= 75; AI failed; confidence < 0.60.
- **Notify list:** cross-department map union AI departments (validated against the six names), excluding origin.
- **Severity authority:** the code never lets AI change severity. The prompt claims human_approval_required/incident_required, but those fields are neither in the parser schema nor consumed (F-09). The prompt knows LOW/MEDIUM/HIGH only; CRITICAL is accepted by validation and code (F-08).
- **Clamps:** impact 1-10, confidence 0-1 (second validation layer). Severity-band consistency of impact is **not** checked.

## G. Integrations
Gemini (credential type `googlePalmApi`): no tools, parser attached, `autoFix` not configured. Postgres chat memory (credential `postgres`): table created/managed by n8n, **not in the supplied schema**. HubSpot native search, app-token auth, read-only. Gmail OAuth2 `sendAndWait`. Slack API (3 posts, hard-coded channel). Supabase service credential. Error scenarios: model/parse error -> fallback; HubSpot error -> `LOOKUP_FAILED`, continue; Gmail error -> treated as TIMEOUT.

## H. Database
`incidents`: read (duplicate), insert, update (classification). `approvals`: insert, update. Audit via CT-02. Status path as in [DATABASE_SCHEMA.md](../../DATABASE_SCHEMA.md#state-transitions).

## I. Outputs
HTTP 200/202/400 (before the long steps), incident row, audit rows, approval email, Slack messages, dispatch via CT-03.

## J. Error handling
Node retries (AI 2x, HubSpot 3x), AI fallback, dispatch error output -> RETRYING. **No other node has `onError`; no `errorWorkflow` is set**, so a Supabase failure, unique-constraint violation, or Slack failure in the rejection/timeout notices ends the execution unhandled unless CT-04 is attached in the live instance (F-04).

## K. Configuration
Credentials: Supabase, Google Gemini, Postgres (chat memory), HubSpot app token, Gmail OAuth2, Slack. Replace placeholder approver emails (two different literals), Slack channel ID, re-select CT-02/CT-03 in Execute Workflow nodes, set Gemini model, set CT-04 as error workflow.

## L. Test cases
Success MEDIUM event; HIGH event requiring approval; missing field (400); bad timestamp; future timestamp; duplicate id (200); concurrent duplicate; AI failure; confidence 0.4; Slack failure; Gmail failure. See [TESTING_AND_QA.md](../../TESTING_AND_QA.md).

## M. Troubleshooting
400 with `errors[]`: read the array. Always 200 duplicate for a corrected event: earlier rejected incident owns the id (F-11). Stuck in AI_PROCESSING: CT-05 escalates after 10 min. Agent errors: check model name, parser output, credentials.

## N. Summary
CT-01 is the front desk and project manager: it checks, stores, asks AI, applies fixed rules, asks a human when risky, and hands the Slack job to CT-03.
