# CT-05 Watchdog

## A. Identity
`CT-05 Watchdog`, ID `r0fuIsVxDFgJv6lO`. Purpose: three independent checks on a schedule. Status: IMPLEMENTED, CONFIGURATION REQUIRED. **No `errorWorkflow` in export** (F-04).

## B. Beginner overview
A night watchman who walks the building on a timer. A: re-tries messages that failed. B: flags incidents that have sat untouched too long. C: shouts if one department has too many serious incidents at once.

## C. Trigger
`Schedule - Every 5 Minutes` (scheduleTrigger 1.2). The rule is `{field:'minutes'}` with no explicit interval value in the JSON; the node name and sticky say 5 minutes (1 minute for demo). **Confirm the interval in the n8n UI** (UNVERIFIED). One tick fans out to three branches in parallel.

## D. Input
None (reads the database).

## E. Nodes
### Branch A: retry loop
| Node | Type | Logic | Next |
|---|---|---|---|
| Database - Get Retrying Incidents | supabase getAll `incidents`, `state eq RETRYING`, return all | | Select Due |
| Code - Select Due Retries | code | keep rows where `next_retry_at` is null or <= now | Loop |
| Loop - Each Due Retry | splitInBatches v3 | batch size default; output 1 = next item, output 0 (done) unconnected | Prepare PROCESSING |
| Set - Prepare PROCESSING Update + Sub-workflow - Set PROCESSING (CT-02) | | `RETRY_ATTEMPT` RETRYING->PROCESSING, `context` = full row | Prepare Re-Dispatch |
| Set - Prepare Re-Dispatch | set | `{incident: <loop row>, attempt: retry_count+1}` | CT-03 |
| Sub-workflow - Re-Dispatch Action (CT-03) | executeWorkflow, error output | success -> Self-Healed; error -> Build Failure Update | |
| Set - Prepare Self-Healed Update + Sub-workflow - Set COMPLETED (CT-02) | | `SELF_HEALED` PROCESSING->COMPLETED, patch `last_error:null` | back to Loop |
| Code - Build Failure Update | code | `attempts = retry_count+1`; `max = max_retries || 3`; exceeded when `attempts >= max`; if not exceeded `next_retry_at = now + attempts x 1 min` (demo constant); event `RETRY_SCHEDULED` or `RETRIES_EXHAUSTED`; state RETRYING or ESCALATED | Record Failure |
| Sub-workflow - Record Failure (CT-02) | | | IF Exhausted |
| IF - Retries Exhausted? | if (strict) | `Build Failure Update.new_state == 'ESCALATED'` | true: Gmail; false: Loop |
| Gmail - Send Escalation Email | gmail send | hard-coded recipient; includes recover instruction `POST /webhook/ops-recover` | Slack |
| Slack - Send Escalation Alert | slack | hard-coded channel | Loop |
### Branch B: stale scan
| Database - Get Open Incidents | supabase getAll, 7 `state eq` conditions (RECEIVED...PROCESSING), `matchType` omitted | relies on default OR (F-13) | Find Stale |
| Code - Find Stale Incidents | code | limits 10 min; AWAITING_APPROVAL 60; builds ESCALATED update with `APPROVAL_TIMEOUT` or `STALE_INCIDENT`, `context` with `age_min` | Set ESCALATED |
| Sub-workflow - Set ESCALATED (CT-02) | | | Slack Stale Alert |
| Slack - Send Stale Alert | slack | uses `$json.context.*` from CT-02's echo | end |
### Branch C: systemic detection
| Database - Get Recent Incidents | supabase getAll, `created_at >= now-15 min` | | Detect |
| Code - Detect Systemic Failures | code | group HIGH/CRITICAL by department; alert if >= 3 and not alerted in last 30 min (`$getWorkflowStaticData('global')`, persists only in production runs) | Slack |
| Slack - Send Systemic Alert | slack | lists incident numbers | Build Audit Row |
| Set - Build Audit Row / Database - Log Systemic Alert | | `SYSTEMIC_ALERT` WARNING audit, no `incident_id` | end |

## F. Decision logic
Retry timing; exhaustion rule: CT-01 sets `retry_count=1` on the first failure, so with `max_retries=3` there are **two** CT-05 retries after the initial dispatch (attempts become 2, then 3 -> ESCALATED). Stale limits as above. Systemic: threshold 3, window 15 min, cooldown 30 min. In Branch A, `$('Loop - Each Due Retry').first()` is used for the current row; correct only because batch size is 1 (UNVERIFIED in-loop behaviour).

## G. Integrations
Slack, Gmail (OAuth2), Supabase, CT-02, CT-03. Gmail and Slack nodes have no `onError`: a failure there stops the loop for remaining incidents in that tick.

## H. Database
Read `incidents` (three queries). Update via CT-02 (`state`, `retry_count`, `next_retry_at`, `last_error`). Insert `audit_logs` (systemic alert).

## I. Outputs
State changes, audit rows, Gmail escalation, Slack alerts.

## J. Errors
CT-03 failures routed to error output; other node errors fail the execution; stuck PROCESSING items are escalated by Branch B after 10 min. Retry re-sends the whole Slack message and can duplicate it.

## K. Configuration
Schedule interval, Slack/Gmail/Supabase credentials, escalation recipient, channel ID, `BASE_MINUTES` (1 -> 5 for production), recovery URL in the email (it uses `/webhook/ops-recover`).

## L. Tests
Retry succeeds after setting `simulate_slack_failure=false`; retries exhausted; stale PROCESSING; stale AWAITING_APPROVAL; three HIGH incidents same department; cooldown; Gmail failure during escalation; two due incidents in one tick.

## M. Troubleshooting
Nothing retried: schedule inactive, `next_retry_at` in future, or Supabase filter semantics. Repeated systemic alerts: static data not persisting (test runs).

## N. Summary
It re-tries what failed, nags about forgotten items, and warns when many serious problems cluster.
