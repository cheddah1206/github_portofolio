# CT-07 Sub - HubSpot Outcome Sync

## A. Identity
`CT-07 Sub - HubSpot Outcome Sync`, ID `Vv0K3JHniLnTpKqV`. Status: IMPLEMENTED but **not invoked**: its only caller node in CT-02 is `disabled: true` (F-07). No `errorWorkflow`.

## B. Beginner overview
After an incident ends, if it concerns a known customer, add a note to that customer's HubSpot record and log what happened.

## C. Trigger
Execute Workflow Trigger (passthrough). Intended caller: CT-02 for COMPLETED, REJECTED, ESCALATED, FAILED (fire-and-forget).

## D. Input
`{ "incident_id": "<uuid>" }` (illustrative). Requires `incident_id` in the first item.

## E. Nodes
| Node | Type | Logic | Next |
|---|---|---|---|
| Trigger - Receive Final State | executeWorkflowTrigger | | Get Incident |
| Database - Get Incident | supabase getAll `incidents`, `id eq incident_id`, limit 1 | | IF Email |
| IF - Customer Email Present? | if | regex on `payload.customer_email` | true: Find Contact; **false: not connected (silent end, no audit)** |
| HubSpot - Find Contact | hubspot search | email EQ lowercased, limit 1, retry 3x/2 s, continue on error, `alwaysOutputData` | IF Found |
| IF - Contact Found? | if | `!$json.error && !!$json.id` | true: Build Note; false: Build Audit Row |
| Code - Build Note Request | code | HTML-escaped note: incident no, priority, outcome state, department/type/entity, AI summary, last error (200 chars) | Create Note |
| HubSpot - Create Note | httpRequest POST `https://api.hubapi.com/crm/v3/objects/notes` | body `{properties:{hs_timestamp, hs_note_body}, associations:[{to:{id}, types:[{HUBSPOT_DEFINED, 202}]}]}`; HubSpot credential; timeout 15 s; continue on error | Build Audit Row |
| Code - Build Audit Row | code | Determines outcome: `HUBSPOT_NOTE_CREATED` (SUCCESS), `HUBSPOT_SYNC_FAILED` (FAILURE), `HUBSPOT_CONTACT_NOT_FOUND` (WARNING) using whether Build Note ran | Log |
| Database - Log HubSpot Sync | supabase insert `audit_logs` | | end |

## F. Decision logic
Email present -> contact found -> note -> audit. Contacts are never created. No-email path writes nothing (the sticky says "nothing to do").

## G. Integrations
HubSpot native node (search) and HTTP Request for notes (association type 202 = note-to-contact, HubSpot-defined; verify against your portal). Authentication: HubSpot app token. The HTTP node could be replaced by a native HubSpot engagement operation only after checking that the node version supports note creation with associations (not assessed). Personal data (customer email, AI summary) is sent to HubSpot.

## H. Database
Read `incidents`; insert `audit_logs` (`actor system:hubspot-sync`, state unchanged).

## I. Outputs
HubSpot note, audit row.

## J. Errors
HubSpot failures are captured and audited; Supabase failures are not handled. Because the caller is non-blocking, failures cannot affect state changes.

## K. Configuration
HubSpot private app token with contacts read and notes write scopes; enable CT-02 hook; link both nodes.

## L. Tests
Manual run with a completed incident; no email; email not in HubSpot; HubSpot 401/429; note API failure; duplicate invocation (would create two notes; no idempotency key).

## M. Troubleshooting
Nothing happens: hook disabled. `CONTACT_NOT_FOUND` audit: email mismatch/casing. Failure audit: scopes/token.

## N. Summary
A courtesy update to the CRM after the incident closes, currently switched off.
