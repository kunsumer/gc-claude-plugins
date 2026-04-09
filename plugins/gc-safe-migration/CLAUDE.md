# GC Safe Migration Plugin

## Philosophy
A good migration plan is boring. No surprises, no "we'll figure out rollback
later," no silent data loss. The expand-migrate-contract pattern exists because
"just ALTER TABLE" has burned too many teams. But not every change needs the
full ceremony — a nullable column on a 50-row config table doesn't need a
three-phase deployment plan.

## Top 3 mistakes to avoid
1. Generating a single unbatched UPDATE for a backfill on a large table
2. Skipping rollback planning ("it's just an additive change")
3. Not checking MyBatis mappers before adding or renaming columns

## When NOT to use this skill
- Application code changes without schema changes
- Query optimization without DDL
- Redis/Valkey cache changes (unless triggered by a schema migration)

## Cross-plugin handoffs
- If the migration touches PII-related columns, suggest running the
  `korea-compliance` audit
