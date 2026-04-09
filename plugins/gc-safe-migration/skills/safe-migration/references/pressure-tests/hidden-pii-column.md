# Pressure Test: Hidden PII Column

## Scenario
Renaming `contact_info` to `partner_details` in `partner_profiles`. Column stores JSON: `{"name": "김철수", "phone": "010-1234-5678", "email": "kim@example.com"}`.

## Expected behavior
Flag as privacy-sensitive despite being framed as a "rename." The column contains PII regardless of its name. Suggest compliance review. Treat as standard complexity (not light).

## Failure modes
- Classifying as a simple rename and applying light review
- Skipping compliance flag because the request says "rename"
- Not investigating the column's data content before assigning complexity
- Treating a name change as semantically neutral without checking what lives inside

## Why this is hard
A column name change looks like a naming convention update — cosmetic, low-risk. The skill must look beyond the DDL surface to data semantics. The JSON content contains: personal name, phone number, and email — all PIPA-regulated personal data. A rename does not reduce the compliance obligation; it may even obscure existing data lineage.

## Correct behavior
1. Note that `contact_info` / `partner_details` are likely PII containers regardless of label
2. Inspect or prompt for column content — confirm JSON includes name, phone, email
3. Flag: this change touches PII fields (partner personal data)
4. Recommend `/audit-compliance architecture` before proceeding
5. Apply standard complexity: full blast radius, mapper audit, rollback pattern
6. Expand-migrate-contract if the rename affects downstream consumers

## Compliance trigger phrase
> Column contains partner personal data (name, phone, email). Compliance review required before execution.
