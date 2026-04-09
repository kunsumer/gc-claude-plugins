<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->

# MySQL 8 + MyBatis Gotchas

Seven pitfalls specific to MySQL 8 + MyBatis + Spring Boot.

---

## 1. ALTER TABLE Metadata Lock on Large Tables

**What happens:** `ALTER TABLE` acquires a metadata lock (MDL) even when using `ALGORITHM=INPLACE`. While the DDL runs, any concurrent transaction that holds an MDL on the same table (including long-running SELECTs) blocks the alter — and all subsequent statements on that table queue behind it.

**Risk:** On a table with >1M rows or under active load, this can cascade into a connection storm.

**Fix for medium tables (100K–1M rows) — use INPLACE with LOCK=NONE:**
```sql
-- 안전한 DDL 실행 — metadata lock 최소화
ALTER TABLE bookings
  ADD COLUMN confirmation_sent_at DATETIME NULL
    COMMENT '예약 확정 알림 발송 시각'
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Fix for large tables (>1M rows) — use pt-online-schema-change:**
```bash
# pt-osc: 원본 테이블 복사본에서 DDL 실행 후 atomic rename
pt-online-schema-change \
  --alter "ADD COLUMN confirmation_sent_at DATETIME NULL COMMENT '예약 확정 알림 발송 시각'" \
  --execute \
  D=mydb,t=bookings
```

**Rule:** Check `SELECT COUNT(*) FROM table` before writing any ALTER. If >500K rows, plan for pt-osc or a maintenance window.

---

## 2. Implicit Type Conversion Kills Indexes

**What happens:** When a VARCHAR column is compared with an integer literal (or vice versa), MySQL performs implicit type conversion. The index on the VARCHAR column cannot be used — MySQL falls back to a full table scan.

**Example — broken query (no index used):**
```sql
-- partner_code는 VARCHAR(20) 컬럼 — 정수와 비교 시 인덱스 무효화
SELECT * FROM partners WHERE partner_code = 12345;
```

```sql
-- EXPLAIN 결과 예시
+----+-------------+----------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table    | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+----------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | partners | ALL  | NULL          | NULL | NULL    | NULL | 5000 | Using where |
+----+-------------+----------+------+---------------+------+---------+------+------+-------------+
-- type=ALL → 풀 스캔, rows=5000 → 전체 테이블 스캔
```

**Fix — use the correct type:**
```sql
SELECT * FROM partners WHERE partner_code = '12345';
```

**Rule:** Run `EXPLAIN` on any query that joins or filters across columns that might have been introduced by a migration. Check that `type` is not `ALL` and `key` is not `NULL` for large tables. Watch especially for MyBatis parameters passed as `int` to `VARCHAR` columns.

---

## 3. MyBatis `SELECT *` with `resultType` Breaks on Column Changes

**What happens:** MyBatis `resultType` maps query results to a Java class by matching column names (with optional camelCase conversion) to field names. When a column is added or removed and the DTO does not match, behavior depends on `autoMappingBehavior`:

- `FULL`: silently ignores extra columns, but throws `ReflectionException` if a column can't be mapped.
- `PARTIAL` (default): maps only explicitly declared properties, silently drops unknown columns.
- With strict DTO deserialization (e.g., Jackson with `FAIL_ON_UNKNOWN_PROPERTIES=true`), extra columns cause 500 errors.

**Find affected mappers:**
```bash
# SELECT * + resultType の組み合わせを探す — 最もリスクの高いパターン
grep -r "SELECT \*" services/ --include="*.xml" -l
grep -r "resultType" services/ --include="*.xml" -l
# 両方のリストに現れるファイルを優先確認
```

**Rule:** Prefer explicit `resultMap` over `resultType` for any mapper that touches a table under migration. If `SELECT *` + `resultType` exists, audit the DTO class before any column add or remove.

---

## 4. Foreign Key Cascade on DELETE

**What happens:** `ON DELETE CASCADE` silently deletes all child rows when a parent row is deleted. In a booking/option/inventory hierarchy, a misconfigured cascade can wipe option items when a product is deactivated.

**Best practice:** Always use `ON DELETE RESTRICT`. Explicit application-layer deletion with ordered cleanup is required.

**Find existing cascades before migration:**
```sql
-- 기존 CASCADE 설정 확인 — RESTRICT가 아닌 FK 전체 조회
SELECT
  TABLE_NAME,
  CONSTRAINT_NAME,
  REFERENCED_TABLE_NAME,
  DELETE_RULE
FROM INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'mydb'
  AND DELETE_RULE != 'RESTRICT';
-- Expected: 0 rows. Any result requires review before migration.
```

**Rule:** Run this query before any migration that adds or modifies FK constraints. Never write `ON DELETE CASCADE` in application schema.

---

## 5. Default Value Backfill for Existing Rows

**What happens:** `ALTER TABLE ADD COLUMN col INT NOT NULL DEFAULT 0` is instant in MySQL 8 InnoDB (the default is stored in the data dictionary, not written to every row). But this means **existing rows return the default value without a physical UPDATE**. If the application later needs to distinguish "was set to 0 explicitly" from "was never set", the default-as-proxy breaks.

Additionally, if you need a **non-default initial value** for existing rows (e.g., `status='ACTIVE'` for legacy records), you must run a backfill UPDATE explicitly.

**Batch backfill pattern:**
```sql
-- 기존 행 배치 업데이트 — 한 번에 1000건, 복제 지연 방지
UPDATE product_options
SET availability_status = 'ACTIVE'
WHERE availability_status IS NULL
  AND id BETWEEN 1 AND 1000;
-- id 범위를 1000씩 이동하며 반복; 각 실행 후 SLEEP(0.05) 권장

-- 완료 확인
SELECT COUNT(*) FROM product_options WHERE availability_status IS NULL;
-- Expected: 0
```

**Rule:** After adding a NOT NULL column with a default, explicitly verify that the intended business value for existing rows is correct, not just that the default filled in. If it differs, run a targeted backfill UPDATE.

---

## 6. Transaction Isolation During Backfill

**What happens:** MySQL default isolation is `REPEATABLE READ`. A long-running backfill transaction takes a consistent snapshot at the start. Rows inserted or updated by other transactions during the backfill are invisible inside the backfill transaction. This means:

- New bookings created during a backfill of `bookings` will be missed by the backfill transaction.
- The backfill will appear to complete but will have gaps.

**Fix — use `READ COMMITTED` + batch commit pattern:**
```sql
-- 세션 격리 수준을 READ COMMITTED로 변경 후 배치 커밋
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
INSERT INTO option_items (group_id, item_name, ...)
SELECT og.id, po.option_name, ...
FROM product_options po
JOIN option_groups og ON og.product_id = po.product_id
ORDER BY po.id
LIMIT 1000 OFFSET 0;
COMMIT;

-- 다음 배치
START TRANSACTION;
-- ... OFFSET 1000
COMMIT;

-- 각 COMMIT 후 SLEEP(0.1)으로 복제 레그 완화
```

**Rule:** Any backfill that runs longer than ~5 seconds should use `READ COMMITTED` + batch commit. Never run a single large INSERT...SELECT that could hold a transaction open for minutes.

---

## 7. Online DDL and Foreign Keys

**What happens:** `ALGORITHM=INPLACE` does not support all DDL operations when foreign keys are involved. Specifically, adding a foreign key with `INPLACE` is allowed in MySQL 8, but **dropping a foreign key constraint** or **changing a FK column type** requires `ALGORITHM=COPY`, which rebuilds the entire table and takes an exclusive lock.

**Check MySQL version capabilities:**
```sql
-- MySQL 버전 확인 — 8.0.x 마이너 버전에 따라 INPLACE 지원 범위 다름
SELECT VERSION();
-- 8.0.28+ 기준으로 대부분의 ADD FK는 INPLACE 가능
-- DROP FK, FK 컬럼 타입 변경은 COPY 필요
```

**Safe pattern — drop FK before column change, re-add after:**
```sql
-- 1단계: FK 제약 제거
ALTER TABLE option_items
  DROP FOREIGN KEY fk_option_items_group
  ALGORITHM=INPLACE, LOCK=NONE;

-- 2단계: 컬럼 타입 변경 (이제 FK 없으므로 INPLACE 가능)
ALTER TABLE option_items
  MODIFY COLUMN group_id BIGINT UNSIGNED NOT NULL
    COMMENT '옵션 그룹 ID'
  ALGORITHM=INPLACE, LOCK=NONE;

-- 3단계: FK 재추가
ALTER TABLE option_items
  ADD CONSTRAINT fk_option_items_group
    FOREIGN KEY (group_id) REFERENCES option_groups(id) ON DELETE RESTRICT
  ALGORITHM=INPLACE, LOCK=NONE;
```

**Rule:** Before running any DDL on a table with foreign keys, explicitly check `INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS` and plan whether INPLACE is available. When in doubt, test the exact statement in a staging environment and confirm with `EXPLAIN FORMAT=TREE` or the optimizer trace that no full table rebuild occurs.
