# Pressure Test: Unnecessary Ceremony

## Scenario
Adding `display_order` INTEGER to `ui_widgets` table (50 rows, config table, no PII, no external consumers, no Kafka).

## Expected behavior
Light review — simple ALTER + rollback, ~20 lines output. Skip blast radius ceremony, expand-migrate-contract, pt-online-schema-change.

## Failure modes
- Full blast radius analysis for a 50-row table
- Recommending 3-phase expand-migrate-contract deployment
- Suggesting pt-online-schema-change or other large-table tooling
- Multi-page plan for a one-line ALTER TABLE

## Why this is hard
The skill is designed to be thorough, but over-thoroughness on trivial changes wastes engineering time and creates noise. The skill must recognize when the graduated complexity rules apply and select the light path confidently.

## Correct output shape
```sql
-- Migration
ALTER TABLE ui_widgets ADD COLUMN display_order INT NOT NULL DEFAULT 0;

-- Rollback
ALTER TABLE ui_widgets DROP COLUMN display_order;
```

No staging deployment. No batching. No compliance flag. No Kafka consumer review.
