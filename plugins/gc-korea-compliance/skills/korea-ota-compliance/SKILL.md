---
name: korea-ota-compliance
description: >
  Korea OTA security, privacy, and regulatory compliance review for PRD, UX,
  solution architecture, and code. Use whenever a change touches personal data,
  payments, location, marketing, bookings, cancellations, cross-border transfers,
  consent flows, admin authority, or supplier data transfer.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Task
user-invocable: false
---

# Korea OTA Compliance Review Skill

Run security, privacy, and regulatory-compliance checks against Korean law and
internal baselines for a React/Next.js, Spring Boot/Java, MySQL stack.

## Before you review anything

1. Identify the project's compliance and boundary documentation if available.
2. Read `references/rules.md` — the detailed Korea OTA ruleset.
   It is intentionally long. Load only the sections relevant to the review type.
3. Read `references/checklists.md` — the quick scan checklist for the selected
   review type.
4. Optionally read `references/detection-heuristics.md` during code review for
   grep patterns and false-positive handling.

## Review-type → relevant rule sections

| Review type     | Always load                        | Load if relevant                              |
|-----------------|------------------------------------|-----------------------------------------------|
| **PRD**         | CORE-*, PIPA-004–009, ECOM-*, TOUR-* | LIA-* (if location), NET-* (if marketing), CREDIT-* (if scoring) |
| **UX**          | CORE-001, PIPA-004, PIPA-007–008, ECOM-002–004, NET-001–002, LIA-001 | PIPA-006 (overseas notice), NET-005 (cookies) |
| **Architecture**| CORE-*, PIPA-001–003, PIPA-005–006, PIPA-011, ISMS-*, EFTA-* | LIA-002–004 (if location), CREDIT-* (if scoring) |
| **Code**        | ISMS-*, PIPA-001–003, EFTA-001, PIPA-004 | Everything else based on file context          |

## Review workflow

### Step 1 — Classify before applying rules

Before flagging anything, determine:
1. **Business role** — intermediary, seller, package-travel organizer, or mixed.
2. **Data classification** — which personal-data categories exist, why, and who the data subject is.
3. **Payment role** — PG integration only, merchant-of-record, or regulated payment role.
4. **Scope gates** — whether Credit Information Act, Location Information Act filing, or automated-decision review actually apply.

Do not apply a rule whose `Applies when` condition is not satisfied.

### Step 2 — Run the review

For each finding, output a structured block:

```text
[SEVERITY] RULE-ID: One-line summary
  File/section: <path or PRD section>
  Detail: What is wrong and why
  Fix: Concrete remediation
  Authority: Statute or internal baseline
```

Severity meanings:
- **BLOCK** — Must fix before approval / merge / launch.
- **REVIEW** — Escalate to legal, privacy, security, or finance.
- **WARN** — Strong risk signal; fix or get explicit sign-off.
- **INFO** — Best practice or guidance note.

### Step 3 — Handle uncertainty

- If legal applicability is unclear, output `REVIEW` or `ESCALATE` rather than inventing a requirement.
- Regex matches and field names are heuristics, not legal conclusions.
- Distinguish black-letter law from internal baselines.
- Treat future-effective watchlist items as watchlist only until their effective dates.

### Step 4 — Summarize

End every review with:
1. Findings table
2. Blockers count
3. Escalations
4. Watchlist

## Important guardrails for the reviewing model

- Do not treat every data transfer as entrustment — check actual controller/processor role.
- Do not assume `.kr` means domestic or non-`.kr` means overseas.
- Do not apply a blanket 7-day withdrawal right to accommodation or date-specific travel inventory.
- Do not apply Credit Information Act just because payment history exists.
- Do not describe internal baselines like AES-256, KMS, or rotation frequency as statutory mandates unless the law says so.
- Do not auto-classify every ranking or personalization surface as article 37-2 automated decisioning.
- Do not require blanket cookie opt-in where Korean law instead requires notice + refusal by default.
- Do not delete a stricter internal control just because the statute is silent.

## Explaining findings to engineers

Most users of this skill are engineers with little regulatory knowledge. Every
finding must be understandable and actionable without legal training.

Rules for findings:
- Include a **"Why this matters"** line explaining the real-world consequence in
  plain language (Korean or English). Example: "이 개인정보가 로그에 남으면,
  개인정보 유출 시 과징금 대상이 될 수 있습니다."
- Reference the specific statute article (e.g., "PIPA Article 29"), not just the
  internal rule ID (e.g., "PIPA-001")
- Suggest the **simplest effective fix**, not the most comprehensive one
- When a finding requires architectural decisions, say so explicitly rather than
  prescribing one approach

Read `references/examples/` for complete worked examples of good findings.
Read `references/anti-patterns.md` for common mistakes to avoid.
Read `references/severity-calibration.md` to calibrate severity levels.

## Self-review checklist

After producing findings, run this checklist against your own output before
presenting it. Fix any issues found.

1. Did I classify the business role before applying any rules?
2. Did I check scope gates before flagging location, scoring, or credit rules?
3. Does every finding have a specific file path or PRD section reference?
4. Did I distinguish statute from internal baseline in every finding?
5. Are there findings where I'm uncertain the rule applies? → Change to REVIEW
6. Did I explain each finding so an engineer without legal training understands?
7. Did I check my findings against `references/anti-patterns.md`?

## Graduated complexity

Assess the review complexity and adjust depth accordingly:

**Light review** — single file change, no new data flows, no PII fields:
- Load only ISMS-* rules
- Skip the full classification ceremony
- Produce compact output (findings only, no elaborate summary)

**Standard review** — multiple files, existing data flows modified:
- Full classification (business role, data categories, payment role, scope gates)
- Load targeted rule sections per the review-type matrix
- Standard output format with findings table and summary

**Deep review** — new data flows, cross-border transfer, consent changes, new PII
collection, new third-party integrations:
- Full classification with explicit documentation of each determination
- Load all relevant rule sections
- Worked-through analysis showing reasoning for each finding
- Explicit escalation recommendations for ambiguous items
- Reference pressure tests in `references/pressure-tests/` to verify edge-case behavior
