# DATABASE_SCHEMA.md

Source: the SQL supplied with the task (treated as the **intended** definition). **UNVERIFIED:** that it has been applied to a live Supabase project. Workflow reader/writer claims are from the JSON.

## Security review (read first)

> **RLS is enabled on all four tables and no policies are defined.** For roles subject to RLS (`anon`, `authenticated`) this means **no rows are visible or writable** (reads return empty, writes are rejected). The `service_role` key bypasses RLS, so n8n works only if its Supabase credential uses that key (CONFIGURATION REQUIRED, UNVERIFIED). Consequences:
> - If n8n is given an `anon` key, duplicate checks return nothing and inserts fail, so behaviour looks like "empty database".
> - If it uses the service-role key, RLS gives no protection against n8n bugs, and that key grants full database access. **Never expose it in browser code, public repos, client apps or documentation.**
> - Recommendation (not in the SQL): keep service-role use server-side in n8n only; add **no** public policies. If a dashboard later reads data, add narrowly scoped `SELECT` policies for `authenticated` on `incidents` and `audit_logs` only, and none on `system_config`.

Script hazards: it runs `DROP TABLE ... CASCADE` on all four tables (**destroys data if re-run**), and the seed uses `ON CONFLICT DO UPDATE`, so a re-run **overwrites** a configured `dispatch_channel_id` with `C_REPLACE_ME`.

## Concepts

| Item | Purpose |
|---|---|
| `pgcrypto` / `gen_random_uuid()` | Generates UUID primary keys for `incidents.id`, `approvals.id` |
| JSONB `payload`, `metadata` | Stores event body and audit metadata without fixed schema |
| `event_id UNIQUE` | DB-level duplicate protection; not full idempotency (see below) |
| `incident_no` identity | Human-friendly number (`INC-<n>`); **not unique-constrained** (F-14) |
| Indexes | `idx_incidents_state`, `idx_incidents_dept_time (department, created_at DESC)`, `idx_audit_incident (incident_id, created_at DESC)` |
| `set_updated_at` triggers | Force `updated_at = now()` on UPDATE of `incidents` and `system_config` |
| RLS | See above |
| `NOTIFY pgrst, 'reload schema'` | Refreshes the PostgREST schema cache so the Supabase node sees new tables |
| Seeds | `simulate_slack_failure=false`, `dispatch_channel_id=C_REPLACE_ME`, `escalation_channel_id=C_REPLACE_ME` |

Relationships: incident 1..n audit_logs (`ON DELETE SET NULL`, audit survives incident deletion); incident 1..n approvals (`ON DELETE CASCADE`).

## Table: `public.incidents`

Purpose: one row per received event; holds state, AI result, scoring and retry bookkeeping.

| Column | Type | Key | Null / default | Notes |
|---|---|---|---|---|
| id | UUID | PK | `gen_random_uuid()` | |
| incident_no | BIGINT | | identity ALWAYS | Do not insert |
| event_id | TEXT | UNIQUE | NOT NULL | |
| event_timestamp | TIMESTAMPTZ | | null | null for invalid timestamps |
| department, event_type, severity, entity_id | TEXT | | null | No CHECK on severity |
| payload | JSONB | | NOT NULL `{}` | |
| payload_hash | TEXT | | null | SHA-256 of raw body (CT-01) |
| execution_id | TEXT | | null | CT-01 execution id; used by CT-04 lookup |
| state | TEXT | | NOT NULL `RECEIVED` | CHECK 12 values |
| rejection_reason | TEXT | | null | `INVALID_SCHEMA`, `HUMAN_REJECTED` |
| ai_category, ai_summary, ai_recommended_action | TEXT | | null | |
| ai_impact_score, ai_confidence, risk_score | NUMERIC | | null | no range CHECK |
| ai_failed, requires_approval | BOOLEAN | | NOT NULL false | |
| priority | TEXT | | null | P1..P4 |
| notify_departments | TEXT | | null | comma-separated |
| sla_due, next_retry_at, resolved_at | TIMESTAMPTZ | | null | `resolved_at` only for COMPLETED/REJECTED |
| retry_count / max_retries | INT | | NOT NULL 0 / 3 | no relation check |
| last_error | TEXT | | null | |
| created_at / updated_at | TIMESTAMPTZ | | NOT NULL `now()` | |

Writers: CT-01 (insert; update classification), CT-02 (state + patch). Readers: CT-01 (duplicate check), CT-04 (by `execution_id`), CT-05 (by `state`, by `created_at`), CT-06 (by `incident_no`), CT-07 (by `id`).

Example (fictional): `{id:"6f1c...", incident_no:101, event_id:"EVT-DEMO-1001", department:"Logistics", severity:"HIGH", state:"COMPLETED", priority:"P1", risk_score:92, retry_count:0}`

## Table: `public.audit_logs`

Purpose: append-only trail (append-only by convention only; nothing prevents UPDATE/DELETE).

Columns: `id` BIGINT identity PK; `incident_id` UUID FK (SET NULL); `event_id`, `execution_id`, `workflow_name` TEXT; `event` TEXT NOT NULL; `previous_state`, `new_state` TEXT (no CHECK); `status` NOT NULL default `SUCCESS` CHECK in (`SUCCESS`,`FAILURE`,`WARNING`,`INFO`); `error_message` TEXT; `actor` NOT NULL default `system`; `payload_hash` TEXT; `metadata` JSONB NOT NULL `{}`; `created_at`.

Writers: CT-02 (every state change), CT-04 (error-only), CT-05 (systemic alert), CT-07 (sync result, never reached). Readers: none in the workflows. **Verified:** all `status` values used by the workflows are within the CHECK.

Example: `{event:"RETRY_SCHEDULED", previous_state:"PROCESSING", new_state:"RETRYING", status:"FAILURE", actor:"system:watchdog", error_message:"channel_not_found"}`

Not audited: classification column writes (only the `CLASSIFIED` transition), approval row writes, HubSpot read, Slack posts (apart from the resulting state change).

## Table: `public.approvals`

Purpose: approval request/decision. Columns: `id` UUID PK; `incident_id` UUID NOT NULL FK (CASCADE); `status` NOT NULL default `PENDING` CHECK (`PENDING`,`APPROVED`,`REJECTED`,`TIMEOUT`); `approver_email`, `reason`, `execution_id` TEXT; `requested_at` NOT NULL `now()`; `decided_at`.

Writers: CT-01 only (insert PENDING with placeholder approver email; update status + `decided_at`). Readers: none. No uniqueness per incident; if CT-05 escalates a stale AWAITING_APPROVAL incident the row stays `PENDING`. Example: `{status:"PENDING", approver_email:"approver@example.com", reason:"Severity HIGH; Risk score 92 >= 75"}`.

## Table: `public.system_config`

Key/value settings. `key` TEXT PK; `value` TEXT NOT NULL; `description`; `updated_at` (trigger). Reader: CT-03 (`dispatch_channel_id`, `simulate_slack_failure`). Writers: none in workflows (manual SQL/UI). `escalation_channel_id` is **not read by any workflow**. Values are strings (`'true'` compared via `String(...).toLowerCase()==='true'`).

## State model

States per CHECK: `RECEIVED, VALIDATING, AI_PROCESSING, CLASSIFIED, AWAITING_APPROVAL, APPROVED, PROCESSING, COMPLETED, RETRYING, FAILED, ESCALATED, REJECTED`. The CHECK proves nothing about transitions; CT-02's Code node enforces the map below **at the application level**, using the caller-supplied `previous_state`.

### State transitions

| Current State | Event or Condition | Next State | Responsible Workflow | Evidence or Status |
|---|---|---|---|---|
| (none) | Valid or invalid event stored | RECEIVED | CT-01 | Verified: `Set - Build Incident Row` (`state:'RECEIVED'`) |
| RECEIVED | `EVENT_RECEIVED` audit only | RECEIVED | CT-01 via CT-02 | Verified (same-state allowed) |
| RECEIVED | `VALIDATION_STARTED` | VALIDATING | CT-01 | Verified |
| VALIDATING | `is_valid=false` | REJECTED | CT-01 | Verified, `rejection_reason=INVALID_SCHEMA` |
| VALIDATING | `AI_STARTED` | AI_PROCESSING | CT-01 | Verified |
| AI_PROCESSING | `AI_FALLBACK` | AI_PROCESSING | CT-01 | Verified (audit only) |
| AI_PROCESSING | Classification saved | CLASSIFIED | CT-01 | Verified |
| CLASSIFIED | `requires_approval=true` | AWAITING_APPROVAL | CT-01 | Verified |
| CLASSIFIED | `requires_approval=false` | PROCESSING | CT-01 | Verified (`DISPATCH_STARTED`) |
| AWAITING_APPROVAL | approver approves | APPROVED | CT-01 | Verified in code; runtime UNVERIFIED |
| AWAITING_APPROVAL | approver rejects | REJECTED | CT-01 | Verified, `HUMAN_REJECTED` |
| AWAITING_APPROVAL | timeout / mail failure | ESCALATED | CT-01 | Verified |
| AWAITING_APPROVAL | no progress > 60 min | ESCALATED | CT-05 | Verified |
| APPROVED | dispatch starts | PROCESSING | CT-01 | Verified |
| PROCESSING | Slack ok | COMPLETED | CT-01 / CT-05 | Verified |
| PROCESSING | Slack failed (first) | RETRYING | CT-01 | Verified, `retry_count=1` |
| RETRYING | due retry begins | PROCESSING | CT-05 | Verified |
| PROCESSING | retry failed, attempts < max | RETRYING | CT-05 | Verified |
| PROCESSING | retry failed, attempts >= max | ESCALATED | CT-05 | Verified |
| RECEIVED..PROCESSING (7 states) | no progress > 10 min | ESCALATED | CT-05 | Verified |
| Open state (not ESCALATED/FAILED/COMPLETED/REJECTED) | error linked by `execution_id` | FAILED | CT-04 | Verified for CT-01 executions; CT-05/CT-06 not linkable (F-06) |
| ESCALATED / FAILED | `/ops-recover` | RETRYING | CT-06 | Verified; skips approval (F-02) |
| ESCALATED | to FAILED | FAILED | none | Allowed by CT-02 map; no caller found: UNVERIFIED |
| RETRYING | error linked by `execution_id` | FAILED | CT-04 | Allowed and covered by CT-04's open check; ESCALATED from RETRYING has no caller (stale scan excludes RETRYING) |
| COMPLETED, REJECTED | any | | | Final per CT-02 |

## Compatibility with workflow operations

All columns written by the workflows exist in the SQL (checked for CT-01 insert/update, CT-02 patch/audit, CT-04, CT-05, CT-07). **Verified by reading, not execution.** Note CT-02's update also sends `id` in the body (same value).

## Idempotency and duplicates

`event_id UNIQUE` prevents two rows with the same id. End-to-end idempotency is **not** achieved: CT-01 does check-then-insert (concurrent duplicates can hit the constraint and error), `payload_hash` is stored but never compared, and CT-05 can retry a dispatch whose first Slack post actually succeeded (Slack node times out after posting), producing duplicate Slack messages.

## Recommendations (not in the supplied SQL)

| Area | Suggestion |
|---|---|
| Severity | `CHECK (severity IN ('LOW','MEDIUM','HIGH','CRITICAL'))` (decide on CRITICAL, F-08) |
| Ranges | `ai_confidence BETWEEN 0 AND 1`, `ai_impact_score BETWEEN 1 AND 10`, `risk_score BETWEEN 0 AND 100` |
| Retries | `CHECK (retry_count >= 0 AND max_retries >= 0)` |
| State transitions | Trigger enforcing the CT-02 map; or a Postgres function doing `UPDATE ... WHERE id=$1 AND state=$prev` returning rows |
| Duplicate approvals | `CREATE UNIQUE INDEX ON approvals(incident_id) WHERE status='PENDING'` |
| Audit integrity | Revoke UPDATE/DELETE; trigger raising on modification; CHECK on `previous_state/new_state` |
| incident_no | `UNIQUE` constraint (CT-06 relies on it) |
| Lookups | Index `incidents(execution_id)` |
| Config | Restrict `system_config` to service role; do not seed with `DO UPDATE` |
| Concurrency | Optimistic update guard (`WHERE state = previous_state`), see F-05 |

## Common queries

```sql
SELECT incident_no, event_id, state, priority, retry_count, next_retry_at, last_error
FROM incidents ORDER BY created_at DESC LIMIT 20;

SELECT created_at, workflow_name, event, previous_state, new_state, status, actor
FROM audit_logs WHERE incident_id = '<uuid>' ORDER BY created_at, id;

SELECT * FROM incidents WHERE state IN ('RETRYING','ESCALATED','FAILED');
```

Troubleshooting: empty results with a valid row => RLS/anon key; Supabase node does not list a table => run `NOTIFY pgrst, 'reload schema';`; inserts failing on `event_id` => duplicate; invalid state => CHECK violation.
