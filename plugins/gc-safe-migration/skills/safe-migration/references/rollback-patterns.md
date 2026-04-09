<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->

# Rollback Patterns

Five patterns covering the most common migration scenarios in this stack. Choose by migration type; every migration must have a named rollback pattern before execution.

---

## Pattern 1 — Reversible Additive (ADD → DROP)

**Use when:** Adding a new nullable column or a NOT NULL column with a safe default.

**Forward:**
```sql
ALTER TABLE partners
  ADD COLUMN guide_certification_type VARCHAR(20) NULL
    COMMENT '가이드 자격증 유형 코드'
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Rollback:**
```sql
ALTER TABLE partners
  DROP COLUMN guide_certification_type
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Notes:**
- DROP COLUMN is instant for nullable columns in MySQL 8 InnoDB.
- No data at risk — the column was new and contained only NULLs or defaults.
- Application code that writes to the column must be rolled back before the SQL rollback, or writes will fail.

---

## Pattern 2 — Reversible Rename (RENAME COLUMN → reverse)

**Use when:** Renaming a column for clarity or to align with a new naming convention.

**Critical prerequisite — audit mappers first:**
```bash
# 이름 변경 전 모든 참조 위치 확인 (XML 매퍼, Java 클래스, 테스트 픽스처 포함)
grep -r "old_column_name\|oldColumnName" \
  services/ packages/ \
  --include="*.xml" --include="*.java" --include="*.sql" \
  -n
# 모든 참조를 새 이름으로 변경하고 배포 완료 후 SQL 실행
```

**Forward:**
```sql
-- 컬럼 이름 변경 (MySQL 8 RENAME COLUMN 지원)
ALTER TABLE bookings
  RENAME COLUMN traveler_phone_raw TO traveler_contact_phone
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Rollback:**
```sql
-- 원래 이름으로 복원
ALTER TABLE bookings
  RENAME COLUMN traveler_contact_phone TO traveler_phone_raw
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Notes:**
- RENAME COLUMN is supported from MySQL 8.0. Confirm `SELECT VERSION()` is 8.0+.
- Application code must be deployed with the new column name **before** the SQL rename, or queries will fail. Use dual-read (accept both names in the application layer) if zero-downtime is required.
- On rollback: revert application code first, then run the reverse RENAME.

---

## Pattern 3 — Backfill with Recovery Table

**Use when:** Updating existing row data (backfill) where the original values need to be recoverable.

**Setup — create recovery table before any modification:**
```sql
-- 수정 전 복구용 테이블 생성
CREATE TABLE option_items_pre_migration_backup AS
  SELECT id, price_delta, inventory_count, NOW() AS backed_up_at
  FROM option_items;

-- 건수 확인
SELECT COUNT(*) FROM option_items_pre_migration_backup;
-- 원본과 일치 확인 후 진행
```

**Forward migration — batch UPDATE:**
```sql
-- 가격을 원화 단위로 정규화 (원/100 → 원, 1000건 배치)
UPDATE option_items
SET price_delta = price_delta * 100
WHERE id BETWEEN 1 AND 1000
  AND price_unit = 'JEON';
-- 1000씩 범위 이동하며 반복
```

**Recovery — restore from backup table:**
```sql
-- 복구: backup 테이블에서 원래 값으로 JOIN UPDATE
UPDATE option_items oi
JOIN option_items_pre_migration_backup bk ON bk.id = oi.id
SET oi.price_delta = bk.price_delta,
    oi.inventory_count = bk.inventory_count;

-- 복구 건수 확인
SELECT COUNT(*) FROM option_items oi
JOIN option_items_pre_migration_backup bk ON bk.id = oi.id
WHERE oi.price_delta != bk.price_delta;
-- Expected: 0 (모든 행 복구 완료)
```

**Cleanup:**
```sql
-- T+7~30일 이후 백업 테이블 삭제 (일정 등록 필요)
DROP TABLE option_items_pre_migration_backup;
```

**Notes:**
- The recovery table only captures rows that existed at backup time. Rows inserted between backup and rollback execution will not be in the backup — their modified values will remain after recovery.
- 7-day minimum retention; 30-day maximum before the backup itself should be cleaned up.
- Document the backup table name and creation timestamp in the migration artifact.

---

## Pattern 4 — Non-Reversible Destructive (Backup Before DROP)

**Use when:** Dropping a column, table, or rows containing data that cannot be regenerated (especially PII after vault migration).

**Prerequisite — backup before DROP:**
```sql
-- 삭제 전 반드시 백업 테이블 생성
CREATE TABLE bookings_passport_backup_20260409 AS
  SELECT id, booking_code, traveler_passport_number, NOW() AS backed_up_at
  FROM bookings
  WHERE traveler_passport_number IS NOT NULL;

SELECT COUNT(*) FROM bookings_passport_backup_20260409;
```

**Forward:**
```sql
ALTER TABLE bookings
  DROP COLUMN traveler_passport_number
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Rollback — EXPLICITLY NOT STANDARD:**

This is not a standard rollback. Recovery is a partial, emergency-only procedure:
```sql
-- 긴급 복구 (사전 승인 필요, 표준 롤백 아님)
ALTER TABLE bookings
  ADD COLUMN traveler_passport_number VARCHAR(30) NULL
  ALGORITHM=INPLACE, LOCK=NONE;

UPDATE bookings b
JOIN bookings_passport_backup_20260409 bk ON bk.id = b.id
SET b.traveler_passport_number = bk.traveler_passport_number;
```

**Recovery is partial:** rows inserted after the backup was created will not have data. Explicitly document this limitation in the migration artifact.

**Notes:**
- Label the rollback section in the migration artifact: "NOT REVERSIBLE — recovery requires backup table + vault data."
- For PII columns: the destruction itself may be legally required (PIPA Article 21). Document that recovery contradicts the destruction intent and requires compliance sign-off.
- Retain backup table for 30 days, then destroy it per the same retention policy.

---

## Pattern 5 — Schema-Only Rollback with Application Dependency

**Use when:** The schema change and the application code must move together (e.g., a NOT NULL column without a default, or a column type change that is not backward-compatible). Rolling back only the schema without rolling back the application will cause errors.

**Dependency order:**
- **Deploy forward:** application first (with backward-compatible code), then schema.
- **Rollback:** schema first (restore old DDL), then application (deploy old version).

**Rollback procedure template:**

```markdown
## Rollback Procedure — [Migration Name] — [Date]

### Trigger conditions
- Error rate on [endpoint] exceeds X% for Y minutes
- Booking flow returning 500 on option selection
- Mapper deserialization exception in logs

### Step 1 — Roll back application (deploy prior version)
- Revert to tag: `release/YYYY-MM-DD-HH`
- Deploy command: `./deploy.sh domain-api release/YYYY-MM-DD-HH`
- Wait for health check: `GET /actuator/health` returns 200

### Step 2 — Roll back schema (after application is stable)
```sql
-- 스키마 롤백 SQL
ALTER TABLE option_items
  MODIFY COLUMN option_type VARCHAR(20) NOT NULL
    COMMENT '옵션 유형 코드 (구버전 호환)'
  ALGORITHM=INPLACE, LOCK=NONE;
```

### Step 3 — Verify
- Confirm `SHOW COLUMNS FROM option_items` matches pre-migration state.
- Smoke test booking flow end-to-end.
- Confirm error rate returns to baseline.

### Step 4 — Cache invalidation (if applicable)
```bash
redis-cli --scan --pattern "product:*:options" | xargs redis-cli del
```

### Step 5 — Notify
- Update incident channel with rollback status.
- Create post-mortem ticket.
```

**Notes:**
- Never roll back the schema first if the live application still expects the new schema — it will make things worse.
- The rollback procedure document should be written and reviewed **before** the migration is executed, not drafted during an incident.
- Keep the rollback procedure in the migration artifact (`05-implementation-plan.md` or the active spec folder).
