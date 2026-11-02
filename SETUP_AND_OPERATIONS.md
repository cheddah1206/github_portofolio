# SETUP_AND_OPERATIONS.md

**Legend:** [FILES] = derivable from the uploaded files. [DEPLOY] = requires access to your actual n8n/Supabase/Slack/Gmail/HubSpot accounts. Nothing below has been executed by the author of this document.

## 1. Prerequisites
[FILES] n8n with LangChain/AI nodes (workflows use AI Agent v2.2, Gemini model, Postgres memory, `sendAndWait` in Gmail, Execute Workflow v1.2, Supabase node). [DEPLOY] confirm n8n version supports them; n8n must be reachable over HTTPS for webhooks and Gmail approval links; Supabase project; Slack app with bot in target channel; Gmail OAuth2; HubSpot private app token (optional, CT-07/CT-01 read); Google AI Studio key.

## 2. Supabase
1. [FILES] Review the SQL. It **drops and recreates** all four tables; run only on a new/empty project.
2. [DEPLOY] Run in the SQL editor. Confirm `NOTIFY pgrst` took effect (tables visible in the n8n Supabase node).
3. [DEPLOY] Set `dispatch_channel_id` to a real channel ID. Do not re-run the seed afterward (it overwrites).
4. [DEPLOY] Use the service-role key only inside n8n credentials. Do not put it in `.env.example`, repos or docs. Add no public RLS policies.
5. Optional hardening: see recommendations in DATABASE_SCHEMA.md (unique `incident_no`, index on `execution_id`, CHECKs).

## 3. Credentials (create in n8n; names in the exports are only labels)
Supabase API; Google Gemini (PaLM) API; Postgres (for chat memory; separate connection string); Slack API; Gmail OAuth2; HubSpot app token. After import, **re-select the credential on every node** (credential IDs in exports are instance-specific).

## 4. Import and wiring
1. Import order: CT-02, CT-03, CT-07, CT-04, CT-01, CT-05, CT-06.
2. Every Execute Workflow node stores a workflow ID (e.g. CT-02 `7krMfRZepEVi3L9V`). New instance IDs differ, so re-select the target in each node: CT-01 (CT-02 x11, CT-03), CT-04, CT-05 (CT-02 x4, CT-03), CT-06, CT-02 (CT-07).
3. CT-04 as Error Workflow: set in Settings of **CT-01 and CT-05** (missing in export) and verify CT-06. Consider CT-02/03/07.
4. Replace placeholders: approver email in CT-01 (`Database - Create Approval Record` and `Code - Read Approval Decision` and Gmail `sendTo`), Gmail recipient in CT-05, hard-coded Slack channel IDs in CT-01/04/05 (or move to `system_config`).
5. Set Gemini model name in the chat model node (not specified in the export).
6. Decide whether to enable the CT-07 hook in CT-02.
7. Change demo constants for production: retry delay 1 -> 5 min in CT-01 `Code - Build Retry Update` and CT-05 `BASE_MINUTES`; confirm CT-05 schedule interval.

## 5. Environment variables
The workflows reference **no** `$env` variables. Environment configuration is: n8n `WEBHOOK_URL`, encryption key, DB settings (deployment-specific, [DEPLOY]). Example `.env.example` (placeholders only):
```
N8N_HOST=n8n.example.com
WEBHOOK_URL=https://n8n.example.com/
N8N_ENCRYPTION_KEY=<generate-your-own>
SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<store-in-n8n-credentials-only>
```

## 6. Activation and webhooks
Activate CT-02, CT-03, CT-07 (sub-workflows), CT-04, CT-01, CT-05, CT-06. Production webhook URLs: `/webhook/ops-events`, `/webhook/ops-recover`; test URLs use `/webhook-test/`. **Put authentication or a network control in front of both before exposure** (F-01, F-03).

## 7. Test execution
Follow TESTING_AND_QA.md: T-01 first, then T-18/T-19 (failure demo using `simulate_slack_failure`). Check the SQL queries after each.

## 8. Monitoring
Slack alerts (CT-04, CT-05), n8n execution list (failed/waiting), `audit_logs` queries, unresolved-state query. Waiting executions represent pending approvals. Recommended (not implemented): dashboard on `incidents`, alert if CT-05 has not run for N minutes.

## 9. Backup and recovery
[DEPLOY] Supabase backups/PITR per plan; export workflows to version control; back up n8n encryption key and credentials separately. Chat memory table lives in the Postgres database the memory credential points to. Recovery: restore DB, re-import workflows, re-link sub-workflow IDs, re-select credentials. Incidents in RETRYING resume via CT-05.

## 10. Deployment readiness checklist
- [ ] Schema applied to a clean project; config values set
- [ ] All credentials re-selected; no secrets in repo
- [ ] Sub-workflow IDs re-linked
- [ ] CT-04 attached to CT-01, CT-05, CT-06
- [ ] Webhooks authenticated or network-restricted
- [ ] Approver/recipient/channel placeholders replaced
- [ ] Gemini model set; test AI failure path
- [ ] Supabase filter `matchType` default confirmed (CT-03, CT-05)
- [ ] CT-05 interval and retry constants set for production
- [ ] Decision made on approval-bypass recovery (F-02)
- [ ] Tests T-01..T-35 run and recorded
