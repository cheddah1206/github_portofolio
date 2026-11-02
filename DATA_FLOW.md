# DATA_FLOW.md

Example event: `EVT-DEMO-1001` (fictional). Every AI output and every runtime value below is **illustrative**; the field paths, expressions and transformations are **confirmed** from the JSON. Nothing here was executed.

**Persistence rule:** n8n executions do not share memory. Data crosses boundaries only as (1) Execute Workflow input/output items, (2) rows in Supabase, (3) HTTP bodies. CT-01 additionally re-reads its own earlier nodes with `$('Node').first().json` within one execution.

## Step 1. Webhook input (CT-01)
`Webhook - Receive Ops Event` outputs one item: `{ headers, params, query, body: {...event...} }`. Example `body`: see `examples/sample-event.json`. Confirmed: the validator reads `$input.first().json.body`.

## Step 2. Validation and normalisation
`Code - Validate Event` output (confirmed shape):
```json
{ "is_valid": true, "errors": [], "raw_string": "{\"event_id\":\"EVT-DEMO-1001\",...}",
  "normalized": { "event_id": "EVT-DEMO-1001", "event_timestamp": "2026-10-10T08:15:00.000Z",
    "department": "Logistics", "event_type": "SHIPMENT_DELAY", "severity": "HIGH",
    "entity_id": "ORD-55412", "payload": { "delay_days": 5, "customer_commitment_at_risk": true, "customer_email": "jane.doe@example.com" } } }
```
Transformations: department case-corrected; `timestamp` renamed to `event_timestamp` (ISO); `event_type` upper-snaked. `execution_id` is **not** part of the event contract; it is added in Step 4.

## Step 3. Hash and duplicate check
`Crypto - Hash Payload` adds `payload_hash` (SHA-256 of `raw_string`, i.e. of the whole raw body). `Database - Check Duplicate` runs `SELECT ... WHERE event_id = 'EVT-DEMO-1001' LIMIT 1`; with no row, `alwaysOutputData` emits an empty item, so `IF - Duplicate Event?` evaluates `!!$json.id` = false. Note: `Check Duplicate` outputs only the DB result; later nodes read `payload_hash` via `$('Crypto - Hash Payload')` (correct).

If a row exists: response `200 {"status":"duplicate","incident_no":101,"state":"COMPLETED"}`; no audit row, no Slack message.

## Step 4. Incident insert
`Set - Build Incident Row` -> `{...normalized, payload_hash, execution_id: <CT-01 execution id>, state:'RECEIVED'}`; `Database - Create Incident` inserts it. DB defaults: `id`, `incident_no`, `created_at`, `retry_count=0`, `max_retries=3`. Output = the stored row (used later through `$('Database - Create Incident').first().json`).

## Step 5. Audit and state calls to CT-02
`Set - Prepare Event Received Log` creates the CT-02 input (RECEIVED->RECEIVED). CT-02 flow: `Code - Validate Transition` -> `{update, audit, echo}` -> `incidents` UPDATE -> `audit_logs` INSERT -> `{ok, incident_id, new_state, context}`. Then VALIDATING. DB side effects after step 5: `incidents.state='VALIDATING'`; two `audit_logs` rows (`EVENT_RECEIVED`, `VALIDATION_STARTED`).

Branch: `is_valid=false` -> `400 {"status":"rejected","incident_no":..,"errors":[..]}` then REJECTED (`rejection_reason='INVALID_SCHEMA'`, `resolved_at` set).

## Step 6. Accept
`Respond - Accepted (202)`: `{"status":"accepted","incident_no":101,"event_id":"EVT-DEMO-1001","state":"VALIDATING"}`. The caller is released here; the remaining steps run asynchronously. AI_PROCESSING audit `AI_STARTED`.

## Step 7. CRM context
`IF - Customer Email Present?` -> HubSpot search (read-only). `Code - Build AI Context` output (illustrative values): `{ "crm_status":"FOUND", "contact":{"id":"501","lifecycle_stage":"customer","lead_status":null,"company":"Example Co"}, "session_key":"ct-hs-501" }`. Without email: `crm_status:"NO_EMAIL"`, key `ct-ent-Logistics-ORD-55412`. Confirmed gap: `Code - Build AI Context` determines whether the lookup ran with `try { $('HubSpot - Find Contact').first() }`; correct only because n8n throws for a node that did not run.

## Step 8. AI assessment
Input prompt: `Assess this operations event. Event JSON: <normalized> HubSpot status: FOUND HubSpot customer context: {...}`; plus the Brainpower system message and Postgres memory for `ct-hs-501`. Example output (illustrative; parser requires 6 fields):
```json
{ "category":"SUPPLY_CHAIN", "impact_score":8, "summary":"A shipment is delayed five days, putting a customer commitment at risk.",
  "recommended_action":"Logistics to investigate the delay and agree a recovery plan.",
  "affected_departments":["Logistics","Customer Service"], "confidence":0.9 }
```
The agent returns it as `$json.output`. On failure the error output feeds `Code - Build AI Fallback Audit`.
**Potential defects:** (a) the prompt demands additional fields the parser/consumers ignore (F-09); (b) CRITICAL is not recognised by the prompt (F-08).

## Step 9. Scoring and routing
`Code - Compute Priority & Routing`: impact 8; `risk_score = round((3*15 + 8*4) * 1.2) = 92`; priority P1; `sla_due = now + 30 min`; `reasons = ['Severity HIGH', 'Risk score 92 >= 75']`; `requires_approval = true`; `notify = Customer Service, Sales, Finance` (cross map for Logistics merged with AI departments, origin removed). Output keys: `row`, `dispatch_incident` (`{...incident, ...row}`), `requires_approval`, `approval_reasons`, `approval_subject`, `approval_message`, `priority`, `risk_score`.
`Database - Save Classification` updates the incident (11 columns) -> CT-02 `CLASSIFIED` audit with `metadata` (priority, risk, crm status, contact id).

## Step 10. Approval
`IF - Needs Approval?` true -> CT-02 `AWAITING_APPROVAL` -> `Database - Create Approval Record` (`approvals`: `status=PENDING`, placeholder `approver_email`, `reason='Severity HIGH; Risk score 92 >= 75'`) -> Gmail `sendAndWait`. The execution is paused until a click or 45 minutes.
Decision item: approve -> `data.approved=true`. `Code - Read Approval Decision` -> `decision='APPROVED'` + state_update (`APPROVAL_GRANTED`, actor = hard-coded approver placeholder, not the real clicker). `Database - Record Decision` sets `approvals.status='APPROVED'`, `decided_at`. CT-02 sets APPROVED. `Switch - Route Decision` -> output "Approved".
Alternative outcomes: Reject -> REJECTED (+ Slack notice); no click/mail error -> ESCALATED (+ Slack alert). **Flag:** recorded approver identity is the code constant, not the person who clicked.

## Step 11. Dispatch
`Set - Prepare PROCESSING Update` (`previous_state = requires_approval ? 'APPROVED':'CLASSIFIED'`) -> CT-02 PROCESSING. `Set - Prepare Dispatch`: `{incident: dispatch_incident, attempt: 1}`. CT-03: reads `system_config` (`dispatch_channel_id`, `simulate_slack_failure`), builds message, posts to Slack, returns `{ok:true, incident}`. Sample message text begins `:rotating_light: *[P1] INC-101 | Logistics | SHIPMENT_DELAY*`. Success -> CT-02 COMPLETED (`ACTION_DISPATCHED`, `resolved_at` set). **Final state: COMPLETED**; records: 1 incident, 1 approval, 8 audit rows (EVENT_RECEIVED, VALIDATION_STARTED, AI_STARTED, CLASSIFIED, APPROVAL_REQUESTED, APPROVAL_GRANTED, DISPATCH_STARTED, ACTION_DISPATCHED). No HubSpot note (hook disabled).

## Step 12. Failure and retry variant
`simulate_slack_failure=true`: CT-03's Slack node errors after 3 tries; CT-01 takes the error output -> `Code - Build Retry Update` -> RETRYING with `retry_count=1`, `next_retry_at=now+1 min`, `last_error`. CT-05 tick: `Get Retrying Incidents` -> due -> PROCESSING (`RETRY_ATTEMPT`) -> CT-03 with `attempt=2`. Failure: `retry_count=2`, `next_retry_at=now+2 min`, `RETRY_SCHEDULED`. Next failure: attempts 3 >= 3 -> `RETRIES_EXHAUSTED`, ESCALATED, Gmail + Slack. Success at any point -> `SELF_HEALED` -> COMPLETED. Operator: `POST /ops-recover {"incident_no":101}` -> RETRYING, `retry_count=0`.

## Cross-workflow data table
| From | To | Fields |
|---|---|---|
| CT-01 / CT-04 / CT-05 / CT-06 | CT-02 | `incident_id, event_id, execution_id, workflow_name, event, previous_state, new_state, status, error_message, actor, payload_hash, metadata, patch, context` |
| CT-02 | caller | `ok, incident_id, new_state, context` (CT-05 Branch B uses `context`) |
| CT-01 / CT-05 | CT-03 | `incident` (row), `attempt` |
| CT-03 | caller | `ok, incident` or an error |
| CT-02 | CT-07 (disabled) | item containing `incident_id` |

## Flagged mismatches
1. Prompt output fields vs parser (F-09). 2. Approver identity constant vs actual approver. 3. CT-04 execution id only matches CT-01 runs (F-06). 4. CT-02 trusts `previous_state` (F-05). 5. Audit `execution_id` is CT-04's own id in the FAILED path but the failed run's id in the audit-only path.
