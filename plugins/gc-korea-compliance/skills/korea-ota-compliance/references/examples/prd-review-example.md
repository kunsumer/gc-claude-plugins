# Worked Example: PRD Review — Partner Onboarding with Settlement Information

<!-- Last reviewed: 2026-04-09 -->

This example walks through a complete compliance review of a product requirements document
covering partner registration. It shows how to classify scope for a PRD, generate
proportionate findings, identify escalation needs, and summarise results.

---

## Input

**Artifact type:** PRD review
**Surface:** `apps/partner-center` — Partner Registration Flow
**Document title:** "Partner Onboarding v1.2 — Registration and Settlement Profile Setup"

**Relevant PRD excerpt:**

> **Partner Registration Flow**
>
> Partners register through the Unified Account Platform (UAP), which handles credential
> creation and identity verification. Once the UAP auth handoff is complete, partners are
> directed to the Partner Center to complete their business profile.
>
> **Data collected in the Partner Center registration flow:**
> - Business name (법인명 or 상호)
> - Representative name (대표자 성명)
> - Business registration number (사업자등록번호)
> - Bank account information: bank name, account number, account holder name
> - Contact person information: name, phone number, email address
> - Guide certification document upload (if sole proprietor or freelance guide)
>
> **Settlement:**
> Monthly KRW settlement. After each settlement cycle, a settlement summary and
> supporting transaction data are sent to the Finance Platform for reconciliation
> and downstream accounting.
>
> **Data flow:**
> UAP → Partner Center (business profile) → Finance Platform (settlement data)
>
> **Out of scope for this PRD:**
> Credential management and password recovery (UAP responsibility).
> Tax invoice issuance (Finance Platform responsibility).

---

## Step 1 — Classify

| Dimension | Assessment |
|---|---|
| **Business role** | Intermediary — the OTA collects partner data to facilitate the marketplace relationship, not as the merchant of record. However, settlement data may create a secondary financial-processing role that warrants scrutiny. |
| **Data classification** | Multiple PII categories: bank account information (금융 개인정보 — high-risk financial PII), representative name + contact person name/phone/email (일반 개인정보), business registration number (사업자등록번호 — may be linked to RRN for sole proprietors; see Finding 4). |
| **Payment role** | Not a PG role. The OTA collects bank account info for settlement dispatch, not for payment processing. EFTA card-handling rules do not apply. Settlement transfer to Finance Platform is in scope. |
| **Scope gates active** | PIPA (collection, consent, retention, transfer), ISMS-P (credential handoff, access control for sensitive fields), E-Commerce Act (intermediary disclosure obligations). |
| **Overseas transfer** | Not indicated. Finance Platform is assumed domestic. If Finance Platform or any downstream system is overseas, PIPA cross-border transfer rules apply. |
| **Sole proprietor nuance** | Sole proprietors and freelance guides are individuals. Their 사업자등록번호 may be constructed from or linked to their 주민등록번호 (RRN). This elevates data sensitivity beyond what a corporate registration number implies. |

Rule sets loaded: **PIPA**, **ISMS-P**, **E-Commerce Act (intermediary provisions)**.

---

## Step 2 — Findings

### Finding 1

```text
Severity : REVIEW
Rule     : PIPA-005 — Data transfer to Finance Platform: entrustment vs. third-party provision
Location : PRD §Settlement — "sent to Finance Platform for reconciliation and downstream accounting"
```

**What the PRD says:** Settlement data (including, by implication, bank account number and
account holder name) is sent to the Finance Platform at the end of each settlement cycle.

**왜 문제인가 (Why this matters):**
개인정보보호법 제17조(제3자 제공)와 제26조(업무 위탁)는 개인정보의 외부 이전 방식에 따라
요구사항이 달라집니다.

- **위탁(Entrustment, 제26조):** 수탁자(Finance Platform)가 위탁자(the OTA)의 지시에 따라
  데이터를 처리하는 경우. 이 경우 개인정보 처리방침에 수탁자 정보를 공개하고,
  위탁 계약서에 보안 관리 조항을 포함하면 됩니다. 별도의 정보주체 동의는 불필요합니다.

- **제3자 제공(Third-party provision, 제17조):** Finance Platform이 자신의 목적을 위해
  데이터를 활용하는 경우(예: 자사 회계 시스템에 파트너를 등록하는 행위 등). 이 경우
  정보주체(파트너)의 별도 동의 또는 법령상 근거가 필요합니다.

현재 PRD만으로는 Finance Platform이 수탁자인지 제3자인지 결정할 수 없습니다.
잘못 분류할 경우 법적 근거 없는 개인정보 제공에 해당하여 시정명령 및 과징금 대상이 됩니다.

**Required action:** Escalate to legal/privacy review. The product and legal teams must jointly
determine whether the Finance Platform relationship is entrustment or third-party provision,
then document it in the data-transfer register and update the privacy policy accordingly.

---

### Finding 2

```text
Severity : WARN
Rule     : PIPA-009 — No retention period specified for settlement data
Location : PRD — data collected and transferred, no lifecycle noted
```

**What the PRD says:** The PRD specifies data collection and transfer but is silent on how long
bank account details, settlement summaries, and transaction records are retained, and what
happens to them when a partner account is closed.

**왜 문제인가 (Why this matters):**
개인정보보호법 제21조는 개인정보의 보유 기간이 경과하거나 처리 목적이 달성되면
해당 개인정보를 지체 없이 파기하도록 규정합니다.
또한 제22조에 따라 정보주체에게 수집 시 보유 기간을 고지해야 합니다.

정산 데이터(은행계좌, 정산 내역)의 경우 조세특례제한법 및 부가가치세법에 따라
5년간 보관 의무가 있을 수 있습니다. 이 법령 근거를 명시하지 않으면 보유 기간 고지 의무를
이행하지 않은 것으로 판단될 수 있습니다.

**Fix:** The PRD must specify:
- Retention period for bank account data (suggested: duration of active partnership + legally
  required tax record period, likely 5 years from last settlement).
- Retention period for contact person data (suggested: duration of active partnership + grace
  period for dispute resolution).
- Deletion or anonymisation procedure when a partner account is closed.

---

### Finding 3

```text
Severity : WARN
Rule     : ECOM-002 — Intermediary role disclosure timing for partners
Location : PRD — partner registration flow, no disclosure step described
```

**What the PRD says:** The registration flow collects business data and routes partners to the
Finance Platform for settlement. There is no mention of when or how the OTA's role as an
intermediary (and not the merchant of record) is disclosed to the registering partner.

**Why this matters:**
The E-Commerce Act requires that marketplace operators make their role in the transaction clear
to the parties involved. Partners need to understand that the OTA acts as an intermediary,
that they retain merchant-of-record responsibility for consumer claims, and what the settlement
and commission structure is — before they complete registration and begin listing products.

**Fix:** The PRD should include a step in the registration flow where the partner explicitly
acknowledges:
- The OTA's intermediary role.
- That the partner is the merchant of record for their listings.
- The commission structure and settlement terms (or a reference to the partner agreement that
  contains these terms).

This acknowledgement should be logged as a timestamped consent record.

---

### Finding 4

```text
Severity : INFO
Rule     : CORE-002 — Sole proprietor detection: business registration number may be RRN-linked
Location : PRD — "Business registration number (사업자등록번호)" and "sole proprietor or freelance guide"
```

**What the PRD says:** The PRD collects 사업자등록번호 for all partner types, including sole
proprietors and freelance guides.

**Note for implementation:**
For sole proprietors (개인사업자), the 사업자등록번호 is constructed using the proprietor's
주민등록번호(RRN) as a seed component. While the registration number itself is not the full RRN,
systems that cross-reference 사업자등록번호 against other data sources may be able to derive or
confirm the proprietor's RRN.

This does not block the PRD, but the implementation team should:
- Not store 사업자등록번호 and RRN together in the same unprotected table or log line.
- Apply the same access-control and encryption standards to 사업자등록번호 for sole proprietors
  as would apply to RRN (고유식별정보 standards under PIPA Article 24).
- Confirm with legal whether the guide certification document upload may contain RRN
  (many Korean certificates embed the holder's RRN), and if so, apply appropriate masking.

---

## Step 3 — Uncertain findings

**Finding 1 (PIPA-005)** remains at **REVIEW** severity because the business and legal
relationship between the OTA and the Finance Platform cannot be determined from the PRD
alone. This is not a reviewer failure — it is a deliberate escalation. The classification
(entrustment vs. third-party provision) must come from legal and product, not from a
compliance reviewer reading the PRD.

If further information reveals that the Finance Platform acts purely as an internal processing
team within the same legal entity, the classification changes and the finding may be downgraded.

---

## Step 4 — Summary

| Severity | Count | Finding IDs |
|---|---|---|
| BLOCK | 0 | — |
| REVIEW | 1 | PIPA-005 |
| WARN | 2 | PIPA-009, ECOM-002 |
| INFO | 1 | CORE-002 |

**Verdict:** This PRD may proceed to design and implementation planning. The REVIEW and WARN
findings must be resolved before the feature ships.

**Escalations required:** 1
- PIPA-005: Legal/privacy team must classify the Finance Platform data-transfer relationship
  and update the data-transfer register and privacy policy.

**Blockers:** 0

**Watchlist — regulatory calendar item:**
The E-Commerce Act amendments effective **2026-07-21** include updated provisions for
marketplace operator disclosure obligations to both consumers and business partners.
This partner onboarding flow, and specifically the intermediary-role disclosure step
(ECOM-002), should be reviewed against the amended provisions before the 2026-09-07 launch.
Assign ownership to the legal/policy team to confirm no additional steps are required.
