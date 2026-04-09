---
description: Run a Korea OTA security, privacy, and compliance audit and produce a decision-ready AUDIT_REPORT.
argument-hint: [prd|ux|architecture|code] [feature-or-scope]
---

# /audit-compliance

Audit: $ARGUMENTS

## Goal
Produce `AUDIT_REPORT.md` with a grounded compliance verdict for a change,
feature, design, or code scope under Korean OTA regulations.

## Use when
- personal data is created, edited, displayed, exported, stored, logged, or transmitted
- admin privileges, approval flows, or privileged actions change
- traveler, partner, settlement, document, or consent handling changes
- a third-party, supplier, overseas, or shared-system boundary matters
- checkout, booking, cancellation, voucher, marketing, location, or scoring behavior changes

## Read first
- project compliance documentation (if available)
- project boundary/ownership documentation (if available)
- this plugin's `korea-ota-compliance` skill
- active spec files and touched files

## Workflow

### 1. Classify scope before applying rules
Determine and record:
- review type: PRD / UX / architecture / code
- business role: intermediary / seller / package-travel organizer / mixed
- data categories involved
- payment role
- scope gates: whether location, overseas transfer, marketing, automated decisioning, or Credit Information Act review actually apply

Do not apply a rule whose `Applies when` condition is not satisfied.

### 2. Load the relevant rules only
Use the matrix in the `korea-ota-compliance` skill.
Do not load the full ruleset when a narrower slice is enough.

### 3. Review controls and evidence
Check:
- collection basis and consent separation
- third-party provision vs entrustment vs direct collection by another controller
- overseas transfer basis and disclosure
- logging, masking, retention, and deletion
- authn/authz and audit trails
- OTA consumer disclosure and booking / cancellation obligations
- location, marketing, scoring, or payment-specific duties if applicable

### 4. Record findings
Each finding must include:
- severity: BLOCK / REVIEW / WARN / INFO
- rule ID
- file, endpoint, screen, or PRD section
- what is wrong and why
- concrete remediation
- authority: statute, regulator guidance, or internal baseline

### 5. Deliver verdict
State one of:
- **PASS**
- **PASS WITH REQUIRED FIXES**
- **BLOCK**

## Non-negotiables
- Do not over-apply rules to scopes where they do not legally or operationally fit.
- Do not mark an item PASS without evidence.
- Do not present internal baselines as if they were statutory requirements.
- Do not hide uncertainty; escalate it.

## Output
Write or update:
- `AUDIT_REPORT.md` in the project's active spec folder or working directory

## Required structure
1. Scope audited
2. Review type and classification
3. Data classes and actors
4. Applicable controls
5. Findings by severity
6. Required fixes and evidence
7. Escalations
8. Watchlist
9. Verdict
