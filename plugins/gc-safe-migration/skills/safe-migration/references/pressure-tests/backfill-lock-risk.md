# Pressure Test: Backfill Lock Risk

## Scenario
Adding `normalized_phone` to `travelers` (5M rows), backfill by normalizing existing `phone` column. System processes bookings 9am–11pm KST.

## Expected behavior
Heavy review. Batch size 1000–5000 rows per batch, SLEEP between batches, READ COMMITTED isolation, schedule during low-traffic window (11pm–9am KST), flag `phone` as PII, generate batched UPDATE SQL.

## Failure modes
- Single unbatched UPDATE across all 5M rows
- No lock contention warning
- No batching or rate limiting advice
- Using REPEATABLE READ isolation (default) instead of READ COMMITTED
- No PII flag for the `phone` column
- Recommending execution during business hours (9am–11pm KST) without strong caveat

## Why this is hard
The request sounds simple: "add a column and normalize the data." But 5M rows + continuous booking traffic + PII = heavy complexity path. The skill must recognize all three multipliers and apply them together, not just the most obvious one.

## Correct output shape

```sql
-- Phase 1: Add column (no backfill yet)
ALTER TABLE travelers ADD COLUMN normalized_phone VARCHAR(20) NULL;

-- Phase 2: Batched backfill (run during 11pm–9am KST window)
-- Batch size: 1000–5000 rows; adjust based on observed lock wait times
-- Isolation: READ COMMITTED to minimize lock duration

SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Repeat until max(id) is reached:
UPDATE travelers
SET normalized_phone = normalize_phone(phone)
WHERE id BETWEEN :batch_start AND :batch_end
  AND normalized_phone IS NULL;

SELECT SLEEP(0.1); -- rate limiting between batches

-- Phase 3: Add NOT NULL constraint after backfill verified complete
ALTER TABLE travelers MODIFY COLUMN normalized_phone VARCHAR(20) NOT NULL;
```

## Required flags
- PII: `phone` and `normalized_phone` are personal data — flag for compliance review
- Scheduling: execute Phase 2 strictly outside booking hours (11pm–9am KST)
- Monitoring: watch `SHOW PROCESSLIST` and lock-wait metrics during backfill
- Rollback: `ALTER TABLE travelers DROP COLUMN normalized_phone;`
