<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A -->

# Worked Example: Expand-Migrate-Contract — Table Split on `product_options`

**Scenario:** Split `product_options` (~5K rows) into `option_groups` and `option_items` to support a multi-level option hierarchy (1–10 depth, JOIN/PRIVATE product types, quantity + condition options).

This is a structural refactor using the three-phase expand-migrate-contract pattern. No PII involved.

---

## Step 1 — Classify

| Attribute        | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Migration type   | Structural refactor (table split via expand-migrate-contract)         |
| Table size       | ~5K rows — small, but logic complexity is high                        |
| Risk level       | Heavy                                                                 |
| Review path      | Full — multiple services, mappers, and Kafka schemas affected         |
| Compliance flag  | No — not PII, no consent or settlement implications                   |

**Reasoning:** The expand-migrate-contract pattern is required because the old `product_options` table is actively read and written by four services in production. A direct DROP would cause downtime. Instead: expand (add new tables, dual-write), migrate (backfill + switch reads), contract (remove old table).

---

## Step 2 — Blast Radius

### Source table
```
product_options
  id BIGINT PK
  product_id BIGINT FK → products.id
  option_name VARCHAR(100)
  option_type VARCHAR(20)       -- QUANTITY | CONDITION
  price_delta INT
  inventory_count INT
  sort_order INT
  ...
```

### New tables
```
option_groups
  id BIGINT PK
  product_id BIGINT FK → products.id
  group_name VARCHAR(100)
  group_type VARCHAR(20)        -- JOIN | PRIVATE
  sort_order INT

option_items
  id BIGINT PK
  group_id BIGINT FK → option_groups.id
  item_name VARCHAR(100)
  option_type VARCHAR(20)       -- QUANTITY | CONDITION
  price_delta INT
  inventory_count INT
  sort_order INT
```

### Mapper audit — 4 mappers

| Mapper                        | Impact                                                          |
|-------------------------------|-----------------------------------------------------------------|
| `ProductOptionMapper.xml`     | Primary CRUD — must dual-write and then switch reads            |
| `ProductSearchMapper.xml`     | Option join in search results — switch after backfill verified  |
| `BookingOptionMapper.xml`     | Reads option price/inventory at booking time — critical path    |
| `InventoryMapper.xml`         | Deducts `inventory_count` — must point to `option_items` after  |

### Service audit — 4 services

| Service               | Impact                                           |
|-----------------------|--------------------------------------------------|
| `domain-api`          | Product/option CRUD, inventory deduction         |
| `partner-api`         | Partner Center option management                 |
| `web-api`             | B2C product detail page option display           |
| `batch`               | Review/sync jobs that may read option metadata   |

### Redis
Product detail cache includes option data under `product:{id}:options`. Must be invalidated after read switchover.

### Kafka
`ProductUpdatedEvent` includes `options[]` array. Schema must be updated to reflect `optionGroups[].optionItems[]` after Phase 2.

---

## Step 3 — Compliance

`option_groups` and `option_items` contain product pricing and option structure only. No PII, no settlement data, no consent implications.

No `/audit-compliance` flag needed.

---

## Phase 1 — Expand

**Goal:** Add new tables. Begin dual-writing to both old and new tables. Reads still come from `product_options`.

### Create `option_groups`
```sql
-- 옵션 그룹 테이블 생성 (Phase 1 Expand)
CREATE TABLE option_groups (
  id          BIGINT      NOT NULL AUTO_INCREMENT COMMENT '옵션 그룹 고유 식별자',
  product_id  BIGINT      NOT NULL                COMMENT '상품 ID (products.id FK)',
  group_name  VARCHAR(100) NOT NULL               COMMENT '옵션 그룹명 (예: 인원 유형, 투어 시간)',
  group_type  VARCHAR(20) NOT NULL                COMMENT '그룹 유형 코드 (JOIN | PRIVATE)',
  sort_order  INT         NOT NULL DEFAULT 0      COMMENT '정렬 순서',
  created_at  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  INDEX idx_option_groups_product_id (product_id),
  CONSTRAINT fk_option_groups_product
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='상품 옵션 그룹 — 하나의 상품에 여러 옵션 그룹 가능';
```

### Create `option_items`
```sql
-- 옵션 아이템 테이블 생성 (Phase 1 Expand)
CREATE TABLE option_items (
  id              BIGINT  NOT NULL AUTO_INCREMENT COMMENT '옵션 아이템 고유 식별자',
  group_id        BIGINT  NOT NULL               COMMENT '옵션 그룹 ID (option_groups.id FK)',
  item_name       VARCHAR(100) NOT NULL          COMMENT '옵션 아이템명 (예: 성인, 아동)',
  option_type     VARCHAR(20) NOT NULL           COMMENT '옵션 유형 코드 (QUANTITY | CONDITION)',
  price_delta     INT     NOT NULL DEFAULT 0     COMMENT '기본가 대비 가격 조정값 (원화)',
  inventory_count INT     NOT NULL DEFAULT 0     COMMENT '재고 수량 (결제 완료 시 차감)',
  sort_order      INT     NOT NULL DEFAULT 0     COMMENT '정렬 순서',
  created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  INDEX idx_option_items_group_id (group_id),
  CONSTRAINT fk_option_items_group
    FOREIGN KEY (group_id) REFERENCES option_groups(id) ON DELETE RESTRICT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='상품 옵션 아이템 — 옵션 그룹의 말단 선택 단위, 재고 차감 기준';
```

### Application change in Phase 1
Update `ProductOptionMapper.xml` and the application layer to **dual-write**: every INSERT/UPDATE to `product_options` also writes to `option_groups` + `option_items`. Reads still use `product_options`.

### Phase 1 rollback
```sql
-- Phase 1 롤백: 신규 테이블 제거 (dual-write 코드 rollback 후 실행)
DROP TABLE IF EXISTS option_items;
DROP TABLE IF EXISTS option_groups;
```

---

## Phase 2 — Migrate

**Goal:** Backfill `option_groups` and `option_items` from existing `product_options` data. Verify consistency. Switch reads to new tables.

### Backfill — option_groups
```sql
-- 기존 product_options에서 option_groups 백필 (배치 단위: 1000건)
-- 참고: product_options의 option_type이 그룹 유형에 해당하지 않으므로 DEFAULT 값 사용
INSERT INTO option_groups (product_id, group_name, group_type, sort_order, created_at, updated_at)
SELECT DISTINCT
  product_id,
  '기본 옵션 그룹'  AS group_name,
  'JOIN'           AS group_type,
  0                AS sort_order,
  NOW(),
  NOW()
FROM product_options
ORDER BY product_id
LIMIT 1000;
-- 반복 실행하여 전체 이관 완료; 각 실행 후 SLEEP(0.1) 권장
```

### Backfill — option_items (batch pattern)
```sql
-- option_items 백필 — 1000건씩 처리 후 0.1초 대기
INSERT INTO option_items (group_id, item_name, option_type, price_delta, inventory_count, sort_order, created_at, updated_at)
SELECT
  og.id           AS group_id,
  po.option_name  AS item_name,
  po.option_type,
  po.price_delta,
  po.inventory_count,
  po.sort_order,
  po.created_at,
  po.updated_at
FROM product_options po
JOIN option_groups og ON og.product_id = po.product_id
ORDER BY po.id
LIMIT 1000 OFFSET 0;
-- OFFSET을 1000씩 증가시키며 반복; SELECT COUNT(*) FROM product_options로 총 건수 확인
-- 각 배치 실행 사이에 SLEEP(0.1) 추가하여 슬레이브 레그 방지
```

### Consistency verification
```sql
-- 이관 정합성 검증: option_items 건수가 product_options 건수와 일치해야 함
SELECT
  (SELECT COUNT(*) FROM product_options)  AS source_count,
  (SELECT COUNT(*) FROM option_items)     AS migrated_count,
  (SELECT COUNT(*) FROM product_options) - (SELECT COUNT(*) FROM option_items) AS delta;
-- Expected: delta = 0
```

### Switch reads
Update all four mappers to read from `option_groups` JOIN `option_items` instead of `product_options`. Deploy application. Dual-write continues during this phase.

### Phase 2 rollback
Switch reads back to `product_options` in mapper configs. No data change needed — `product_options` was continuously dual-written during Phase 2.

---

## Phase 3 — Contract

**Goal:** Remove `product_options` after a 7-day stability window. Stop dual-write.

**Wait 7 days** after Phase 2 switchover with no errors before proceeding.

### Backup before DROP
```sql
-- 삭제 전 백업 테이블 생성 (30일 보존)
CREATE TABLE product_options_backup_20260409 AS
  SELECT *, NOW() AS backed_up_at FROM product_options;

SELECT COUNT(*) FROM product_options_backup_20260409;
-- 원본 건수와 일치 확인
```

### DROP original table
```sql
-- dual-write 코드 제거 배포 완료 후 실행
DROP TABLE product_options;
```

### Phase 3 rollback — from backup
```sql
-- Phase 3 롤백 (비상시만): 백업에서 원본 테이블 복원
CREATE TABLE product_options AS
  SELECT id, product_id, option_name, option_type, price_delta,
         inventory_count, sort_order, created_at, updated_at
  FROM product_options_backup_20260409;

ALTER TABLE product_options ADD PRIMARY KEY (id);
ALTER TABLE product_options MODIFY id BIGINT NOT NULL AUTO_INCREMENT;
-- 이후 매퍼 읽기를 product_options로 재전환
```
Note: rows inserted after backup creation will not be in the backup — recovery is partial.

### 30-day backup cleanup
```sql
-- T+30일 이후 실행 (별도 일정 등록 필요)
DROP TABLE product_options_backup_20260409;
```

---

## Verification Checklist

### Phase 1
- [ ] `option_groups` table created with correct schema and FK
- [ ] `option_items` table created with correct schema and FK
- [ ] `SHOW CREATE TABLE option_groups` and `option_items` confirmed
- [ ] Dual-write code deployed and no errors in application log
- [ ] Write to `product_options` triggers write to both new tables — verified by spot-check query

### Phase 2
- [ ] Backfill completed — consistency verification query returns `delta = 0`
- [ ] No duplicate rows in `option_groups` per product
- [ ] All four mappers updated and deployed (reads switched)
- [ ] `BookingOptionMapper` verified — booking flow reads correct `inventory_count` from `option_items`
- [ ] Redis product option cache invalidated:
  ```bash
  redis-cli --scan --pattern "product:*:options" | xargs redis-cli del
  ```
- [ ] `ProductUpdatedEvent` Kafka schema updated to `optionGroups[].optionItems[]`
- [ ] No 500 errors on B2C product detail page
- [ ] Partner Center option management smoke tested

### Phase 3
- [ ] 7-day stability window passed with no option-related errors in Sentry/logs
- [ ] Backup table created and row count verified
- [ ] Dual-write code removed and deployed
- [ ] `DROP TABLE product_options` executed
- [ ] `SHOW TABLES LIKE 'product_options'` returns empty set
- [ ] Full E2E booking flow smoke tested end-to-end
- [ ] 30-day backup cleanup task scheduled
