# PROJECT_STRUCTURE.md (proposed, not existing)

```text
business-control-tower/
├── README.md
├── ARCHITECTURE.md
├── ARCHITECTURE_REVIEW.md
├── DATABASE_SCHEMA.md
├── API_AND_INTEGRATIONS.md
├── DATA_FLOW.md
├── TESTING_AND_QA.md
├── SETUP_AND_OPERATIONS.md
├── architecture-diagram.md
├── workflows/                 # sanitised exports (strip credential IDs, instance IDs)
│   ├── CT-01-main-event-intake-orchestration.json
│   ├── CT-02-sub-state-audit-writer.json
│   ├── CT-03-sub-dispatch-action.json
│   ├── CT-04-global-error-handler.json
│   ├── CT-05-watchdog.json
│   ├── CT-06-human-recovery-endpoint.json
│   └── CT-07-sub-hubspot-outcome-sync.json
├── docs/workflow-readmes/     # CT-01-README.md ... CT-07-README.md
├── database/schema.sql        # the supplied SQL; add a separate non-destructive migration for changes
├── examples/
│   ├── sample-event.json
│   └── sample-test-cases.json
└── .env.example               # placeholders only
```

Name the workflow files by the real workflow names above. Keep `schema.sql` destructive-by-design only in a `dev/` variant; production uses additive migrations. Never commit credentials, instance IDs you consider sensitive, or real emails/channel IDs.
