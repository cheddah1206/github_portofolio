# Business Control Tower: Architecture Diagrams

Convention: **solid arrows** = relationship verified in the workflow JSON. **Dashed arrows** = present but disabled, or proposed/unverified. All diagrams reflect the exports only; no runtime execution evidence exists.

## Diagram A: Executive Overview

```mermaid
flowchart LR
  subgraph IN["Business event"]
    E["Event from a department"]
  end
  subgraph CORE["Control Tower processing"]
    V["Validate, sign, de-duplicate"]
    AI["AI impact assessment"]
    S["Score priority and risk"]
    D{"Approval needed?"}
    H["Human approval by email"]
    X["Dispatch Slack alert"]
  end
  subgraph SAFE["Safety nets"]
    R["Scheduled retries"]
    ESC["Escalation"]
  end
  DB[("Supabase: incidents, audit_logs, approvals, system_config")]
  OUT1["Completed"]
  OUT2["Rejected"]
  OUT3["Escalated / Failed"]

  E --> V --> AI --> S --> D
  D -- "yes" --> H
  D -- "no" --> X
  H -- "approved" --> X
  H -- "rejected" --> OUT2
  H -- "no answer 45 min" --> OUT3
  X -- "posted" --> OUT1
  X -- "Slack failed" --> R
  R -- "retry ok" --> OUT1
  R -- "retries used up" --> ESC --> OUT3
  V -- "invalid" --> OUT2
  V --> DB
  S --> DB
  R --> DB
```

**Explanation.** An event enters, is checked, scored and either dispatched straight to Slack or held for a human decision. If Slack fails, a scheduler retries; if retries run out, the incident is escalated. Everything is stored in Supabase. "Completed" means the Slack message was posted; the files contain no further business action.

## Diagram B: Seven-Workflow System Blueprint

```mermaid
flowchart TB
  subgraph ENTRY["Entry points"]
    WH1["Webhook POST /ops-events"]
    WH2["Webhook POST /ops-recover"]
    SCH["Schedule trigger"]
    ERR["n8n error event"]
  end
  subgraph WF["n8n workflows"]
    CT01["CT-01 Main - Event Intake and Orchestration"]
    CT02["CT-02 Sub - State and Audit Writer"]
    CT03["CT-03 Sub - Dispatch Action"]
    CT04["CT-04 Global Error Handler"]
    CT05["CT-05 Watchdog"]
    CT06["CT-06 Human Recovery Endpoint"]
    CT07["CT-07 Sub - HubSpot Outcome Sync"]
  end
  subgraph EXT["External services"]
    GEM["Google Gemini"]
    PGM["Postgres chat memory"]
    HSR["HubSpot contact search"]
    HSN["HubSpot notes API"]
    GM["Gmail"]
    SL["Slack"]
  end
  subgraph DB["Supabase"]
    INC[("incidents")]
    AUD[("audit_logs")]
    APP[("approvals")]
    CFG[("system_config")]
  end

  WH1 --> CT01
  WH2 --> CT06
  SCH --> CT05
  ERR --> CT04
  CT01 --> CT02
  CT01 --> CT03
  CT05 --> CT02
  CT05 --> CT03
  CT06 --> CT02
  CT04 --> CT02
  CT02 -. "hook node disabled" .-> CT07

  CT01 --> GEM
  CT01 --> PGM
  CT01 --> HSR
  CT01 --> GM
  CT01 --> SL
  CT03 --> SL
  CT04 --> SL
  CT05 --> GM
  CT05 --> SL
  CT07 --> HSR
  CT07 --> HSN

  CT01 -->|"read, insert, update"| INC
  CT01 -->|"insert, update"| APP
  CT02 -->|"update"| INC
  CT02 -->|"insert"| AUD
  CT03 -->|"read"| CFG
  CT04 -->|"read, insert"| INC
  CT04 -->|"insert"| AUD
  CT05 -->|"read"| INC
  CT05 -->|"insert"| AUD
  CT06 -->|"read"| INC
  CT07 -->|"read"| INC
  CT07 -->|"insert"| AUD
```

**Explanation.** CT-02 is the hub: every state change from CT-01, CT-04, CT-05 and CT-06 passes through it. CT-03 is the only place that posts incident alerts; CT-01 calls it once, CT-05 calls it for retries. CT-04 is wired as error workflow only on CT-06 in the export (not drawn, because it is a setting, not a node connection). `escalation_channel_id` in `system_config` is read by no workflow.

## Diagram C: Detailed Data and Decision Flow

```mermaid
flowchart TD
  A["Webhook: body"] --> B["Code: Validate Event"]
  B --> C["Crypto: SHA-256 of raw body"]
  C --> D["DB: find incident by event_id"]
  D --> E{"Already exists?"}
  E -- "yes" --> F["Respond 200 duplicate"]
  E -- "no" --> G["DB: insert incident, state RECEIVED"]
  G --> H["CT-02: audit EVENT_RECEIVED then VALIDATING"]
  H --> I{"is_valid?"}
  I -- "no" --> J["Respond 400"] --> K["CT-02: REJECTED, reason INVALID_SCHEMA"]
  I -- "yes" --> L["Respond 202"] --> M["CT-02: AI_PROCESSING"]
  M --> N{"customer_email valid?"}
  N -- "yes" --> O["HubSpot read-only lookup, 3 tries"]
  N -- "no" --> P["Build AI context"]
  O --> P
  P --> Q["AI Agent: Gemini + memory + parser, 2 tries"]
  Q -- "ok" --> R["Compute risk, priority, SLA, approval"]
  Q -- "error" --> Q2["CT-02 audit AI_FALLBACK, safe default"] --> R
  R --> S["DB: save classification"] --> T["CT-02: CLASSIFIED"]
  T --> U{"requires_approval?"}
  U -- "yes" --> V["CT-02: AWAITING_APPROVAL"] --> W["DB: insert approval PENDING"]
  W --> X["Gmail Send and Wait 45 min"]
  X --> Y{"decision"}
  Y -- "approved" --> Z["CT-02: APPROVED"]
  Y -- "rejected" --> Y1["CT-02: REJECTED"] --> Y2["Slack: rejection notice"]
  Y -- "timeout or mail error" --> Y3["CT-02: ESCALATED"] --> Y4["Slack: timeout alert"]
  U -- "no" --> PR
  Z --> PR["CT-02: PROCESSING"]
  PR --> DI["CT-03: build message, Slack post, 3 tries"]
  DI -- "ok" --> CO["CT-02: COMPLETED"]
  DI -- "fail" --> RT["CT-02: RETRYING, retry_count 1, next_retry_at +1 min"]
  RT --> W5["CT-05 every tick: due retries"]
  W5 --> W6["CT-02: PROCESSING"] --> W7["CT-03 again"]
  W7 -- "ok" --> W8["CT-02: COMPLETED"]
  W7 -- "fail" --> W9{"attempts >= max_retries?"}
  W9 -- "no" --> RT2["CT-02: RETRYING, later time"]
  W9 -- "yes" --> ES["CT-02: ESCALATED, Gmail and Slack"]
  ES -. "human calls /ops-recover" .-> HR["CT-06 calls CT-02: RETRYING, retry_count 0"]
  HR --> W5
```

**Explanation.** The diagram follows one event from webhook to a terminal state, using only nodes present in CT-01, CT-03 and CT-05. The dashed arrow is the human recovery loop (CT-06): it is a separate workflow reached by an operator, not an automatic step. Not shown: stale scan, systemic alert and global error handling (see ARCHITECTURE.md section 8).
