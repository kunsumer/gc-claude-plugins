<!-- Last reviewed: 2026-04-09 | Legal effective dates: N/A (calibration guidance) -->

# Severity Calibration Guide

This guide anchors severity assignments with concrete examples. Use it to calibrate findings before presenting them. Consistent severity signals help teams prioritize correctly and avoid both under-reaction to real risk and fatigue from over-escalation.

## Severity definitions

| Level | Meaning |
|---|---|
| **BLOCK** | Must be fixed before approval, merge, or launch. Creates direct legal exposure, enables harm, or violates a non-negotiable control. |
| **REVIEW** | Escalate to legal, privacy, security, finance, or tax before proceeding. Situation is ambiguous or context-dependent enough that a reviewer cannot make the call alone. |
| **WARN** | Strong risk signal. Fix before launch or obtain explicit documented sign-off from the appropriate owner. |
| **INFO** | Guidance or best-practice note. No immediate risk, but worth addressing over time. |

---

## BLOCK — examples

These conditions require a fix before the change can be approved or merged.

1. **Raw resident registration number (RRN / 주민등록번호) written to application logs.**
   Direct statutory violation. PIPA and the RRN-specific provisions of the Personal Information Protection Act prohibit collection without explicit legal basis; logging raw RRNs with no access control creates uncontrolled exposure. Fix: mask before logging; audit collection basis.

2. **Passport or travel-document data collected with no documented consent basis.**
   Passport data is sensitive personal information requiring explicit consent (명시적 동의) or a clear statutory collection basis under PIPA Article 15. A collection flow with no basis documented in the privacy notice and no consent capture is a direct violation. Fix: document basis, add consent step, or remove collection if not required.

3. **Personal data transferred to an overseas system with no transfer basis established.**
   Cross-border personal data transfer without consent, adequacy finding, standard contractual clauses, or another Article 28-8 basis is a PIPA violation. Fix: establish and document a transfer basis before the integration goes live; update the transfer register.

4. **Privileged admin action (e.g., force-cancel booking, override refund, modify partner status) executed with no audit trail.**
   ISMS-P requires audit logging for privileged operations. The absence of an audit trail for actions that affect partner standing, financial state, or traveler data means the action cannot be reviewed, disputed, or investigated after the fact. Fix: add audit log capture before merge.

5. **Marketing consent and service-agreement consent bundled into a single checkbox.**
   정보통신망법 requires that consent for optional marketing communications be obtained separately from consent required for service provision. Bundling makes both consents legally suspect. Fix: separate into distinct consent items before launch.

---

## REVIEW — examples

These conditions require human judgment from the appropriate domain expert before a decision can be made.

1. **Ambiguous controller / processor relationship with a new third-party integration.**
   The integration passes partner contact data to an external notification service. It is unclear whether the service acts solely on the OTA's instructions (entrustment) or processes data for its own purposes (third-party provision). The correct legal framing determines whether an entrustment contract or a third-party provision disclosure is required. Escalate to legal for relationship classification before finalizing the integration.

2. **Location data collection where LIA (위치정보법) filing status is unclear.**
   The feature collects approximate user location to surface nearby tours. LIA requires business registration as a location information business or location-based service provider depending on collection method and storage. Whether the specific implementation triggers the filing requirement is a legal judgment call. Escalate to legal to confirm filing status before collecting location data.

3. **Partner bank account and settlement data classification shared with the Finance Platform.**
   Settlement data ownership sits at the boundary between the platform and the shared Finance Platform. Whether the platform is acting as a controller, a processor, or both for this data when passing it downstream is not established in current boundary documentation. Escalate to legal and Finance Platform owners to document the data relationship and any resulting contract obligations.

4. **Newly effective statute with provisions that may apply to the current feature.**
   A regulatory amendment has come into effect since the last compliance review of this feature area. The reviewer has identified the potentially applicable provision but cannot determine without legal interpretation whether the existing implementation satisfies it. Escalate to legal for a gap assessment before the next release touching this area.

---

## WARN — examples

These conditions are strong risk signals that should be fixed before launch or explicitly signed off.

1. **Retention period for collected traveler data is not documented.**
   PIPA requires personal data to be destroyed when the purpose of collection is fulfilled or after the legally required retention period. The absence of a documented retention schedule does not create immediate harm but leaves the organization unable to demonstrate compliance in an audit. Fix: document retention period and destruction trigger for each data class.

2. **Consent UI does not clearly separate required consent from optional consent.**
   The checkout flow presents multiple consent items in a way that makes it visually non-obvious which are required for service and which are optional (e.g., marketing, analytics). While each item has its own checkbox, the visual hierarchy and copy are ambiguous. This is unlikely to invalidate consent in isolation but creates audit and user-complaint risk. Fix: revise UI so required vs. optional consent items are unambiguously distinguished.

3. **Hashed email address treated as anonymized data in analytics pipeline.**
   The analytics export uses SHA-256 hashed email addresses. Hashed emails are pseudonymous, not anonymous — they can be re-identified by anyone with access to the email list (including the analytics vendor). PIPA obligations apply to pseudonymous data. Fix: either use a privacy-preserving alternative (e.g., differential privacy, aggregation) or apply pseudonymous-data controls to the export.

4. **API error response includes internal field names that reveal data model structure.**
   A 400 error response returns field names that correspond to internal database column names or PII field names (e.g., `residentRegistrationNumber`, `bankAccountNumber`). This does not directly expose sensitive values but provides reconnaissance information. Fix: normalize error responses to use public field names and generic error codes at the API boundary.

---

## INFO — examples

These are best-practice notes. No immediate risk, but worth addressing in a normal improvement cycle.

1. **Encryption key rotation schedule is not documented for data at rest.**
   AES-256 encryption is in place for sensitive fields, but there is no documented key rotation schedule or procedure. This does not create a present vulnerability but is a gap in operational security posture documentation. Recommend: document rotation schedule and procedure in the security runbook.

2. **Audit log entries do not include a correlation ID linking them to the originating request.**
   Audit entries capture the action, actor, and timestamp but do not include a request or session correlation ID. This makes it harder to reconstruct a full request chain during an incident investigation. Recommend: add correlation ID to audit log schema in the next logging improvement cycle.

3. **Data flow diagram for the booking flow does not reflect the current integration architecture.**
   The existing data flow diagram was last updated before the Kafka consumer was added. The diagram is used in compliance reviews and onboarding. It does not constitute a compliance gap by itself but increases the risk of incorrect compliance assessments. Recommend: update the diagram to reflect current state before the next compliance review.

4. **Code comment for sensitive field does not document purpose or owner.**
   A field storing traveler nationality is present in the domain model with no comment indicating why it is collected, who owns it, or what its retention rule is. This is a documentation gap, not a functional issue. Recommend: add a structured comment (purpose, owner, retention class) to all sensitive personal data fields as a code hygiene standard.

---

## Decision tree for borderline cases

Use this tree when you are uncertain whether to escalate a severity or lower it.

1. **Does the condition directly violate a statute or a non-negotiable internal control?**
   Yes → BLOCK. Stop.

2. **Could the condition result in measurable harm to a data subject, partner, or traveler if left unaddressed through this release?**
   Yes → BLOCK or WARN depending on imminence. Stop.

3. **Does the condition require a judgment call from legal, privacy, security, finance, or tax that the reviewer cannot make alone?**
   Yes → REVIEW. Stop.

4. **Is the condition a known risk pattern that materially increases the probability of a future violation or incident?**
   Yes → WARN. Stop.

5. **Is the condition a best-practice gap with no near-term harm pathway?**
   Yes → INFO. Stop.

**Still unsure → REVIEW.** When in doubt, escalate rather than suppress. A REVIEW finding that turns out to be low-risk is safer than a suppressed BLOCK.
