<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A (behavioral guidance) -->

# Compliance Review Anti-Patterns

These are documented failure modes observed in model-generated compliance reviews. Check your output against this list before presenting findings.

---

## 1. Everything is entrustment

### What goes wrong
The review treats every data transfer to a third party as PIPA Article 26 entrustment, triggering entrustment contract requirements, disclosure obligations, and supervision duties — regardless of whether the recipient is actually acting as a processor on behalf of the controller.

### Why it's wrong
PIPA distinguishes entrustment (위탁) from third-party provision (제3자 제공). Entrustment requires the recipient to process data solely on the controller's behalf and under the controller's instructions. If the recipient determines its own purpose or receives data to fulfill its own obligations, the relationship is third-party provision, not entrustment. Mislabeling the relationship leads to wrong legal obligations and missed disclosure requirements.

### Correction
Apply a 3-step check before labeling any transfer:
1. Who controls the purpose and means of processing — the transferor, the recipient, or both?
2. Does the recipient use the data for its own independent purpose (e.g., settlement, fraud detection, credit assessment)?
3. Did the data subject provide the data directly to the recipient as part of the transaction?

If the recipient has its own purpose (step 2) or the data subject dealt with the recipient directly (step 3), the transfer is third-party provision. Document the actual relationship, not the assumed one.

---

## 2. Blanket 7-day withdrawal right

### What goes wrong
The review applies the E-Commerce Act (전자상거래법) 7-day withdrawal right (청약철회) to all OTA products, flagging missing withdrawal UI or requiring unconditional refund within 7 days of purchase.

### Why it's wrong
The E-Commerce Act contains an explicit date-bound exception. Products where performance begins on a specific date or where a consumer-requested reservation is involved — including tour reservations with fixed departure dates — are excluded from the blanket 7-day right when the cancellation terms were clearly disclosed before purchase. Applying a blanket withdrawal right to OTA bookings misrepresents the law and conflicts with valid disclosed cancellation policies.

### Correction
Check whether the date-bound or reservation exception applies before flagging withdrawal-right absence:
- Was the departure date fixed at time of booking?
- Were the cancellation/refund terms disclosed prior to purchase (PDP + checkout)?
- Does the product fall within a category the Act explicitly exempts?

If the exception applies, the operator's disclosed cancellation policy governs refund obligations — not the 7-day withdrawal right. Flag the finding only if disclosure of the cancellation terms was itself deficient.

---

## 3. AES-256 is law

### What goes wrong
The review states that "the law requires AES-256 encryption" or presents a specific algorithm, key length, or cipher mode as a statutory mandate, creating the impression that using a different algorithm is a legal violation.

### Why it's wrong
No Korean statute (PIPA, ISMS-P criteria, the Personal Information Security Measures Notice) mandates AES-256 by name. The law requires "appropriate technical and managerial measures" (적절한 기술적·관리적 보호조치) to protect personal data. AES-256 is an internal security baseline and industry best practice — not a black-letter legal requirement. Presenting it as statutory can cause engineers to treat a baseline control gap as a legal violation, distorting prioritization and escalation.

### Correction
Show both phrasings clearly:
- **Statutory requirement**: "Appropriate technical measures to prevent unauthorized access, loss, theft, falsification, or damage" (개인정보보호법 제29조 및 개인정보의 안전성 확보조치 기준).
- **Internal baseline**: "AES-256 is the organization's minimum standard for encryption at rest of sensitive personal data."

Flag a deviation from the internal baseline as a WARN or REVIEW, not as a BLOCK statutory violation, unless the deviation results in data that is effectively unprotected (e.g., no encryption at all for sensitive fields).

---

## 4. Every ranking is automated decisioning

### What goes wrong
The review flags any algorithmically influenced sort order — such as sort-by-popularity, recency, or review score — as PIPA Article 37-2 automated decision-making, requiring individual objection rights, explanations, and human review procedures.

### Why it's wrong
PIPA Article 37-2 applies to fully automated decisions that produce legal effects on an individual or similarly significant effects on that individual's rights and interests. A product sort order presented to a consumer browsing search results does not produce legal or similarly significant effects on the consumer as an individual. The article targets decisions like credit scoring, insurance eligibility, hiring, or access to services — not catalog ranking logic that affects which products appear first in a list.

### Correction
Before flagging a ranking or scoring feature under Article 37-2, confirm:
- Does the automated decision produce a legal effect on a specific individual (e.g., denial of service, contract terms)?
- Does it produce a similarly significant effect (e.g., profiling that meaningfully affects the individual's rights or access)?

Sort-by-popularity displayed to a browsing consumer does not meet this threshold. Record it as INFO at most, noting the scope gate is not satisfied. Reserve Article 37-2 findings for features that directly determine individual-level outcomes.

---

## 5. Cookie consent = opt-in

### What goes wrong
The review requires affirmative opt-in consent before setting any cookies or similar tracking technologies, citing "cookie consent law" and framing the absence of an opt-in prompt as a BLOCK violation.

### Why it's wrong
Korea's 정보통신망법 (Act on Promotion of Information and Communications Network Utilization and Information Protection) governs cookies and online behavioral tracking. Korean law requires notice and a refusal mechanism (거부 수단) — not affirmative prior consent. This is materially different from the EU's ePrivacy Directive / GDPR regime, which requires opt-in consent for non-essential cookies. Applying EU-style opt-in requirements to a Korean service overrepresents legal obligation and creates unnecessary UX friction.

### Correction
Distinguish the applicable regime:
- **Korean law (정보통신망법)**: notice of cookie use + accessible refusal mechanism. No affirmative opt-in required for most analytics and functional cookies.
- **EU/GDPR regime**: opt-in consent required for non-essential cookies.

For a Korea-primary service, flag the absence of a refusal mechanism as BLOCK, but flag the absence of a blanket opt-in prompt as INFO (noting it would be required if the service enters EU markets). Do not conflate the two regimes.

---

## 6. Shallow finding

### What goes wrong
The review lists findings like "this might violate PIPA," "possible privacy issue here," or "consider data protection" without specifying what file, what code path, what rule, or what the actual risk is. Findings are vague enough that an engineer cannot act on them.

### Why it's wrong
Shallow findings cannot be triaged, assigned, or fixed. They create noise that desensitizes engineers to real compliance signals. A finding that cannot be traced to a specific artifact, rule, and harm is not a finding — it is speculation. It also makes it impossible to close the finding once the issue is addressed, because the acceptance criterion was never stated.

### Correction
Every finding must include:
- **File and location**: specific file path and line range or component name.
- **Rule ID and statute article**: e.g., PIPA Article 15 (collection basis), ISMS-P control AC-3 (access control), E-Commerce Act Article 8 (pre-contract information).
- **What is wrong**: the specific observable condition that is non-compliant.
- **Why it matters**: the concrete harm or violation if left unaddressed.
- **What to change**: a specific, actionable remediation step.

If a concern is real but the reviewer cannot yet be specific about rule or artifact, classify it as REVIEW with a clear description of what needs to be investigated further. Never present vague concern as a concrete finding.
