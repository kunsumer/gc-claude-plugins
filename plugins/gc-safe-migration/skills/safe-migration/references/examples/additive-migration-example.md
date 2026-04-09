<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->

# Worked Example: Additive Migration — Nullable Column on `partners`

**Scenario:** Add `guide_certification_type VARCHAR(20) NULL` to the `partners` table (~300 rows).

---

## Step 1 — Classify

| Attribute        | Value                                           |
|------------------|-------------------------------------------------|
| Migration type   | Additive (new nullable column)                  |
| Table size       | ~300 rows — small; DDL is near-instant          |
| Risk level       | Light                                           |
| Review path      | Lightweight — no compliance flag needed         |

**Reasoning:** Adding a nullable column does not affect existing reads or writes. MySQL online DDL can add a nullable column with `ALGORITHM=INPLACE, LOCK=NONE` on InnoDB without blocking. 300 rows makes this trivial.

---

## Step 2 — Blast Radius

### Table
```
partners
  id BIGINT PK
  partner_name VARCHAR(100)
  status VARCHAR(20)
  guide_certification_type VARCHAR(20) NULL  ← new column
  ...
```

### Indexes needed?
No. `guide_certification_type` is metadata only; no query filters on it at launch. Skip index creation — add it later if selectivity warrants it.

### Foreign keys?
No FK required — certification type is a simple code value.

### Mapper audit

**`PartnerMapper.xml`** — uses explicit `resultMap`:
```xml
<resultMap id="PartnerResultMap" type="Partner">
  <id property="id" column="id"/>
  <result property="partnerName" column="partner_name"/>
  <result property="status" column="status"/>
  <!-- Add: -->
  <!-- <result property="guideCertificationType" column="guide_certification_type"/> -->
</resultMap>
```
Status: **resultMap=safe** — unmapped column is silently ignored. Add mapping when the field is read.

**`PartnerSearchMapper.xml`** — uses `SELECT *` with `resultType`:
```xml
<select id="searchPartners" resultType="PartnerSummary">
  SELECT * FROM partners WHERE ...
</select>
```
Status: **needs check** — `resultType` mapping is positional/name-based. If `PartnerSummary` does not have a `guideCertificationType` field, MyBatis will attempt to set an unknown property and may throw a `ReflectionException` depending on the `mapUnderscoreToCamelCase` and `autoMappingBehavior` settings. Verify before deploy.

### Redis partner cache
Partner detail objects are cached under key pattern `partner:{id}`. Cached objects are serialized Java DTOs. Adding the new column does **not** invalidate existing cached objects (old DTOs simply lack the field). However, once the DTO is updated to include `guideCertificationType`, **stale cache entries will return null for the new field** until evicted.

Invalidation command (run after deploying the DTO update):
```bash
redis-cli --scan --pattern "partner:*" | xargs redis-cli del
```
For production use with Valkey cluster:
```bash
redis-cli -c --scan --pattern "partner:*" | xargs -L 100 redis-cli -c del
```

### Kafka
No Kafka events reference `partners` table directly in Phase 1. No schema change needed.

---

## Step 3 — Compliance

`guide_certification_type` is a business classification code (e.g., `CULTURAL`, `ADVENTURE`). It is **not PII** and does not contain personal, identity, or sensitive partner data.

No `/audit-compliance` flag required for this change alone.

---

## Step 4 — Artifacts

### Forward migration SQL
```sql
-- 가이드 자격증 유형 컬럼 추가 (신규 nullable 컬럼, 기존 데이터 영향 없음)
ALTER TABLE partners
  ADD COLUMN guide_certification_type VARCHAR(20) NULL
    COMMENT '가이드 자격증 유형 코드 (예: CULTURAL, ADVENTURE, FOOD — 초기값 NULL 허용)'
  ALGORITHM=INPLACE, LOCK=NONE;
```

### Rollback SQL
```sql
-- 롤백: guide_certification_type 컬럼 제거
ALTER TABLE partners
  DROP COLUMN guide_certification_type
  ALGORITHM=INPLACE, LOCK=NONE;
```

---

## Step 5 — Verification

1. **Confirm DDL executed cleanly:**
   ```sql
   SHOW COLUMNS FROM partners LIKE 'guide_certification_type';
   -- Expected: Field=guide_certification_type, Type=varchar(20), Null=YES, Default=NULL
   ```

2. **Confirm no table lock was held** (check slow query log or `SHOW PROCESSLIST` during migration window).

3. **Check PartnerSearchMapper resultType binding:**
   ```bash
   grep -r "PartnerSummary" services/domain-api/src --include="*.java" -l
   # Open PartnerSummary.java and verify no strict deserialization error will occur
   ```
   If `autoMappingBehavior=FULL` is configured and the DTO lacks the new field, add the field or change to `PARTIAL`.

4. **Invalidate Redis partner cache** (after DTO is deployed):
   ```bash
   redis-cli --scan --pattern "partner:*" | xargs redis-cli del
   ```
   Confirm cache miss + re-population via application log.

5. **Smoke test** — call `GET /api/partners/{id}` and confirm response includes `guideCertificationType: null` for existing records. Verify no 500 errors in logs.

---

## Why this is light review

- Nullable column: existing INSERTs and SELECTs continue to work without code change.
- Small table: DDL is sub-second; no need for `pt-online-schema-change`.
- No PII: no PIPA or consent implications.
- No downstream schema contracts change (Kafka, shared platform APIs).
- Rollback is a single `DROP COLUMN` — reversible in under a second.

The only real risk is the `PartnerSearchMapper` `SELECT *` / `resultType` interaction, which is caught in Step 5.3 above.
