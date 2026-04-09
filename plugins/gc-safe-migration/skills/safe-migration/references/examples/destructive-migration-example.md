<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->
<!-- Compliance reference: PIPA Article 21 (개인정보 파기 의무) -->

# Worked Example: Destructive Migration — PII Column Drop on `bookings`

**Scenario:** Drop `traveler_passport_number VARCHAR(30)` from the `bookings` table (~50K rows) after vault migration.  
Passport numbers are high-risk PII. PIPA Article 21 requires destruction of personal data when the retention purpose has been fulfilled.

---

## Step 1 — Classify

| Attribute        | Value                                                      |
|------------------|------------------------------------------------------------|
| Migration type   | Destructive (DROP COLUMN containing PII)                   |
| Table size       | ~50K rows — medium; DDL still fast with INPLACE            |
| Risk level       | Heavy                                                      |
| Review path      | Full — PIPA Article 21 destruction requirement applies     |
| Compliance flag  | YES — run `/audit-compliance` before executing             |

**Reasoning:** Dropping a column holding passport numbers is an irreversible, privacy-obligated action. Vault completeness must be verified first. The operation must be documented in the data destruction log.

---

## Step 2 — Blast Radius

### Table
```
bookings
  id BIGINT PK
  booking_code VARCHAR(20)
  traveler_id BIGINT
  traveler_passport_number VARCHAR(30)  ← dropping this column
  status VARCHAR(20)
  ...
```

### Mapper audit — three mappers affected

**`BookingMapper.xml`**
```xml
<!-- 영향: INSERT/resultMap에서 traveler_passport_number 제거 필요 -->
<insert id="createBooking" parameterType="Booking">
  INSERT INTO bookings (booking_code, traveler_id, traveler_passport_number, status)
  VALUES (#{bookingCode}, #{travelerId}, #{travelerPassportNumber}, #{status})
</insert>
```
Action: Remove `traveler_passport_number` from INSERT and resultMap before running DROP.

**`BookingDetailMapper.xml`**
```xml
<!-- 영향: SELECT에서 traveler_passport_number 컬럼 제거 필요 -->
<select id="getBookingDetail" resultMap="BookingDetailResultMap">
  SELECT id, booking_code, traveler_id, traveler_passport_number, status
  FROM bookings WHERE id = #{id}
</select>
```
Action: Remove column from SELECT and resultMap.

**`BookingExportMapper.xml`**
```xml
<!-- 영향: 내보내기 쿼리에서 여권번호 제거 — 가장 민감한 매퍼 -->
<select id="exportBookings" resultType="BookingExportRow">
  SELECT id, booking_code, traveler_passport_number, ...
  FROM bookings WHERE created_at BETWEEN #{from} AND #{to}
</select>
```
Action: Remove column from export SELECT. This is the highest-risk mapper — exporting passport numbers to downstream systems must stop before drop.

### Redis cache
Booking detail objects cached under `booking:{id}` may include `travelerPassportNumber`. After DTO update removes the field, cached objects with the old field will be harmless but should be invalidated to prevent confusion in debugging.

### Kafka event schema
`BookingCreatedEvent` and `BookingDetailSyncEvent` reference `travelerPassportNumber` in their Avro schemas. Both schemas must be updated and backward-compatibility verified before column drop.

---

## Step 3 — Compliance

**PIPA Article 21 applies.** Passport number is sensitive personal data. Destruction is required once the vault migration is confirmed complete.

Required actions:
- Document the destruction event in the internal data processing records (개인정보 처리방침 기록).
- Confirm vault holds all passport numbers with cryptographic completeness check.
- Retain a destruction log with: table name, column name, row count, destruction timestamp, executor.
- Run `/audit-compliance` and confirm PIPA Article 21 destruction procedure is satisfied before proceeding.

---

## Step 4 — Artifacts

### Vault completeness verification
```sql
-- 금고(vault) 이관 완료 여부 확인: 여권번호가 있는 예약 건 중 vault에 없는 건 조회
SELECT b.id, b.booking_code
FROM bookings b
LEFT JOIN vault_passport_index v ON v.booking_id = b.id
WHERE b.traveler_passport_number IS NOT NULL
  AND v.booking_id IS NULL;
-- Expected: 0 rows. If any rows returned, STOP — vault migration is incomplete.
```

### Backup table creation (before DROP)
```sql
-- 삭제 전 여권번호 백업 테이블 생성 (30일 보존 후 별도 파기 처리)
CREATE TABLE bookings_passport_backup_20260409 AS
  SELECT id, booking_code, traveler_passport_number, NOW() AS backed_up_at
  FROM bookings
  WHERE traveler_passport_number IS NOT NULL;

-- 백업 건수 확인
SELECT COUNT(*) FROM bookings_passport_backup_20260409;
```

### DROP migration SQL
```sql
-- 여권번호 컬럼 파기 (PIPA 제21조 파기 의무 이행)
-- 선행 조건: vault 완료 확인, 백업 테이블 생성, 매퍼 배포 완료
ALTER TABLE bookings
  DROP COLUMN traveler_passport_number
  ALGORITHM=INPLACE, LOCK=NONE;
```

### Rollback note
**THIS OPERATION IS NOT REVERSIBLE.**

Once `traveler_passport_number` is dropped from `bookings`:
- The column data is gone from the live table.
- Recovery requires reading from `bookings_passport_backup_20260409` and re-adding the column — this is a last-resort emergency procedure only, not a standard rollback.
- If vault is complete, there is no legitimate reason to recover the column to the live table.

Emergency recovery (if required by incident response only):
```sql
-- 긴급 복구 절차 (운영 인시던트 대응 전용 — 사전 승인 필요)
ALTER TABLE bookings
  ADD COLUMN traveler_passport_number VARCHAR(30) NULL
  ALGORITHM=INPLACE, LOCK=NONE;

UPDATE bookings b
JOIN bookings_passport_backup_20260409 bk ON bk.id = b.id
SET b.traveler_passport_number = bk.traveler_passport_number;
```
Note: rows inserted *after* the DROP will not have data in the backup — recovery is partial.

---

## Step 5 — Verification (10 steps)

1. **Vault completeness** — run the verification query above. Must return 0 rows before proceeding.

2. **Backup row count matches** — confirm `COUNT(*)` from `bookings_passport_backup_20260409` equals `COUNT(*)` of non-null passport rows in `bookings`.

3. **Mapper deployment** — confirm all three mapper XMLs (`BookingMapper`, `BookingDetailMapper`, `BookingExportMapper`) have been deployed without the `traveler_passport_number` column reference. Validate with:
   ```bash
   grep -r "traveler_passport_number\|travelerPassportNumber" \
     services/domain-api/src --include="*.xml" --include="*.java"
   # Expected: 0 matches after deploy
   ```

4. **Kafka schema update** — confirm `BookingCreatedEvent` and `BookingDetailSyncEvent` Avro schemas no longer include `travelerPassportNumber`. Run schema registry compatibility check.

5. **Cache invalidation** — flush booking caches:
   ```bash
   redis-cli --scan --pattern "booking:*" | xargs redis-cli del
   ```

6. **Run DROP migration** — execute the ALTER TABLE statement in the migration tool (Flyway/Liquibase). Confirm no lock timeout in slow query log.

7. **Post-drop schema check:**
   ```sql
   SHOW COLUMNS FROM bookings LIKE 'traveler_passport_number';
   -- Expected: Empty set (column no longer exists)
   ```

8. **Application smoke test** — call `POST /api/bookings` and `GET /api/bookings/{id}` and confirm no 500 errors, no references to passport in response payloads.

9. **Destruction log entry** — record in the personal-data destruction register:
   - Table: `bookings`, Column: `traveler_passport_number`
   - Row count destroyed: (value from step 2)
   - Timestamp: (execution timestamp)
   - Executor: (engineer name / pipeline run ID)

10. **30-day backup retention** — schedule destruction of `bookings_passport_backup_20260409` at T+30 days. Add a calendar reminder or Jira ticket.

---

## Key Lessons

- **Never drop a PII column without vault/archive completeness proof.** One missed row means a PIPA violation.
- **The export mapper is the most dangerous.** If `BookingExportMapper` is not updated before DROP, the export job will 500 and may expose that passport data was in scope.
- **Kafka schema must move first.** If the consumer still expects `travelerPassportNumber`, it will fail silently or throw deserialization errors.
- **Backup tables are not indefinite.** They must themselves be destroyed on schedule — don't let a "temporary" backup become a permanent PII store.
- **Document the destruction.** PIPA Article 21 requires a record. A migration script execution alone is not sufficient documentation.
