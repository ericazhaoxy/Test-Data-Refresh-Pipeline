# Test-Data-Refresh-Pipeline

Refresh non-production databases (dev/test/staging) from production safely and repeatably.
This pipeline copies production data into an isolated workspace, **sanitizes / masks sensitive fields**, **minimizes data size**, then **restores** the final dataset to one or many non-prod environments with environment-specific updates.

## Why this exists

Teams need realistic data in dev/test/staging, but production data often contains PII and is too large to copy as-is.
This project provides an automated, configurable workflow to:
- keep non-prod data up to date,
- protect sensitive information,
- reduce storage/cost by keeping only what’s needed,
- standardize refresh across multiple environments.

## What it does (in plain English)

Think of production data as a big pot of soup:
1) copy it to a temporary pot (so production stays untouched),
2) remove/cover ingredients that shouldn’t be shared (PII),
3) pour only a small portion needed for testing,
4) deliver the final soup to different teams (dev/test/staging),
5) change each team’s “labels/settings” so it works in that environment.

## End-to-end workflow

**Step 1 — Copy Production → Temporary Clean DB**
- Create an isolated “clean workspace” database from production.

**Step 2 — Sanitization + Obfuscation (Masking)**
- Remove or mask sensitive fields (e.g., names, phone numbers, emails, IDs).
- Strategies can include redaction, masking, tokenization, field removal, or anonymization.

**Step 3 — Copy Clean DB → Temporary Mini DB**
- Create a second workspace for “small, test-ready data”.

**Step 4 — Minimization**
- Reduce data volume by applying rules (e.g., time-window filters, sampling, table/column selection, aggregation).

**Step 5 — Restore Mini DB → dev/test/staging**
- Restore the minimized dataset to one or many target environments.
- Apply environment-specific updates (URLs, feature flags, configs, etc.) after restore.

## Architecture (high level)

- **Production Database**: source of truth (never modified)
- **Rules Engine**: decides what to copy, how to transform, where to restore
- **Transformation Modules**:
  - Sanitization (remove unsafe content)
  - Obfuscation/Masking (protect PII while keeping format usable)
  - Minimization (keep only what’s needed)
- **Orchestration (Airflow DAG)**: runs the workflow as a repeatable pipeline (retry, schedule, monitoring)

## Configuration (example)

This project is designed to be **config-driven** so you can refresh multiple environments without changing core logic.

Example structure (you can adapt):
```json
{
  "targets": [
    {"name": "dev", "connection": "DEV_DB"},
    {"name": "staging", "connection": "STAGING_DB"}
  ],
  "copy": {
    "from": "PROD_DB",
    "clean_db": "TEMP_CLEAN_DB",
    "mini_db": "TEMP_MINI_DB"
  },
  "masking": {
    "rules": [
      {"table": "users", "column": "email", "method": "mask_email"},
      {"table": "users", "column": "phone", "method": "mask_phone"},
      {"table": "users", "column": "name", "method": "fake_name"}
    ]
  },
  "minimization": {
    "rules": [
      {"table": "orders", "filter": "order_date >= NOW() - INTERVAL '30 days'"},
      {"table": "events", "sample": 0.05},
      {"table": "audit_logs", "drop": true}
    ]
  },
  "post_restore_updates": [
    {"target": "dev", "sql": "UPDATE config SET api_base_url='https://dev.api...'"},
    {"target": "staging", "sql": "UPDATE config SET api_base_url='https://staging.api...'"}
  ]
}
