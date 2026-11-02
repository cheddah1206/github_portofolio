# API_AND_INTEGRATIONS.md

Status key: **Verified** = node configuration present in JSON (not runtime-tested). **Partial** = configured but depends on missing values. **Unverified** = cannot be established. No secrets are reproduced; credentials are named by type only.

| # | Integration | Node type | Workflow(s) | Purpose | Auth | Status |
|---|---|---|---|---|---|---|
| 1 | Supabase | native `supabase` | all | CRUD on four tables | Supabase API credential (service role expected) | Partial (key type UNVERIFIED) |
| 2 | Google Gemini | LangChain `lmChatGoogleGemini` | CT-01 | Impact assessment | Google PaLM/Gemini API key credential | Partial (model name not set) |
| 3 | Postgres chat memory | LangChain `memoryPostgresChat` | CT-01 | Per customer/entity conversation memory | Postgres credential | Partial (table not in schema) |
| 4 | HubSpot search | native `hubspot` v2 | CT-01, CT-07 | Contact lookup | App token | Verified config |
| 5 | HubSpot notes | **HTTP Request** v4.2 | CT-07 | Create note + association | HubSpot credential via `predefinedCredentialType` | Verified config; never triggered |
| 6 | Slack | native `slack` v2.3 | CT-01, CT-03, CT-04, CT-05 | Alerts | Slack API credential | Verified config |
| 7 | Gmail | native `gmail` v2.1 | CT-01 (`sendAndWait`), CT-05 (`send`) | Approval, escalation email | Gmail OAuth2 | Partial (placeholder recipients) |
| 8 | n8n sub-workflow calls | `executeWorkflow` | CT-01/04/05/06/02 | Internal | n8n | Verified |
| 9 | Inbound webhooks | `webhook` | CT-01, CT-06 | Intake / recovery | **None configured** | Verified (no auth) |

## 1. Supabase
Operations: `getAll` with filters (`eq`, `gte`), `insert` (autoMap or field list), `update` (filter + autoMap or fields). Used columns match the schema. Responses: arrays of rows (inserted/updated row returned). Failure: node error (no retries configured on any Supabase node; several set `alwaysOutputData`). Filters without `matchType` in CT-03 and CT-05 Branch B depend on the node default (F-13). Config: project URL + key; PostgREST schema cache reload after DDL.

## 2. Gemini
Chat model temperature 0.2. Request: system message (Brainpower prompt, ~16 sections) + user text `Assess this operations event. Event JSON: ... HubSpot status ... context`. Expected response: JSON per `Structured Output Parser` (required: `category`, `impact_score` 1-10 int, `summary`, `recommended_action`, `affected_departments` enum list, `confidence` 0-1). Prompt additionally asks for 8 other fields (`business_impact`, `human_approval_required`, `incident_required`, ...) that are not in the parser and are ignored. Failure: agent retries twice then error output -> deterministic fallback. Risks: invalid/hallucinated content is mitigated only by schema + clamping + code-side decisions; no check that the score fits the severity band; the full event (including `customer_email` if present in payload) is sent to Google.

## 3. Postgres chat memory
Session key `ct-hs-<contactId>` or `ct-ent-<department>-<entity_id>` (sanitised, never an email). Past events for the same key are available to later assessments (cross-event influence, F-21). Requires a direct Postgres connection string (host, DB, user, password) which differs from the Supabase REST credential.

## 4-5. HubSpot
Search: `email EQ <lower-cased email>`, sort `createdate ASC`, properties `email` (CT-07) or `email, lifecyclestage, hs_lead_status, company` (CT-01). Retry 3x/2 s then continue. Notes: `POST https://api.hubapi.com/crm/v3/objects/notes`, JSON body with `hs_timestamp`, `hs_note_body` (HTML escaped), association type id 202. Success response carries `id`. Opportunity: the HTTP node may be replaceable by a native operation; compatibility (note creation + association support in the installed node version) was not assessed, so no substitution is proposed as a fix.

## 6. Slack
Channel by ID. CT-03 uses the configured channel; CT-01/04/05 use a hard-coded channel ID. `text` only (no blocks). Failure behaviour: CT-03 retries 3x then fails; CT-01 notices, CT-04 (continue on error), CT-05 notices have no retry.

## 7. Gmail
CT-01: `sendAndWait` with "double" approval (approve and "Reject" buttons), 45-minute limit; response `data.approved`. CT-05: plain send. Recipients are placeholders/hard-coded; `appendAttribution:false`.

## 9. Webhooks
`POST /ops-events`, `POST /ops-recover`. Request/response shapes in the workflow READMEs. Responses: CT-01 200/202/400; CT-06 200/409. No auth, no rate limit, no signature verification.

## Native vs HTTP
Only one HTTP Request node exists (HubSpot notes). Everything else uses native or LangChain nodes.
