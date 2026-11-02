# CT-06 Human Recovery Endpoint

## A. Identity
`CT-06 Human Recovery Endpoint`, ID `tD6epzMZZTjPTWuy`. Status: IMPLEMENTED; settings: `errorWorkflow` = CT-04, `callerPolicy: workflowsFromSameOwner`. **Webhook has no authentication** (F-01).

## B. Beginner overview
A "try again" button for an operator. Give it an incident number; if the incident is stuck or failed it is put back in the retry tray; otherwise it says "no".

## C. Trigger
`Webhook - Receive Recovery Request`: `POST /ops-recover`, `responseMode: responseNode`.

## D. Input (illustrative)
```json
{ "incident_no": 101, "actor": "ops.lead@example.com", "note": "Slack channel fixed" }
```
`incident_no` is the only functional requirement; no validation node exists (F-22).

## E. Nodes
| Node | Type | Logic | Next |
|---|---|---|---|
| Webhook - Receive Recovery Request | webhook | | Get Incident |
| Database - Get Incident | supabase getAll `incidents`, `incident_no eq body.incident_no`, limit 1, `alwaysOutputData` | | IF |
| IF - Incident Recoverable? | if | `['ESCALATED','FAILED'].includes($json.state)` | true: Prepare; false: 409 |
| Set - Prepare RETRYING Update | set | `HUMAN_RECOVERY`, new state RETRYING, `actor = body.actor || 'human'`, `metadata.note`, patch `retry_count:0, next_retry_at:now, last_error:null` | CT-02 |
| Sub-workflow - Set RETRYING (CT-02) | executeWorkflow | | Respond 200 |
| Respond - Recovered (200) | respond | `{status:'recovering', incident_no, message}` | end |
| Respond - Not Recoverable (409) | respond | `{status:'not_recoverable', state: state ?? 'NOT_FOUND'}` | end |

## F. Decision logic
Recoverable only when current state is ESCALATED or FAILED. Everything else (including unknown number) -> 409, no change.
**Control gap (F-02):** an incident escalated because approval timed out can be recovered to RETRYING and CT-05 will dispatch it; the approval gate is never re-applied.

## G. Integration
Supabase read; CT-02. Authentication: none. The `actor` value is caller-supplied, unverified identity, and lands in `audit_logs.actor`.

## H. Database
Read `incidents`; update through CT-02. `incident_no` has no index/unique constraint (F-14).

## I. Outputs
HTTP 200/409; state RETRYING, retry counter reset; audit row `HUMAN_RECOVERY`. Actual retry occurs only when CT-05 is active.

## J. Errors
Any failure -> n8n error -> CT-04 (Slack + audit). Webhook caller receives an n8n error response.

## K. Configuration
Network protection or authentication (Header Auth, IP allow-list, or reverse proxy), CT-02 target, CT-04 as error workflow (already set).

## L. Tests
Recover ESCALATED; recover FAILED; recover COMPLETED (409); unknown number (409); missing body; repeated call (second call returns 409 because the state is now RETRYING); unauthenticated call.

## M. Troubleshooting
Always 409/NOT_FOUND: number wrong or RLS/anon key. 200 but nothing retried: CT-05 inactive or `next_retry_at` filter.

## N. Summary
An operator's reset button that needs a lock on the door.
