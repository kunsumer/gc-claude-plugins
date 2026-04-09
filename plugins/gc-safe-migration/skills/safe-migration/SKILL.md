---
name: safe-migration
description: >
  Use for schema changes, data migrations, and backward-compatibility-sensitive
  changes in Spring Boot + MyBatis + MySQL services. Guides expand-migrate-contract
  patterns, rollback planning, and compliance-aware migration design.
---

# Safe Migration

## Goal
Plan and implement a migration without turning production into a haunted house.

## Steps
1. Identify data contract and blast radius
2. Define forward path and rollback or recovery path
3. Prefer expand -> migrate -> contract when possible
4. Document compatibility window
5. Verify with targeted tests and operational notes

## Blast radius assessment
- Which tables, indexes, and foreign keys are affected?
- Which services read or write the affected columns?
- Are there MyBatis mappers that reference the affected columns by name?
- Will Redis/Valkey caches need invalidation after the migration?
- Are there Kafka consumers that deserialize rows from the affected tables?

## Expand-migrate-contract pattern
1. **Expand:** Add new columns/tables without removing old ones. Deploy code that writes to both.
2. **Migrate:** Backfill existing data. Verify consistency.
3. **Contract:** Remove old columns/tables after all consumers have migrated. Deploy code that only uses new schema.

Each phase should be a separate deployment. Do not combine expand and contract in one release.

## Compliance check
- Does the migration add, rename, or change fields containing PII?
- Does it alter retention, encryption, or access-control characteristics?
- Does it create new data flows to external systems?
- If yes to any: flag for compliance review before execution.

## Constraints
- One-way migrations require explicit approval
- Destructive data operations require a recovery note
- Backfill queries must be bounded (batch size, rate limit) to avoid lock contention
- Migration names must clearly describe the change (no ambiguous names hiding data rewrites)

## Verification
- Run migration on a test database with production-like data volume
- Verify rollback script actually reverses the change
- Confirm MyBatis mapper queries still compile after schema change
- Check that application starts and passes health checks after migration

## Self-review checklist
Before finalizing any migration plan, verify all ten points:

1. **Classify type** — Is this additive, rename, backfill, destructive, or split/merge? Complexity path follows from type.
2. **List affected tables, indexes, and FKs** — Name every object touched. Do not leave implicit.
3. **Check MyBatis mappers** — Search for `resultMap`, `<result column=`, `@Results`, and `SELECT *` that reference affected columns. Column renames silently break mappers.
4. **Rollback / recovery note** — Every migration must have an explicit rollback or a stated reason why rollback is not possible.
5. **Backup strategy for destructive operations** — If the migration drops data (DROP COLUMN, TRUNCATE, DELETE), document the backup step and who approves it.
6. **Batch size for backfills** — Backfill on >1K rows must specify batch size (1000–5000), SLEEP interval, and isolation level (prefer READ COMMITTED).
7. **Check PII columns** — Does the migration add, rename, or transform fields containing personal data? See cross-plugin integration rules below. If yes, flag before proceeding.
8. **Descriptive migration name** — Migration filename must describe what changes, not just "update_table" or "schema_fix." Example: `V20260901__add_normalized_phone_to_travelers`.
9. **Verification steps** — List at least: test-db run, rollback test, mapper compilation check, app health check after migration.
10. **Expand-migrate-contract phases separated** — If the pattern applies, confirm each phase is a distinct deployment. Do not bundle expand and contract.

Reference files: `references/pressure-tests/`, `references/worked-examples/`, `references/mysql-gotchas-and-rollback.md`.

## Graduated complexity

Apply the correct complexity path based on change type and row volume. Do not escalate trivially; do not under-review risky changes.

### Light path
**Applies when:** nullable column add, index add/drop, config/lookup table change, row count <1K, no PII, no external consumers, no Kafka.

**Output:** SQL migration + rollback SQL. Skip blast radius ceremony, expand-migrate-contract, and pt-online-schema-change. ~20 lines total.

### Standard path
**Applies when:** column rename, backfill, FK add, constraint change, row count 1K–1M, or any change with external service consumers or Kafka.

**Output:** Full blast radius assessment, MyBatis mapper audit, rollback pattern, expand-migrate-contract recommendation where applicable, verification checklist.

### Heavy path
**Applies when:** table split/merge, destructive data operation, PII column affected, row count >1M, or any combination of: continuous traffic + backfill + sensitive data.

**Output:** Full blast radius + compliance flag + expand-migrate-contract with separate deployment phases + staged rollout plan + backup strategy + recovery note. Batched backfill SQL with SLEEP, READ COMMITTED isolation, and low-traffic scheduling required.

## Cross-plugin integration

When a migration touches columns with names that suggest personal data, flag for compliance review and suggest running `/audit-compliance architecture`.

**PII-signal column names (non-exhaustive):**
- `passport`, `passport_number`, `passport_no`
- `resident_registration`, `rrn`, `jumin`
- `account_number`, `bank_account`, `settlement_account`
- `phone`, `mobile`, `contact_phone`
- `email`, `contact_email`
- `name` combined with table context: `traveler`, `partner`, `guide`, `personal`, `profiles`

**Rule:** If the affected table is `travelers`, `partner_profiles`, `guides`, `bookings` (traveler fields), or any table whose name or column set suggests personal data storage, apply the heavy path's compliance flag regardless of row count.

**Handoff message to include:**
> This migration touches columns that may contain personal data regulated under PIPA. Recommend running `/audit-compliance architecture` before execution.
