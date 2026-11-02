# CT-03 Sub - Dispatch Action

## A. Identity
`CT-03 Sub - Dispatch Action`, ID `JfhHBm4rMmgkcByL`. Purpose: post the incident alert to Slack. Status: IMPLEMENTED, CONFIGURATION REQUIRED. No `errorWorkflow`.

## B. Beginner overview
The messenger. It looks up which Slack channel to use, writes the message, posts it, and says "sent". If Slack refuses, it **fails on purpose** so the caller knows to retry later.

## C. Trigger
Execute Workflow Trigger (`passthrough`). Called by CT-01 (attempt 1) and CT-05 (retries).

## D. Input (illustrative)
```json
{ "incident": { "incident_no": 101, "priority": "P1", "department": "Logistics", "event_type": "SHIPMENT_DELAY",
  "entity_id": "ORD-55412", "severity": "HIGH", "risk_score": 92, "ai_summary": "...", "ai_recommended_action": "...",
  "notify_departments": "Customer Service, Sales, Finance", "sla_due": "2026-10-10T08:45:00Z" }, "attempt": 1 }
```

## E. Nodes
| Node | Type | Logic | Next / failure |
|---|---|---|---|
| Trigger - Receive Dispatch Request | executeWorkflowTrigger | | Get Config |
| Database - Get Dispatch Config | supabase getAll `system_config`, return all; two conditions `key eq dispatch_channel_id` and `key eq simulate_slack_failure`; `matchType` omitted; `alwaysOutputData` | Intended to return both rows | Build Message |
| Code - Build Dispatch Message | code | Maps rows to `config[key]=value`; throws if `incident` missing or `dispatch_channel_id` missing; if `simulate_slack_failure` equals `true` (case-insensitive) uses channel `C0000SIMFAIL`; builds a multi-line mrkdwn message | Slack |
| Slack - Post Incident Alert | slack v2.3 post to channel by id | Retry on fail: 3 tries, 2 s; no `onError` | Return Result; failure fails the workflow |
| Set - Return Result | set raw | `{ok:true, incident}` | caller |

## F. Decision logic
Only one: the simulate switch. Whether the Supabase filter ORs or ANDs the two `key` conditions depends on the node's default `matchType` (UNVERIFIED, F-13). With AND semantics the lookup would return no rows and every dispatch would throw "dispatch_channel_id is missing".

## G. Integration: Slack
Native node, `slackApi` credential ("Slack account 2" by name). Operation: send message to channel (`chat.postMessage`-equivalent). Message includes priority, incident number, department, event type, entity, severity, risk, AI summary, recommended action, notify list, SLA due, attempt and execution id. No ability to message departments individually. Errors: `channel_not_found` (demo), auth, rate limits -> retried 3 times then workflow fails.

## H. Database
Read `system_config` only.

## I. Outputs
One Slack message; `{ok:true, incident}`. "Success" means Slack accepted the post.

## J. Errors
Failure propagates to the caller's error output (CT-01 -> RETRYING, CT-05 -> RETRYING/ESCALATED). Slack post followed by a timeout can lead to duplicate messages on retry.

## K. Configuration
Slack credential; `dispatch_channel_id` (replace `C_REPLACE_ME`); bot invited to channel.

## L. Tests
Normal post; `simulate_slack_failure=true`; missing channel id; missing `incident`; invalid Slack token; duplicate retry.

## M. Troubleshooting
`channel_not_found`: wrong ID, bot not in channel, or simulate switch on. "CT-03 input must contain { incident }": caller mapping problem.

## N. Summary
Looks up where to post, posts it, and raises its hand if it cannot.
