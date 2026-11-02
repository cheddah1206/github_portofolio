# CT-02 Sub - State & Audit Writer

## A. Identity
`CT-02 Sub - State & Audit Writer`, ID `7krMfRZepEVi3L9V`. The only workflow that updates `incidents.state` (per its note; verified: no other workflow writes `state`, except CT-01's initial insert). Status: IMPLEMENTED, with the HubSpot hook **disabled**. No `errorWorkflow`.

## B. Beginner overview
A records clerk. Anyone who wants to change an incident's status hands over a form. The clerk checks the move is allowed, changes the incident, writes a line in the logbook, and hands back a receipt.

## C. Trigger
`Trigger - Receive State Change` (Execute Workflow Trigger, `passthrough`: accepts all incoming data). Called by CT-01, CT-04, CT-05, CT-06. Not directly callable by HTTP.

## D. Input (illustrative)
```json
{ "incident_id": "<uuid>", "event_id": "EVT-DEMO-1001", "execution_id": "123", "workflow_name": "CT-01 ...",
  "event": "CLASSIFIED", "previous_state": "AI_PROCESSING", "new_state": "CLASSIFIED",
  "status": "SUCCESS", "error_message": null, "actor": "system", "payload_hash": "<sha256>",
  "metadata": {}, "patch": {}, "context": null }
```
Required: `incident_id`, `previous_state`, `new_state`. Others default (`status` SUCCESS, `actor` system, `event` `<prev>_TO_<next>`).

## E. Nodes
| Node | Type | Logic | Next / failure |
|---|---|---|---|
| Trigger - Receive State Change | executeWorkflowTrigger | passthrough | Validate |
| Code - Validate Transition | code | Per item: throws on missing ids, unknown state, or illegal transition (`prev !== next` and not in map). Builds `update` = `{...patch, id, state, updated_at}` (`resolved_at` for COMPLETED/REJECTED), `audit`, `echo` | Row Incident Update; a throw fails the sub-execution and the caller |
| Set - Row: Incident Update | set raw | outputs `update` | Update Incident |
| Database - Update Incident | supabase update `incidents`, `id eq` | autoMap | Row Audit Log |
| Set - Row: Audit Log | set raw | `Validate Transition.item.audit` | Write Audit Log |
| Database - Write Audit Log | supabase insert `audit_logs` | autoMap | IF Final |
| IF - Final State Reached? | if | `new_state` in COMPLETED, REJECTED, ESCALATED, FAILED | true: CT-07 hook; false: Return |
| Sub-workflow - Sync Outcome to HubSpot (CT-07) | executeWorkflow, `mode: each`, `waitForSubWorkflow:false`, **disabled**, continue on error | Would start CT-07 | Return |
| Set - Return Result | set raw | `echo` = `{ok, incident_id, new_state, context}` | caller |

## F. Decision logic (transition map in code)
RECEIVED: VALIDATING, ESCALATED, FAILED. VALIDATING: AI_PROCESSING, REJECTED, ESCALATED, FAILED. AI_PROCESSING: CLASSIFIED, ESCALATED, FAILED. CLASSIFIED: PROCESSING, AWAITING_APPROVAL, ESCALATED, FAILED. AWAITING_APPROVAL: APPROVED, REJECTED, ESCALATED, FAILED. APPROVED: PROCESSING, ESCALATED, FAILED. PROCESSING: COMPLETED, RETRYING, ESCALATED, FAILED. RETRYING: PROCESSING, ESCALATED, FAILED. ESCALATED: RETRYING, FAILED. FAILED: RETRYING. COMPLETED, REJECTED: none. Same-to-same allowed. The sticky note omits that PROCESSING etc. may go to FAILED from every open state; the code is authoritative.

**Defect (F-05):** `previous_state` is supplied by the caller and never compared with the stored state; the update filters only on `id`. Two workflows acting on one incident can overwrite each other (e.g. CT-05's stale scan sets ESCALATED while CT-01 is still running and later writes CLASSIFIED).

## G. Integrations
Supabase only (service credential). CT-07 call is disabled.

## H. Database
Update `incidents` by `id` (columns: `state`, `updated_at`, `resolved_at`, plus patch keys `retry_count`, `next_retry_at`, `last_error`, `rejection_reason`). Insert `audit_logs`. The trigger also sets `updated_at`.

## I. Outputs
Updated row, one audit row per item, echo object. The disabled node means the HubSpot side effect never happens.

## J. Errors
Illegal transitions throw (no audit row is written for the rejected attempt). If the incident update succeeds but the audit insert fails, the state changes **without** an audit row (non-atomic). No retries on Supabase nodes.

## K. Configuration
Supabase credential. To enable HubSpot: enable the node, confirm CT-07 target, test input mapping: CT-07 reads `$json.incident_id`, and the item reaching the hook is the inserted `audit_logs` row, which contains `incident_id` (UNVERIFIED at runtime).

## L. Tests
Legal transition; illegal (COMPLETED->PROCESSING); unknown state; missing `incident_id`; same-state audit; patch containing `state` (overridden); audit insert failure; two concurrent writers.

## M. Troubleshooting
"Illegal transition X -> Y": the caller's idea of the current state is wrong. Incident row updated but no audit: check Supabase error on `audit_logs`.

## N. Summary
The gatekeeper for status changes: checks the rule book, updates the incident, writes the log, replies "done".
