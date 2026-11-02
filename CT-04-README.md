# CT-04 Global Error Handler

## A. Identity
`CT-04 Global Error Handler`, ID `maC3TWvbbZsLZAuZ`. Status: IMPLEMENTED; **attached as error workflow only to CT-06 in the exports** (F-04).

## B. Beginner overview
The fire alarm. When another workflow crashes, it shouts in Slack, then checks whether the crash belongs to an open incident and marks it FAILED, or just writes a note in the logbook.

## C. Trigger
Error Trigger (`n8n-nodes-base.errorTrigger`). Runs only for workflows whose settings name it. Per the files, only CT-06 does.

## D. Input
n8n error object (illustrative): `{ "workflow": {"name":"CT-06 ..."}, "execution": {"id":"456","url":"...","lastNodeExecuted":"...","error":{"message":"..."}} }`.

## E. Nodes
| Node | Type | Logic | Next / failure |
|---|---|---|---|
| Error Trigger - Workflow Failed | errorTrigger | | Slack + Find Incident (parallel) |
| Slack - Send Error Alert | slack | text with workflow name, last node, message, URL; hard-coded channel; `continueRegularOutput` | end |
| Database - Find Incident by Execution | supabase getAll `incidents`, `execution_id eq <failed execution id>`, limit 1, `alwaysOutputData`, continue on error | | IF |
| IF - Incident Found and Open? | if | `!!id && state not in COMPLETED, REJECTED, FAILED, ESCALATED` | true: Set FAILED; false: Audit Row |
| Set - Prepare FAILED Update | set | event `WORKFLOW_ERROR`, `new_state FAILED`, `previous_state` = current, patch `last_error` | CT-02 |
| Sub-workflow - Set FAILED (CT-02) | executeWorkflow | | end |
| Set - Build Error Audit Row | set | `{event, status:FAILURE, workflow_name, execution_id (failed one), error_message, actor, metadata.url}` | Log Error Only |
| Database - Log Error Only | supabase insert `audit_logs` | | end |

## F. Decision logic
Linked and open -> FAILED via CT-02 (audit by CT-02 uses CT-04's own `$execution.id`). Otherwise audit only.
**Linking limitation (F-06):** `incidents.execution_id` holds CT-01's execution id. Errors in CT-05 or CT-06 have different execution ids, so they always take the audit-only path, and the incident being worked on is not marked FAILED.

## G. Integrations
Slack (native, hard-coded channel, error text included in the message: may leak internal detail). Supabase.

## H. Database
Read `incidents` by `execution_id` (no index). Insert `audit_logs`. Update `incidents` via CT-02.

## I. Outputs
Slack alert; FAILED incident or audit row.

## J. Errors
Slack failure ignored; lookup failure ignored (falls to audit-only path, which can itself fail unhandled). CT-04 has no error workflow of its own.

## K. Configuration
Slack credential/channel; set this workflow as Error Workflow on CT-01, CT-05, CT-06 (and consider CT-02, CT-03, CT-07).

## L. Tests
Force an error in CT-06 after attaching; force an error in CT-01 mid-flow (linked incident); error with no incident; Slack down; Supabase down.

## M. Troubleshooting
No alert on CT-01 crash: Error Workflow not set in CT-01's settings. Incident not FAILED: execution ids do not match or incident already ESCALATED/FAILED.

## N. Summary
Raises the alarm and files the incident report when something breaks.
