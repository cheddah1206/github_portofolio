# ARCHITECTURE_REVIEW.md: Architecture Review Report

Basis: static review of seven exports + SQL. No runtime evidence.

## Confirmed architecture
Seven workflows: two HTTP entry points (CT-01, CT-06), one schedule (CT-05), one error trigger (CT-04), three sub-workflows (CT-02 state/audit hub, CT-03 Slack dispatch, CT-07 HubSpot, currently unreachable). Supabase holds all state; all state changes route through CT-02.

## Strengths
Single state writer with an explicit transition map; deterministic scoring ("AI assesses, code decides"); AI fallback with audit; schema-validated AI output plus clamping; duplicate guard; retry loop with backoff, exhaustion and human recovery; audit rows for nearly every transition; read-only CRM context; HTML escaping of notes; all workflow columns match the schema; all audit `status` values satisfy the CHECK.

## Findings (ranked)

| ID | Severity | Finding | Practical impact |
|---|---|---|---|
| F-01 | CRITICAL | CT-06 `/ops-recover` has no authentication; `actor` is caller-supplied | Anyone reaching the URL can re-open ESCALATED/FAILED incidents and forge the audit actor |
| F-02 | HIGH | Recovery (ESCALATED -> RETRYING) bypasses approval; CT-05 then dispatches | An approval-timeout incident can be dispatched without any approval |
| F-03 | HIGH | CT-01 `/ops-events` has no authentication (sticky says "Authenticate") | Forged events, flooding, AI cost, Slack spam |
| F-04 | HIGH | `errorWorkflow` set only on CT-06; CT-01 and CT-05 (named in CT-04's note) have none; CT-02/03/07/04 none | Main pipeline and watchdog failures may go unalerted (verify live instance) |
| F-05 | HIGH | CT-02 trusts caller `previous_state`; update filters only on `id`; audit insert after update is not atomic | Races (stale scan vs live run) can overwrite states or lose audit rows |
| F-06 | HIGH | CT-04 links errors via `incidents.execution_id` = CT-01's id only | CT-05/CT-06 errors never mark an incident FAILED |
| F-07 | MEDIUM | CT-02 -> CT-07 node disabled | HubSpot outcome notes never created; docs/notes claim otherwise |
| F-08 | MEDIUM | CRITICAL accepted by validation/scoring but unknown to the Brainpower prompt | AI text for CRITICAL events is outside prompt rules |
| F-09 | MEDIUM | Prompt demands 14 fields; parser requires 6; `human_approval_required`, `incident_required`, `business_impact`, `recommended_actions` ignored; no severity-band check on impact | Spec drift; AI claims cannot be relied on; incident creation is unconditional |
| F-10 | MEDIUM | Placeholder/hard-coded approver emails (two different literals), Gmail recipient, Slack channel IDs; `escalation_channel_id` unused; approver recorded as constant | Approvals may go to the wrong person; audit shows wrong approver |
| F-11 | MEDIUM | Check-then-insert duplicate race; rejected incident blocks corrected resubmission; `payload_hash` never compared | Execution errors on concurrent duplicates; confusing 200 duplicate responses |
| F-12 | MEDIUM | Gmail send failure is treated as an approval timeout (ESCALATED, "45 minutes" message) | Misleading state/audit |
| F-13 | MEDIUM | Supabase multi-condition filters without `matchType` (CT-03 config lookup, CT-05 open-incident scan) | If default is AND: all dispatches fail, stale scan finds nothing (UNVERIFIED) |
| F-14 | MEDIUM | `incident_no` has no UNIQUE/index; CT-06 and humans depend on it | Ambiguous recovery target; slow lookups |
| F-15 | MEDIUM | RLS enabled, no policies; seed `DO UPDATE` and `DROP TABLE` in one script | Anon access blocked (good) but misconfig looks like empty data; re-run destroys data/config |
| F-16 | MEDIUM | CT-05 Gmail/Slack nodes lack `onError`; no error workflow on CT-05 | One notification failure halts the loop |
| F-17 | LOW | `COMPLETED` = Slack message posted; no business action exists | Overstated completion semantics |
| F-18 | LOW | PII/data sharing: payload (incl. email) to Gemini; email to HubSpot; AI summary to Slack/HubSpot; error text in Slack | Privacy review needed |
| F-19 | LOW | No DB CHECKs on severity, score ranges, retry counts; audit mutable; approvals not unique per incident | Data quality/integrity |
| F-20 | LOW | CT-05 schedule interval not explicit; static-data cooldown only persists in production; Slack retry may duplicate messages; audit `execution_id` inconsistent in CT-04 | Operational surprises |
| F-21 | LOW | Chat memory shared per customer/entity | Earlier events can influence later assessments |
| F-22 | LOW | CT-06 has no body validation | Unclear errors |

## Integration gaps
CT-07 unreachable; `approvals` never read; `system_config.escalation_channel_id` unused; no per-department notification (only a text list); no business action beyond Slack; chat-memory table not in schema.

## Database risks
F-14, F-15, F-19; `ON DELETE SET NULL` for audits (good) but audit rows are mutable; no transition enforcement in DB.

## Security risks
F-01, F-03, F-10, F-15, F-18; service-role key handling [DEPLOY]; prompt injection mitigated by design (no tools, schema, clamping, code-side decisions) but the AI summary still reaches Slack/HubSpot.

## Reliability risks
F-04, F-05, F-06, F-12, F-13, F-16, duplicate Slack posts on retry, 1-minute demo constants.

## Recommended fixes (priority order)
1. Authenticate both webhooks (Header Auth/HMAC). 2. Add `WHERE state = previous_state` guard in CT-02 and make update+audit atomic (Postgres function). 3. Attach CT-04 everywhere; store the failing workflow's execution context or link by incident number. 4. Require re-approval (or a distinct state) when recovering approval-timeout incidents; derive `actor` from authenticated identity. 5. Resolve the CRITICAL/prompt/parser mismatch. 6. Replace placeholders; use `system_config` for channels/recipients. 7. Set explicit `matchType`. 8. Enable or remove the CT-07 hook. 9. Add DB constraints and unique `incident_no`. 10. Handle unique-conflict gracefully.

## Outstanding questions
Is CT-04 attached to CT-01/CT-05 in the live instance? Which Supabase key is in the credential? Default `matchType` in the installed node version? Schedule interval in CT-05? Gemini model used? Is the chat-memory table present? Should CRITICAL be a supported severity? Should recovery re-request approval? Who are the real approvers?

## Documentation completeness
Complete for static analysis; runtime behaviour, credential validity and live configuration are UNVERIFIED.

## Deliverables checklist
| Item | Status |
|---|---|
| README.md | Done |
| 7 workflow READMEs (CT-01..CT-07) in `docs/workflow-readmes/` | Done |
| ARCHITECTURE.md | Done |
| architecture-diagram.md (A, B, C) | Done |
| API_AND_INTEGRATIONS.md | Done |
| DATABASE_SCHEMA.md | Done |
| DATA_FLOW.md | Done |
| TESTING_AND_QA.md | Done |
| SETUP_AND_OPERATIONS.md | Done |
| PROJECT_STRUCTURE.md, examples/sample-event.json | Done (`sample-test-cases.json`, `schema.sql`, `.env.example` file not generated) |
| Workflows reviewed | All 7 read in full |
| Not done | Runtime verification, rendering Mermaid in a viewer |
