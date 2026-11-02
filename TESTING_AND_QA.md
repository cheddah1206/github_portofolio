# TESTING_AND_QA.md

**No test has been executed.** "Expected behavior" is derived from the JSON. Where the implementation is incomplete the row says **GAP** with intended result and needed correction. Use a staging Supabase project; never test against production data.

Helper: `POST <n8n>/webhook/ops-events` with `examples/sample-event.json` (adjust fields). Replace host.

| Test ID | Scenario | Input | Expected Behavior | Database Verification | Pass Criteria |
|---|---|---|---|---|---|
| T-01 | Normal MEDIUM event, no approval | severity MEDIUM, valid fields, AI confidence >= 0.6, risk < 75 | 202; Slack post; state COMPLETED | incident `COMPLETED`, `resolved_at` set; audit chain RECEIVED..COMPLETED | State COMPLETED, Slack message received |
| T-02 | HIGH event needing approval, approved | severity HIGH | 202; approval email; after Approve: Slack; COMPLETED | `approvals.status=APPROVED`, `decided_at` set | Approval row and COMPLETED |
| T-03 | Approval rejected | click Reject | REJECTED, Slack notice | `rejection_reason=HUMAN_REJECTED`, approval REJECTED | No dispatch Slack alert |
| T-04 | Approval timeout (45 min) | no click | ESCALATED + Slack alert; approval `TIMEOUT` | `last_error` text | State ESCALATED. Note the CT-05 stale scan limit is 60 min, so CT-01 acts first |
| T-05 | Missing required field | omit `entity_id` | 400 with `errors[]`; state REJECTED | row with `rejection_reason=INVALID_SCHEMA` | 400 and REJECTED |
| T-06 | Invalid payload | `payload: [1,2]` | 400 | same | error text "payload must be a JSON object" |
| T-07 | Missing/malformed timestamp | `timestamp:"yesterday"`; omit | 400; `event_timestamp` null | row stored | 400 |
| T-08 | Future timestamp | now + 10 min | 400 | | 400; now + 4 min accepted (boundary) |
| T-09 | Boundary IDs | `event_id` length 2, 3, 64, 65 | 2 and 65 -> 400; 3 and 64 -> accepted | invalid ones stored as `INVALID-<exec id>` | boundaries as stated |
| T-10 | Severity levels | LOW, MEDIUM, HIGH, CRITICAL, `urgent` | `urgent` -> 400. CRITICAL accepted; AI prompt does not recognise it. **GAP F-08**: intended AI handling not defined | `severity` column | Approval required for HIGH/CRITICAL |
| T-11 | Duplicate event_id (sequential) | same body twice | second -> 200 duplicate | single row | one incident, no second Slack |
| T-12 | Duplicate (concurrent) | two parallel posts | **GAP F-11:** one may fail on the unique constraint (execution error). Intended: both get 200/202 safely. Correction: upsert / handle conflict | one row | no duplicate row; inspect failed executions |
| T-13 | Corrected resubmission | fix a rejected event, same id | currently 200 "duplicate". **GAP F-11** | | document or change policy |
| T-14 | AI failure | invalid Gemini key | `AI_FALLBACK` audit; `ai_failed=true`; approval required | audit `AI_FALLBACK` | fallback path completes |
| T-15 | Invalid model output | force parser failure (prompt tweak) | same as T-14 | | no crash |
| T-16 | Low confidence | stub AI to 0.4 | approval required (`AI confidence 0.40 < 0.60`) even for LOW | `requires_approval=true` | approval row created |
| T-17 | Impact out of band | AI returns score 9 for LOW | **GAP F-09:** accepted (only clamped 1-10) | `ai_impact_score` | document behaviour |
| T-18 | Slack failure then recovery | `simulate_slack_failure=true`, then false | RETRYING; CT-05 retry succeeds; `SELF_HEALED` | `retry_count`, audit | COMPLETED after flip |
| T-19 | Retry limit exceeded | keep switch true | after two CT-05 retries ESCALATED; Gmail + Slack | `retry_count=3`, `RETRIES_EXHAUSTED` | state ESCALATED |
| T-20 | Human recovery | `POST /ops-recover {"incident_no":N}` on ESCALATED | 200; RETRYING; retry_count 0 | audit `HUMAN_RECOVERY` | then retried |
| T-21 | Recovery of final incident | on COMPLETED | 409 | no change | unchanged row |
| T-22 | Recovery after approval timeout | ESCALATED by T-04, then recover | **GAP F-02:** dispatches without approval | | Intended: re-request approval |
| T-23 | Unauthenticated call | no credentials | **GAP F-01/F-03:** accepted | | Intended: 401/403 |
| T-24 | Database outage | block Supabase | CT-01 execution error; with CT-04 attached: Slack alert; audit may fail too | | alert received (only if CT-04 attached, F-04) |
| T-25 | DB constraint violation | force bad state via SQL | CHECK rejects | | error surfaced |
| T-26 | Audit logging failure | break `audit_logs` insert | **GAP:** state already changed (non-atomic) | state vs audit mismatch | Intended: transactional write |
| T-27 | Duplicate workflow execution | trigger CT-05 twice at once | possible double retry / duplicate Slack (no locking) **GAP F-05** | audit | document |
| T-28 | Partial failure | kill n8n during PROCESSING | CT-05 escalates after 10 min | `STALE_INCIDENT` audit | ESCALATED |
| T-29 | Stale AWAITING_APPROVAL | pause CT-01 wait; age row 61 min | `APPROVAL_TIMEOUT` ESCALATED | | ESCALATED + Slack |
| T-30 | Systemic alert | 3 HIGH incidents in one department < 15 min | Slack + `SYSTEMIC_ALERT` audit; 2nd alert suppressed 30 min | audit row | one alert (production run) |
| T-31 | Global error handler | break CT-06 after body | CT-04 Slack alert; audit-only | `WORKFLOW_ERROR` audit | alert + row |
| T-32 | Error in CT-01 | induce error | Alert only if CT-04 attached to CT-01 | FAILED if linked | verify setting |
| T-33 | HubSpot lookup down | bad token | `LOOKUP_FAILED`; AI continues | metadata `crm_status` | pipeline completes |
| T-34 | HubSpot sync | enable CT-02 hook; complete incident with email | note + `HUBSPOT_NOTE_CREATED` audit | audit | requires enabling (F-07) |
| T-35 | Recovery after temporary outage | Slack down 10 min then restore | retries succeed or escalate per attempts | | state per T-18/19 |

## SQL checks
```sql
-- Incident and audit chain
SELECT incident_no, event_id, state, priority, risk_score, requires_approval, ai_failed, retry_count, last_error
FROM incidents WHERE event_id = 'EVT-DEMO-1001';

SELECT id, created_at, workflow_name, event, previous_state, new_state, status, actor
FROM audit_logs WHERE event_id = 'EVT-DEMO-1001' ORDER BY id;

-- Approvals
SELECT a.status, a.requested_at, a.decided_at, i.incident_no
FROM approvals a JOIN incidents i ON i.id = a.incident_id ORDER BY a.requested_at DESC;

-- Retry counts / unresolved
SELECT incident_no, state, retry_count, max_retries, next_retry_at, last_error
FROM incidents WHERE state IN ('RETRYING','ESCALATED','FAILED') ORDER BY updated_at DESC;

-- State without audit trail (consistency check)
SELECT i.incident_no, i.state FROM incidents i
WHERE NOT EXISTS (SELECT 1 FROM audit_logs a WHERE a.incident_id = i.id AND a.new_state = i.state);

-- Duplicate approvals
SELECT incident_id, count(*) FROM approvals GROUP BY 1 HAVING count(*) > 1;
```
