# Korea OTA Compliance Rules for AI Coding Assistants
**Third-pass review after human legal comments and fresh official-source validation**

- **Applicable stack:** React/Next.js frontend, Spring Boot/Java backend, PostgreSQL
- **Effective as of:** 2026-04-08 (KST)
- **Purpose:** Machine-readable compliance guardrails for code generation, code review, product requirements, launch checklists, and security/privacy/compliance review
- **Important watchlist:** The amended E-Commerce Act provisions enacted on 2026-01-20 take effect on 2026-07-21, and amended EFTA provisions enacted on 2025-12-16 take effect on 2026-12-17. Re-check intermediary duties and payment-role outputs for launches on or after those dates.
- **Reviewed inputs:** revised skill draft, human-reviewed DOCX, and extracted legal comments including comments marked `Comments:` in Korean.

## How models must use this file
1. Classify the business role before applying any product-facing rule.
2. Classify the data before applying any privacy or security rule.
3. Apply only the rules whose `Applies when` condition is actually satisfied.
4. If legal applicability is unclear, output `REVIEW` or `ESCALATE` instead of inventing a requirement.
5. Treat future-effective watchlist items as watchlist only until their statutory effective date.
6. Do not treat regex matches, field names, domain suffixes, or UI labels as conclusive legal determinations.
7. Distinguish black-letter law from internal security baselines. Keep stricter internal baselines when they do not conflict with law, but do not describe them as if the statute itself mandates them.
8. When a prior internal control disappears only because the law is silent, restore it as an internal baseline rather than silently deleting it.

## Severity labels
- **BLOCK**: Do not approve or generate this implementation as-is.
- **REVIEW**: Escalate to legal, privacy, security, or finance before release.
- **WARN**: Strong risk signal; fix or obtain explicit sign-off.
- **INFO**: Internal best practice or guidance note.

## 0. Core classification rules
### CORE-001: Intermediary versus seller classification [BLOCK/REVIEW]
- **Authority / note:** Use before applying any E-Commerce, payment, refund, or tourism rule.
- **Applies when:** Any OTA page, checkout flow, receipt, customer support script, or policy text.
- **Rules:**
  - Classify the product role first: OTA intermediary only, OTA as actual seller or merchant of record, OTA as package-travel organizer, or a mixed model that changes by product.
  - Do not apply seller-only duties to the OTA when the OTA only intermediates a transaction for an independent supplier.
  - Do not describe the OTA as the seller or service provider unless the OTA is actually the contracting party for that product.
  - If the OTA role changes by product line, require the UI and receipt to show the correct role for the specific product.

### CORE-002: Third-party provision versus entrusted processing [REVIEW]
- **Authority / note:** PIPA classification guardrail.
- **Applies when:** Any supplier API, hotel/airline/activity partner feed, cloud/SaaS integration, CRM, analytics vendor, or customer support outsourcing flow.
- **Rules:**
  - Do not label every outbound data transfer as entrustment.
  - If the recipient independently uses the data to perform its own service for the user, treat the transfer as possible third-party provision and apply the stricter disclosure and consent analysis.
  - If the recipient processes data only on our documented instructions, treat it as entrusted processing or storage and reflect it in vendor controls and the privacy notice.
  - If the classification is ambiguous, escalate to privacy or legal review instead of auto-approving.

### CORE-003: Cross-border transfer classification [BLOCK/REVIEW]
- **Authority / note:** PIPA article 28-8 classification guardrail.
- **Applies when:** Any overseas hosting region, CDN, SaaS, supplier API, storage bucket, support tool, analytics tool, or embedded foreign widget that can receive personal data.
- **Rules:**
  - Determine overseas transfer status by recipient location, storage region, support-access region, and legal entity, not by top-level domain alone.
  - Do not assume that non-.kr means overseas transfer, and do not assume that .kr means domestic processing.
  - Classify the transfer type as overseas third-party provision, overseas entrustment, overseas storage, or possible direct overseas collection by another controller before deciding the legal basis and disclosure method.
  - If the foreign service collects the data directly from the user without the OTA first receiving or relaying them, do not automatically label the architecture as the OTA's own overseas transfer. Review the controller relationship, notice flow, and consent path separately.
  - If the model cannot identify the country, recipient, and transfer type, it must output REVIEW or ESCALATE instead of guessing.

### CORE-004: Payment-role classification [REVIEW]
- **Authority / note:** Payment and EFTA applicability guardrail.
- **Applies when:** Any card, wallet, BNPL, account transfer, or PG integration flow.
- **Rules:**
  - Determine whether the OTA is only integrated with a PG, is merchant of record, or performs any regulated payment role beyond ordinary merchant integration.
  - Do not assign financial-company or electronic-financial-business obligations to the OTA unless the architecture and contracts show that the OTA actually performs that regulated role.
  - Regardless of legal role, keep the internal rule that the OTA must not directly capture, store, or log raw card credentials outside an approved PCI-compliant PG flow.

### CORE-005: Credit Information Act applicability gate [REVIEW]
- **Authority / note:** Credit Information Act applicability guardrail.
- **Applies when:** Any audience segmentation, scoring, profiling, payment-history use, loyalty-risk model, or financing feature.
- **Rules:**
  - Do not apply the Credit Information Act automatically just because a dataset contains bookings, spend, refunds, or payment events.
  - First determine whether the data or feature actually involves credit information or personal credit information used to judge creditworthiness or transaction counterpart credit.
  - If that determination is unclear, default to PIPA and require legal review before invoking Credit Information Act-specific rules.

### CORE-006: Legal minimum versus internal baseline [INFO]
- **Authority / note:** Drafting and review guardrail.
- **Applies when:** Any rule in this file combines statutory requirements with company security standards.
- **Rules:**
  - Treat statutory duties, regulator guidance, and internal engineering baselines as separate layers.
  - Do not delete a stricter company control merely because the statute does not spell out the exact implementation detail.
  - If a control such as `AES-256`, `KMS/HSM`, `180-day key rotation`, short exact-location cache TTL, or `PG-hosted card entry only` is stricter than the law, keep it as an internal baseline and label it that way.
  - If a model cannot tell whether a statement is law or internal baseline, output `REVIEW` instead of rewriting the rule as a legal conclusion.

## 1. PIPA — Personal Information Protection Act
Primary authorities: PIPA articles 15, 17, 22, 22-2, 24, 24-2, 28-8, 34, 37-2, 38; Enforcement Decree articles 19, 21-2, 40, 44-2 to 44-4; PIPC 2024 guidance on mandatory-consent practices and automated decisions.

### PIPA-001: Resident registration number handling [BLOCK]
- **Authority / note:** PIPA article 24-2 and Enforcement Decree article 21-2.
- **Applies when:** Any entity, DTO, request, response, log, file export, backup, analytics job, or test data set that could contain a resident registration number.
- **Rules:**
  - Block collection, storage, or use of resident registration numbers unless a specific law or other narrow legal basis clearly permits it.
  - If electronic storage is legally required, require strong encryption at rest, strict access control, key management, and masking in all non-essential views.
  - **Internal baseline:** When lawful RRN processing is unavoidable, use AES-256 or an equivalent strong cipher, keep keys in KMS, HSM, or an equivalent separated key-management plane, and rotate keys at least every 180 days or under a stricter approved policy. Do not present this cadence as a statutory interval.
  - Do not use plaintext storage, MD5, SHA-1, or similar weak digest-based obfuscation as a substitute for encryption where later retrieval or lawful disclosure may be required.
  - Block resident registration numbers in logs, monitoring events, crash reports, analytics payloads, and developer fixtures.
  - Treat pattern matches as high-risk review signals even when the code does not explicitly name the field as residentRegistrationNumber.

### PIPA-002: Passport number and other unique identifier information [BLOCK]
- **Authority / note:** PIPA article 24; Enforcement Decree article 19; Personal Information Security Measures Standard article 7.
- **Applies when:** Any field or payload containing passport numbers, foreign registration numbers, driver's-license numbers, or other unique identifier information.
- **Rules:**
  - Do not treat a passport number as ordinary text, and do not incorrectly classify it as sensitive information under article 23.
  - When a passport number is unavoidable for travel fulfillment and no separate statutory basis exists, require a separate clear consent flow or another documented lawful basis that legal has approved.
  - Encrypt passport numbers at rest and in transit, mask them in UI and logs, and restrict access to staff who actually need them.
  - **Internal baseline:** For production systems, prefer AES-256 or an equivalent strong cipher, enforce access logging for privileged reads and exports, and keep cryptographic keys under managed rotation.
  - Block plaintext storage, broad internal sharing, or retention beyond the documented fulfillment or legal-retention need.

### PIPA-003: Sensitive information and biometric data [BLOCK]
- **Authority / note:** PIPA article 23; Personal Information Security Measures Standard article 7.
- **Applies when:** Biometric login, face or fingerprint processing, identity verification, fraud prevention, and any feature that could capture raw biometric templates.
- **Rules:**
  - Do not store or transmit raw biometric templates on the server unless the business need, legal basis, security design, and retention period have all been separately approved.
  - Where device biometric login is used, rely on OS or device secure hardware and server-side tokens, not raw biometric transfer.
  - Block storage of raw fingerprint, faceprint, voiceprint, or similar templates in application tables, object storage, analytics systems, or logs.

### PIPA-004: Collection legal basis and consent separation [BLOCK]
- **Authority / note:** PIPA articles 15 and 22; PIPC 2024 mandatory-consent guidance.
- **Applies when:** Any sign-up form, booking flow, identity collection flow, checkout page, privacy notice, or API that collects personal data from a user.
- **Rules:**
  - Do not generate or require a so-called mandatory consent checkbox for personal data that are strictly necessary to conclude or perform the service contract when another lawful basis applies.
  - For contract-necessary data, provide clear notice of mandatory items, purpose, retention, and the consequence of refusal instead of forcing a consent ceremony.
  - Separate optional consents from contract-necessary processing, especially for marketing, third-party provision, overseas transfer, profiling, optional traveler preferences, and any additional analytics use.
  - This rule does not remove separate-consent duties for sensitive information, unique identifier information, third-party provision, or overseas transfer when the statute still requires a separate consent or equivalent disclosure.
  - Block pre-checked boxes, bundled consent, or designs that make refusal practically impossible.

### PIPA-005: Third-party provision and entrusted processing [BLOCK/REVIEW]
- **Authority / note:** PIPA articles 17 and 26; privacy-portal guidance on third-party provision versus entrustment.
- **Applies when:** Supplier APIs, hotel/airline/activity fulfillment, outsourced customer service, cloud vendors, and SaaS tools.
- **Rules:**
  - If a hotel, airline, activity operator, or other supplier receives data as the actual service provider to the user, treat the disclosure as a possible third-party provision and apply the correct disclosure and consent analysis.
  - If a vendor processes data only on our instructions, treat it as entrusted processing or storage, document the task, and reflect the vendor in the privacy notice and contracts.
  - If separate consent is the basis for third-party provision, disclose the recipient, purpose, items provided, retention or use period, and the right to refuse in the required consent path.
  - Record the transfer basis, recipient, data items, time, and product context for any booking-related disclosure of personal data.
  - Do not rely on labels like vendor, partner, or middleware to avoid the underlying legal classification.

### PIPA-006: Cross-border transfers [BLOCK]
- **Authority / note:** PIPA article 28-8; PIPC official overseas-transfer guidance.
- **Applies when:** Any transfer, storage, or access of personal data outside Korea.
- **Rules:**
  - Before any overseas transfer, identify the country, recipient legal entity, transfer type, and lawful basis under article 28-8.
  - Document whether the basis is separate consent, law or treaty, contract-necessary overseas entrustment or storage, PIPC-certified safeguards, or an equivalence decision.
  - If separate consent is the basis, record the destination country, recipient, items transferred, purpose, retention or use period, and refusal consequence in the user-facing consent artifact or equivalent legally required notice.
  - For supplier fulfillment, review separately whether the disclosure is overseas third-party provision and therefore requires separate consent or another specific basis.
  - If the foreign service collects the data directly from the user without the OTA first receiving or relaying them, review separately whether this is the OTA's overseas transfer at all or a direct collection by another controller. Do not auto-classify the scenario.
  - Require TLS 1.2 or higher in transit, contractual controls, destination-country disclosure where required, and region-aware system inventory.
  - Block overseas transfer of personal data where the model cannot identify the transfer basis and the user-facing disclosure path.

### PIPA-007: Children under 14 [BLOCK]
- **Authority / note:** PIPA article 22-2 and article 38.
- **Applies when:** Any registration, booking, marketing, location, or profile feature that may be used by children under 14 where consent is the legal basis.
- **Rules:**
  - If consent is required and the user is under 14, require verifiable consent from the legal representative.
  - Collect only the minimum parent or guardian data needed to verify consent, and do not repurpose those data.
  - Use child-readable language for notices and in-product explanations when the feature is designed for or accessible to children.
  - Block consent-based optional features for under-14 users until the guardian-consent flow is complete.

### PIPA-008: Automated decision-making rights [REVIEW]
- **Authority / note:** PIPA article 37-2; Enforcement Decree articles 44-2 to 44-4; PIPC automated-decision guidance.
- **Applies when:** Dynamic pricing, recommendation ranking with significant effects, fraud decisions, eligibility filters, or other automated processing that can materially affect a user.
- **Rules:**
  - Do not hardcode a single required API path. Instead require a real user workflow to refuse, seek explanation, or request review of significant automated decisions.
  - Publish or otherwise disclose the use of automated decision-making, the main data categories used, and how the user can exercise the relevant rights.
  - Do not treat every ordinary personalization widget, hotel sort order, or recommendation surface as automatically within article 37-2. Prioritize review when a fully automated process significantly changes price, contract terms, booking eligibility, fraud holds, or prominent visibility with material effect on the user.
  - If the system can materially change price, availability, eligibility, or visibility without a user-rights workflow and internal handling process, route the feature to REVIEW before release.

### PIPA-009: Retention and destruction [WARN]
- **Authority / note:** PIPA article 21.
- **Applies when:** Any database, log store, warehouse, backup, support tool, marketing tool, or export pipeline containing personal data.
- **Rules:**
  - Destroy personal data without delay when the purpose is achieved unless another statute requires retention.
  - Maintain a dataset-level retention schedule tied to actual legal bases, not a blanket five-year assumption.
  - Block retain-forever defaults for booking personal data, identity documents, or support tickets that include sensitive traveler information.
  - When optional consent is withdrawn, stop the optional processing promptly and delete or de-identify the relevant data unless another law requires retention.

### PIPA-010: Pseudonymization for analytics and audience tooling [WARN]
- **Authority / note:** PIPA chapter on pseudonym information.
- **Applies when:** Analytics, experimentation, CRM audience building, demand modeling, and internal BI.
- **Rules:**
  - Use pseudonymous identifiers before analytics aggregation or cohort generation whenever direct identity is not necessary.
  - Store any mapping table separately with stronger access control, logging, and approval requirements.
  - Block audience outputs that directly expose name, email, phone, passport number, or other direct identifiers.
  - Allow re-identification only through an approved process with recorded purpose and authorized personnel.

### PIPA-011: Personal data breach notification and response [BLOCK]
- **Authority / note:** PIPA article 34; Enforcement Decree article 40.
- **Applies when:** Incident response playbooks, alerting systems, breach automation, crisis messaging, and user-notification tooling.
- **Rules:**
  - Use PIPA as the primary breach-notification authority. Do not rely on the older Network Act citation for this rule.
  - The incident runbook must support data-subject notification without delay and regulator or specialized-agency reporting when the statutory thresholds are met.
  - Build for a 72-hour regulator-reporting clock where reportable thresholds apply, including incidents involving at least 1,000 data subjects or any sensitive information or unique identifier information.
  - Block breach-response tooling that lacks user-notification templates, scope tracking, evidence preservation, and regulator-report escalation paths.

### PIPA-012: Voluntary privacy risk review for high-risk launches [INFO]
- **Authority / note:** Internal control. Note that the formal article 33 impact-assessment duty is directed at public institutions, not private OTA operators.
- **Applies when:** High-risk launches involving sensitive data, unique identifiers, children, AI decisioning, cross-border transfer, large-scale new collection, or new combinations of traveler and payment data.
- **Rules:**
  - Require pre-launch privacy review even where a formal public-sector article 33 impact assessment is not legally mandatory for a private OTA.
  - Do not describe article 33 as a private-sector legal obligation unless legal has confirmed a specific reason.
  - Preserve the review record, key data-flow diagram, and mitigation decisions as launch evidence for privacy, security, and compliance review.

## 2. LIA — Location Information Act
Primary authorities: Location Information Act articles 5, 9, 15, 16, 19, 21-2, 23, 25 and related subordinate rules. Use the current competent-authority references to the Broadcasting Media Communications Commission (방송미디어통신위원회).

### LIA-001: Consent and pre-prompt notice before collecting personal location information [BLOCK]
- **Authority / note:** Location Information Act article 15; internal UX control for the custom pre-prompt.
- **Applies when:** Any browser geolocation, app geolocation SDK, background location update, nearby-search, check-in, or map feature that processes a user's location.
- **Rules:**
  - Do not collect or use personal location information without the required consent.
  - In product UX, show an in-app or in-site pre-prompt notice before triggering the browser or OS permission dialog. The notice should name the collector, describe the purpose, explain whether the collection is one-time or continuous, and state retention or destruction timing.
  - Do not rely on the browser or OS permission prompt alone where the service-specific notice has not yet been surfaced.
  - Block silent or background location collection that starts before the user sees a service-specific notice and permission request.

### LIA-002: Business-model classification and filing requirement [REVIEW]
- **Authority / note:** Location Information Act articles 5 and 9.
- **Applies when:** Any feature that uses personal location information at service scale.
- **Rules:**
  - Do not assume that every geolocation feature creates the same permit, registration, or reporting obligation.
  - Determine whether the service is a personal-location-information business, a location-based service business, a personal-location-based service business, or outside those categories.
  - Before launch, confirm whether registration or prior reporting to the competent authority is required for the actual business model.
  - Where a filing is required, reference the current competent authority as the Broadcasting Media Communications Commission, not older regulator names.
  - Block product or engineering documentation that states no filing is needed without recording the underlying classification.

### LIA-003: Storage minimization and destruction of personal location information [BLOCK]
- **Authority / note:** Location Information Act articles 16 and 23.
- **Applies when:** Nearby search, map personalization, local experiences, geo-fenced promotions, or any feature using exact coordinates.
- **Rules:**
  - Immediately destroy personal location information once the collection, use, or provision purpose is achieved, except for the confirmation data that article 16(2) requires to be automatically recorded and preserved.
  - Do not substitute a blanket "never store location" slogan for the actual statutory model. The legal baseline is to minimize retention, preserve the required confirmation data, and destroy the remaining personal location information without delay.
  - Prefer in-memory processing, ephemeral session storage, or short-TTL caches. The default internal maximum for transient exact-location caches is 10 minutes unless approved otherwise.
  - Store coarse location or search-area abstractions instead of exact coordinates whenever exact coordinates are not necessary.
  - Block exact-location fields in permanent booking tables, analytics marts, long-term logs, or backup exports without documented approval.

### LIA-004: Confirmation-data recording and preservation [BLOCK/WARN]
- **Authority / note:** Location Information Act article 16(2).
- **Applies when:** Any service that collects, uses, or provides personal location information.
- **Rules:**
  - Ensure location collection, use, and provision confirmation data are automatically recorded and preserved in the location information system.
  - Keep the record content and retention period aligned with the law, the service terms, and the location privacy notice.
  - Internal staff-access logs are recommended and may be required by internal control, but do not confuse them with the article 16(2) confirmation-data duty.
  - Block implementations that process personal location information but cannot produce the statutory confirmation-data trail.

### LIA-005: Location terms and privacy-policy disclosure [WARN]
- **Authority / note:** Location Information Act articles 19 and 21-2.
- **Applies when:** Any location-based service that uses personal location information.
- **Rules:**
  - In the service terms and privacy notice, disclose the operator identity and contact, the location-based service content, user and guardian rights and how to exercise them, the legal basis and retention period for confirmation data, the processing purpose and retention period for personal location information, the destruction procedure and method, and third-party provision details where applicable.
  - Keep the location-specific notice consistent with the actual data flow, retention behavior, and vendor architecture.
  - Block launches where personal location information is collected but the location-specific terms or privacy section is missing or materially inconsistent with the implementation.

### LIA-006: Minor users and parental consent [BLOCK/REVIEW]
- **Authority / note:** Location Information Act article 25 and related privacy rules for children.
- **Applies when:** Any location feature available to under-14 users where consent is required.
- **Rules:**
  - If the service supports under-14 users and location collection is consent-based, require parental consent and age checks before collection starts.
  - If the service is not intended for under-14 users, block location collection after age-gate failure or route to a guardian flow.

## 3. ISMS-P / Internal Security Baseline
Primary authorities: ISMS-P controls and internal security baseline. This section mixes compliance expectations and internal engineering controls.

### ISMS-001: No hardcoded secrets [BLOCK]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Source code, CI config, container files, application YAML or properties, IaC, and test fixtures.
- **Rules:**
  - Block hardcoded API keys, tokens, database credentials, private keys, cloud access keys, signing secrets, or certificate material.
  - Reject .env files and other local secret files in commits unless the file contains only empty placeholders and is explicitly intended as an example.
  - Require secret retrieval through approved secret-management tooling or deployment-time injection.

### ISMS-002: Role-based access control and route protection [BLOCK]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Endpoints for personal data, admin actions, payment operations, internal analytics, and privileged tooling.
- **Rules:**
  - Require enforceable authorization for all sensitive routes, either through method-level annotations or documented centralized route-security configuration.
  - Do not require a specific annotation style if an equivalent centralized policy is clearly implemented and tested.
  - Block sensitive endpoints that have authentication but no role or policy-level authorization check.

### ISMS-003: Tamper-resistant audit logs [BLOCK]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Create, update, delete, refund, privilege-change, consent-change, and data-export actions.
- **Rules:**
  - Generate audit records containing actor, action, target entity or dataset, timestamp, and enough context to reconstruct the event.
  - Cover not only create, update, and delete actions but also consent changes, refunds, privileged reads of high-risk identifiers, bulk exports, downloads, and print-like data extractions where those events are operationally relevant.
  - For updates or deletes, store prior value, a controlled diff, or a cryptographic hash sufficient for forensic review.
  - Audit generation may be implemented through application events, interceptors, database triggers, or append-only log pipelines. Do not hardcode one implementation pattern.
  - Block sensitive data mutations or high-risk access events that leave no reliable audit trail.

### ISMS-004: Encryption in transit [BLOCK]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** External APIs, user traffic, admin panels, webhooks, supplier APIs, and data exports.
- **Rules:**
  - Require HTTPS for all external communications involving personal data or authenticated traffic.
  - Block cleartext external URLs in production configuration.
  - For internal service-to-service traffic, allow non-HTTPS only where the environment is segmented, justified, and separately approved.

### ISMS-005: Encryption at rest and high-risk field protection [WARN]
- **Authority / note:** ISMS-P control baseline plus privacy security standards.
- **Applies when:** Databases, caches, object storage, backups, exports, and support tools.
- **Rules:**
  - At minimum, require strong protection for unique identifier information, passport numbers, payment references, account-recovery tokens, authentication secrets, and similar high-risk fields.
  - For ordinary identifiers such as name, email, phone, and address, prefer approved encryption or equivalent compensating controls according to system risk and architecture.
  - Block plaintext storage of high-risk identifiers in production databases or exported flat files.

### ISMS-006: Session and cookie security [WARN]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Session cookies, admin sessions, traveler account sessions, and checkout-auth sessions.
- **Rules:**
  - Require Secure and HttpOnly for authenticated cookies.
  - Use SameSite settings that match the actual flow. Default to Lax or Strict. Document exceptions needed for cross-site identity flows.
  - Enable session-fixation protection and set tighter inactivity timeouts for sensitive flows such as account recovery, payment management, or admin actions.
  - Do not hardcode SameSite=Strict as the only acceptable value where federated login or third-party identity flows require a documented exception.

### ISMS-007: Vulnerability management [WARN/INFO]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Dependencies, base images, build pipelines, and runtime packages.
- **Rules:**
  - Run software-composition analysis in CI for Java, Node, and container dependencies.
  - Treat known high or critical vulnerabilities as a release blocker unless a documented exception with mitigating controls is approved.

### ISMS-008: Backup, recovery, and destructive changes [WARN/INFO]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Schema migrations, destructive batch jobs, and retention-purge tools.
- **Rules:**
  - Before dropping or rewriting tables containing personal data, confirm backup coverage and restore procedure ownership.
  - Block destructive migrations that lack rollback or recovery planning for regulated data sets.

### ISMS-009: Network segmentation [BLOCK/WARN]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Container, Kubernetes, VPC, firewall, and reverse-proxy configuration.
- **Rules:**
  - Block public exposure of database or cache ports without explicit approved architecture.
  - Flag database port mappings to 0.0.0.0 or equivalent public listeners as a severe risk.

### ISMS-010: Security event logging and abuse controls [WARN]
- **Authority / note:** ISMS-P control baseline.
- **Applies when:** Authentication, password reset, admin login, partner login, payment initiation, and account recovery.
- **Rules:**
  - Log failed login attempts, invalid tokens, permission denials, suspicious automation, and rate-limit events in structured form.
  - Apply rate limits, lockout, step-up authentication, or equivalent abuse controls to sensitive endpoints.
  - Do not hardcode one exact threshold such as five attempts for every system. Use a documented control suitable for the risk.

## 4. E-Commerce Consumer Protection Act
Primary authorities: E-Commerce Act article 10, article 17, article 18, article 20; Enforcement Rule article 7; FTC practice concerning platform and seller disclosures.

### ECOM-001: Cybermall operator disclosure [BLOCK]
- **Authority / note:** E-Commerce Act article 10 and Enforcement Rule article 7.
- **Applies when:** Website footer, app business-info page, landing pages, and other persistent or initial-screen operator disclosures.
- **Rules:**
  - Display the cybermall operator information required for the actual site operator, including trade name, representative name, business address including complaint-handling address, phone number or email, business registration number, and terms link.
  - Provide the FTC business-information disclosure link where required.
  - Do not cite article 13 as the basis for the OTA intermediary footer requirement.
  - If the operator also holds and uses a separate mail-order-sales registration number for its own seller activities, display that number where appropriate, but do not treat it as a universal intermediary-only requirement.

### ECOM-002: Intermediary notice and seller or supplier information [BLOCK]
- **Authority / note:** E-Commerce Act article 20; legal-team correction reflected.
- **Applies when:** Any OTA page or checkout flow where the OTA intermediates a supplier's accommodation, activity, transport, or other service.
- **Rules:**
  - Clearly disclose before contract formation that the OTA is an intermediary and not the actual seller or service provider, unless the OTA is in fact the contracting seller for that product.
  - A safe disclosure pattern is to state, in substance, that the platform intermediates the transaction and that the contracting seller or service provider is the individual supplier shown on the product page or checkout.
  - Expose the actual seller or supplier identity or a clear method to view it before payment or booking confirmation.
  - Block UI that names only the OTA while hiding the actual contracting supplier.

### ECOM-003: Pre-purchase information and total payable amount [BLOCK]
- **Authority / note:** E-Commerce Act product-information duties and OTA legal review comments.
- **Applies when:** Listing pages, detail pages, checkout pages, and pre-payment summaries.
- **Rules:**
  - Before payment, show the property or service name, supplier name or view method, address or service location, room or product type, occupancy or capacity, key amenities or included services, service date or check-in/check-out times, and cancellation summary.
  - Show the total payable amount including taxes, fees, and mandatory surcharges before payment authorization.
  - If the product page or rate card shows a partial price first, the UI must still surface the full payable amount before the user commits to payment.
  - Do not hide mandatory price components until after checkout, after login completion, after payment-method selection, or inside an expandable area that an ordinary user can easily miss.

### ECOM-004: Cancellation and refund disclosures for date-specific services [BLOCK]
- **Authority / note:** E-Commerce Act article 17 and related exceptions; legal-team correction reflected.
- **Applies when:** Accommodation, activities, tickets, transport, and other services provided on a specific date or during a specific period.
- **Rules:**
  - Do not make a blanket statement that consumers always have a seven-day withdrawal right.
  - Do not render a generic "7-day cancellation" badge, banner, or reusable checkout copy on date-specific accommodation or activity products unless legal has confirmed that the product actually qualifies.
  - For date-specific services, explain the actual cancellation deadline, penalty schedule, no-show treatment, and any non-refundable conditions before payment.
  - Where statutory withdrawal is restricted because of the service type and lawful prior disclosure, say so accurately and product-specifically.
  - Require affirmative acknowledgment of cancellation terms when penalties or non-refundable conditions apply.

### ECOM-005: Post-purchase confirmation and refund orchestration [WARN]
- **Authority / note:** E-Commerce Act articles 13 and 18.
- **Applies when:** Booking confirmations, receipts, cancellation flows, and refund processes.
- **Rules:**
  - After purchase, provide a structured confirmation showing the core booking details, supplier information, OTA role versus supplier role where relevant, amount paid, payment-method type, booking reference, and cancellation route.
  - If the user validly cancels, trigger refund or payment-reversal orchestration without delay.
  - Where the E-Commerce Act refund timing applies, coordinate the payment cancellation or refund within the statutory three-business-day timeline rather than leaving the user to chase both the OTA and the PG separately.


### ECOM-006: Future-law watchlist for intermediary duties [INFO]
- **Authority / note:** Amended E-Commerce Act provisions effective 2026-07-21.
- **Applies when:** Platform-role analysis, seller-identity disclosure flows, consumer-protection memo updates, and launches scheduled for or after 2026-07-21.
- **Rules:**
  - Do not apply the 2026-07-21 intermediary-amendment package as if it were already in force before that date.
  - For launches on or after 2026-07-21, re-check intermediary notice, seller-information exposure, review-handling, and liability-allocation outputs against the amended Act and sub-rules.

## 5. Payments / Electronic Financial Transactions Act
Primary authorities: EFTA article 9, article 22, Enforcement Decree article 12, plus internal payment-security controls. Some statutory duties apply directly to financial companies, electronic financial businesses, or PGs rather than to every OTA. Use the payment-role classification guardrail first.

### EFTA-001: No raw card-data handling outside PG-hosted controls [BLOCK]
- **Authority / note:** Internal payment-security baseline aligned to PCI and Korean payment practice.
- **Applies when:** Card-payment input, tokenization, web and app payment screens, and any server-side payment payload.
- **Rules:**
  - Do not capture, transmit, store, or log PAN, CVV or CVC, card PIN, track data, or raw expiry data in OTA-controlled frontend or backend fields.
  - Use the PG's PCI-compliant hosted fields, iframe, or tokenization SDK for any raw card entry.
  - Block React state, Redux stores, Java DTOs, logs, analytics events, and database rows from carrying raw card credentials.
  - Block ordinary HTML input fields for card numbers or CVV outside the approved PG component.

### EFTA-002: Payment-role disclosure and architecture truthfulness [WARN]
- **Authority / note:** Legal-team correction reflected; payment-role clarity requirement.
- **Applies when:** Privacy notices, payment pages, support scripts, architectural diagrams, and contracts with PGs.
- **Rules:**
  - If the PG actually processes payment credentials, user-facing materials and internal documentation should state that card data are processed through the PG and are not directly stored by the OTA, unless the architecture differs.
  - Do not claim that the OTA never processes payment data at all if the OTA still handles tokenized payment references, charge data, refund data, settlement data, or fraud signals.
  - If the OTA is merchant of record or assumes additional payment obligations, document that role explicitly.
  - Do not label ordinary OTA settlement assistance or intermediary-side collection of booking charges as registered electronic-payment-gateway activity unless legal or finance has actually confirmed that regulatory role.

### EFTA-003: Transaction records and retention schedule [WARN]
- **Authority / note:** EFTA article 22 and Enforcement Decree article 12; legal-team correction reflected.
- **Applies when:** Payment logs, refund logs, settlement records, and dispute records.
- **Rules:**
  - Log transaction ID, booking reference, amount, currency, payment-method type, PG or token reference, status, and timestamps.
  - Apply record-specific retention periods. Do not hardcode a blanket five-year period for every payment-related record.
  - Some electronic financial transaction records require five years, while some lower-value transaction records and approvals require one year. Other commercial, tax, or e-commerce records may have different periods.
  - Maintain the retention schedule in a policy or config source that engineering can reference.

### EFTA-004: Electronic-financial-accident and dispute response [WARN]
- **Authority / note:** EFTA article 9 and related user-protection practice.
- **Applies when:** Fraud complaints, duplicate charges, unauthorized payment claims, PG incidents, and settlement failures.
- **Rules:**
  - Maintain an incident runbook covering intake channel, evidence preservation, PG escalation, user communication, and responsibility allocation.
  - Even where a PG, issuer, financial company, or electronic-financial business is the primary legally responsible actor, the OTA must provide a clear user path, internal escalation, and status communication so that the issue is not stranded.
  - Block payment support flows that give the user no clear counterparty or no process to investigate or escalate an accident.

### EFTA-005: Fraud and abuse controls [WARN]
- **Authority / note:** Internal control aligned to payment-risk expectations.
- **Applies when:** Payment initiation, refund abuse, gift-card or wallet issuance, coupon stacking with payment, and suspicious repeat transactions.
- **Rules:**
  - Apply velocity limits, anomaly rules, suspicious-device or account checks, and manual review for clearly risky patterns.
  - Example controls may include repeat-transaction limits, unusual-amount review, rapid refund attempts, mismatched device or account signals, and partner- or region-specific fraud rules.
  - Thresholds should be risk-based and documented. Do not treat any single sample threshold as a legal rule.

### EFTA-006: Refund execution path [WARN]
- **Authority / note:** Internal control plus E-Commerce Act coordination.
- **Applies when:** Cancellation, partial refund, charge adjustment, and settlement reversal flows.
- **Rules:**
  - If the product allows cancellation or refund, the system must have a working path to reverse or refund the payment and to notify the user.
  - Block flows that accept cancellation at the product layer but have no actual payment-reversal orchestration.

### EFTA-007: Future-law watchlist for payment-role classification [INFO]
- **Authority / note:** Amended EFTA provisions effective 2026-12-17.
- **Applies when:** Architecture decisions, regulatory memoranda, or model outputs that classify OTA settlement flows.
- **Rules:**
  - Do not apply the 2026-12-17 exclusion for incidental settlement in the course of e-commerce intermediation before its effective date.
  - For launches on or after 2026-12-17, re-check whether a given OTA settlement flow still falls inside the statutory definition of electronic payment gateway activity.

## 6. Network Act / Digital Marketing Rules
Primary authorities: Network Act article 50 and subordinate rules, plus current KISA guidance. This section focuses on advertising messages and tracking practices. Data-breach notification belongs primarily under PIPA in this document.

### NET-001: Marketing opt-in and consent separation [BLOCK]
- **Authority / note:** Network Act article 50 and current KISA guidance.
- **Applies when:** Email, SMS, LMS, push, Kakao, in-app advertising messages, and channel opt-in flows.
- **Rules:**
  - Require explicit prior opt-in for advertising messages unless a specific statutory exception clearly applies.
  - Keep marketing consent separate from service terms, booking consent, and mandatory operational notices.
  - The consent UI must clearly identify itself as advertising-message consent for the relevant channel. Block vague phrases such as "혜택 알림" or "정보제공" when they obscure that the user is agreeing to receive advertising.
  - Block pre-checked marketing boxes, default-on marketing flags, and bundling that makes service use conditional on advertising consent.

### NET-002: Advertising label, sender disclosure, unsubscribe, and nighttime consent [BLOCK]
- **Authority / note:** Network Act article 50.
- **Applies when:** Any outbound advertising message job or template.
- **Rules:**
  - Where required, clearly mark the message as advertising, including the `[광고]` label or equivalent channel-specific marker, identify the sender and contact point, and provide an easy no-cost unsubscribe or consent-withdrawal method.
  - For advertising messages sent between 21:00 and 08:00, obtain separate prior consent unless the channel is exempt by decree.
  - Block campaign schedulers that can send nighttime ads without checking the dedicated nighttime-consent flag.
  - For app push or similar channels, do not require unnecessary login or unduly complex navigation merely to opt out of advertising.

### NET-003: Consent, withdrawal, processing-result notice, and two-year reconfirmation [WARN]
- **Authority / note:** Network Act article 50 and Enforcement Decree articles 62 and 62-3.
- **Applies when:** CRM, consent service, messaging scheduler, and suppression lists.
- **Rules:**
  - Store the channel, timestamp, source, and version for both consent and withdrawal.
  - Apply suppression promptly after withdrawal and block sends when consent evidence is missing.
  - Where required, notify the user that consent or withdrawal has been processed.
  - Reconfirm ongoing advertising consent at least every two years with the information required by the decree.

### NET-004: Operational notices versus advertising classification [REVIEW]
- **Authority / note:** Current KISA guidance.
- **Applies when:** Coupon, mileage, points, wallet-balance, benefit-expiry, membership-benefit, and similar outbound messages.
- **Rules:**
  - Do not automatically classify coupon, mileage, points, or benefit messages as non-advertising operational notices.
  - If the benefit was granted unilaterally or the message promotes future transactions, review it under the advertising-message rules and require consent unless a clear statutory exception applies.
  - Keep pure booking-status, payment-status, and legally required service notices in separately managed non-marketing templates.

### NET-005: Cookies, SDKs, and behavioral advertising [WARN]
- **Authority / note:** PIPC privacy-policy practice for cookies and PIPC enforcement on targeted advertising without valid consent.
- **Applies when:** Website cookies, app SDKs, analytics tags, ad pixels, personalization tools, and consent banners.
- **Rules:**
  - Do not assume that Korean law requires a blanket opt-in banner for every cookie.
  - At minimum, disclose the use, purpose, and refusal method for cookies or similar trackers.
  - If tracker data are combined with personal data or used for personalized or behavioral advertising that requires consent, obtain valid prior consent before loading or activating those trackers.
  - **Internal baseline:** Default-load only essential trackers until the system determines whether consent is required and, if so, whether it has been obtained.
  - Do not tell the user that Korean law universally requires blanket opt-in for every cookie category when the actual legal analysis is notice plus refusal by default and consent only in the cases that trigger personal-data or targeted-advertising rules.

## 7. Tourism Promotion Act
Primary authorities: Tourism Promotion Act and Enforcement Rule article 21 on advertisements for package travel. Apply this section only where the OTA or an affiliate is actually advertising or selling travel-agency or package-travel products.

### TOUR-001: Travel-business disclosure for package-travel products [WARN]
- **Authority / note:** Tourism Promotion Act Enforcement Rule article 21.
- **Applies when:** Package tours, organized travel products, and advertisements by a registered travel business.
- **Rules:**
  - For package-travel advertisements, display the travel-business registration number, business name, location, registration authority where required by the format, and other core items required for the relevant advertisement format.
  - Do not require or claim a KATA membership number unless the company is actually a member and chooses or needs to disclose that fact.
  - Do not invent tourism-law disclosure duties for accommodation-brokerage pages that are not package-travel advertisements.

### TOUR-002: Package-travel disclosures [WARN]
- **Authority / note:** Tourism-law package-travel disclosure practice.
- **Applies when:** Package or organized tour products.
- **Rules:**
  - Before purchase, show itinerary or major destinations, travel dates, major included services such as transport and lodging, total price, and cancellation or force-majeure conditions.
  - If minimum traveler counts, insurance, or operator-specific conditions apply, disclose them before purchase.

## 8. Credit Information Act
Primary authorities: Credit Information Act article 2 and related subordinate rules. This section should be used only after the applicability gate is satisfied.

### CREDIT-001: Applicability gate [REVIEW]
- **Authority / note:** Credit Information Act article 2.
- **Applies when:** Audience Builder, fraud scoring, financing features, installment products, payment-behavior analysis, or loyalty-risk segmentation.
- **Rules:**
  - Do not assume this Act applies merely because a dataset contains payment amount, refund history, booking frequency, or ordinary OTA CRM attributes.
  - First determine whether the data are being used as credit information or personal credit information to assess credit or transaction counterpart creditworthiness.
  - Ordinary OTA segmentation or personalization based on booking and payment history defaults to PIPA unless legal has specifically concluded that the feature falls within the Credit Information Act.
  - Even when this Act does not apply, keep the PIPA pseudonymization, access-control, logging, and explanation controls that remain relevant to profiling or analytics.
  - If the answer is unclear, do not cite this Act as the governing rule without legal review.

### CREDIT-002: If the Credit Information Act does apply, apply sector-specific profiling controls [BLOCK/REVIEW]
- **Authority / note:** Credit Information Act and subordinate pseudonymization or profiling controls.
- **Applies when:** Only after CREDIT-001 is satisfied.
- **Rules:**
  - If personal credit information is used for scoring, profiling, or segmentation, apply the sector-specific consent, access-control, logging, and pseudonymization rules that legal confirms for the feature.
  - Block direct joins between raw identity tables and scoring or cohort outputs unless an approved controlled process exists.

## Appendix A. Retention matrix
| Source | Record type | Retention | Model instruction |
|---|---|---:|---|
| PIPA | Personal data in general | Destroy when purpose is achieved unless another law requires retention | Do not invent a universal five-year rule. |
| E-Commerce Act | Advertising and display records | 6 months | Keep the specific evidence needed for consumer-protection disputes. |
| E-Commerce Act | Contract, offer withdrawal, and transaction records | 5 years | Use for bookings and cancellation disputes where the Act applies. |
| E-Commerce Act | Payment and supply records | 5 years | Coordinate with PG and settlement records. |
| E-Commerce Act | Consumer complaints and dispute handling | 3 years | Preserve support and dispute-resolution evidence. |
| EFTA | Device access logs, application or condition changes, transactions over KRW 10,000, and other article 12(1) items | 5 years | Do not collapse these into the privacy-retention schedule. |
| EFTA | Certain lower-value transactions and approval records | 1 year | Check the exact record class before setting retention. |
| Commercial/Tax | Accounting or tax evidence | Check finance or tax schedule | Some finance records may have longer retention than privacy-driven data. |

## Appendix B. Detection heuristics
These are **heuristics only**. They are review aids, not legal conclusions.

- **High-risk resident-like identifier**: `/\b\d{6}-?\d{7}\b/  # heuristic only, review context before concluding it is an RRN or foreign registration number`
- **Hardcoded secret**: `/(password|secret|api[_-]?key|token|private[_-]?key)\s*[=:]\s*[\"'`].{8,}/i`
- **High-risk field names**: `/(residentRegistration|rrn|passport|foreignerRegistration|visaNumber|cardNumber|cvv|cvc|accountNumber)/i`
- **External non-TLS URL**: `/http:\/\/(?!localhost|127\.0\.0\.1|0\.0\.0\.0)/`
- **Geolocation API usage**: `/navigator\.geolocation\.(getCurrentPosition|watchPosition)/`
- **Pre-checked checkbox**: `/(defaultChecked|checked)\s*=\s*\{?\s*true/`

## Appendix C. Changes applied from legal review and third validation pass
- Removed the blanket assumption that all personal-data collection requires consent. Contract-necessary collection now uses a lawful-basis analysis plus notice instead of forced mandatory consent.
- Rewrote the passport rule so that passport numbers are handled as unique identifier information, not as sensitive information.
- Clarified that direct collection by an overseas recipient is not automatically the same as the OTA's own overseas transfer, so the model must review controller relationship and architecture before deciding the rule.
- Narrowed automated-decision controls so the model does not over-apply article 37-2 to every ordinary recommendation widget.
- Re-cited OTA footer and disclosure rules to the cybermall operator and intermediary provisions, and added the intermediary notice that the OTA is not the seller unless stated otherwise.
- Removed the blanket seven-day withdrawal statement for accommodations and other date-specific services, replaced it with product-specific cancellation and refund disclosure rules, and added a false-positive guard against generic "7-day cancellation" badges.
- Corrected the Location Information Act block to reflect the current competent authority, the article 16 confirmation-data recording duty, the article 21-2 location privacy-policy duty, and the article 23 destruction rule.
- Expanded the marketing block to cover clear advertising-consent wording, no-cost unsubscribe, nighttime-send consent, processing-result notice, and two-year reconfirmation.
- Added future-effective watchlists for the E-Commerce Act amendments taking effect on 2026-07-21 and the EFTA amendments taking effect on 2026-12-17.
- Added applicability gates for payment-role analysis, location-business filing analysis, and Credit Information Act analysis so that the model does not over-apply the wrong law.
- Restored previously approved engineering controls, such as strong field-level encryption, managed key custody, and rotation cadence for high-risk identifiers, as internal baselines rather than presenting them as black-letter statutory text.

## Appendix D. Primary official-source checklist
- Personal Information Protection Commission, 2024 guidance on improving mandatory-consent practices.
- Personal Information Protection Commission, official overseas-transfer guidance under article 28-8.
- National Law Information Center, PIPA article 15, article 24, article 24-2, article 33, article 34, article 37-2, and Enforcement Decree article 19 and article 40.
- National Law Information Center, Location Information Act article 15, article 16, article 19, article 21-2, article 23, and article 25, plus current competent-authority references.
- National Law Information Center, E-Commerce Act article 10, article 17, article 18, article 20, and Enforcement Rule article 7.
- National Law Information Center, Network Act article 50 and Enforcement Decree article 62 and article 62-3.
- KISA 2026 guidance, including the 7th revised spam-compliance guide on clear consent wording and opt-out handling.
- National Law Information Center, Electronic Financial Transactions Act article 9, article 22, and Enforcement Decree article 12.
- National Law Information Center, Tourism Promotion Act Enforcement Rule article 21.
- National Law Information Center, Credit Information Act article 2 and Enforcement Decree article 2.

\newpage

## Appendix E. Future-effective watchlist
| Source | Effective date | Watch item | Model instruction |
|---|---:|---|---|
| E-Commerce Act amendment package | 2026-07-21 | Updated intermediary and platform duties | Treat as watchlist until the effective date, then refresh intermediary outputs. |
| EFTA amendment package | 2026-12-17 | Incidental settlement in the course of e-commerce intermediation excluded from PG scope | Do not cite as current law before the effective date; re-check payment-role outputs for launches on or after that date. |


## Appendix F. Legal-comment disposition log
| Comment theme from reviewed DOCX | Resolution in this version |
|---|---|
| Intermediary notice was missing | `ECOM-002` now requires a clear pre-contract notice that the OTA is an intermediary and not the seller unless the OTA is actually the contracting party. |
| Article 13 was the wrong basis for the OTA footer | `ECOM-001` now uses E-Commerce Act article 10 and the related rule for cybermall operator disclosure, and keeps seller exposure under article 20. |
| Blanket “7-day withdrawal” wording was too broad for accommodation | `ECOM-004` now blocks generic seven-day withdrawal claims for date-specific lodging or activity products and requires product-specific cancellation terms. |
| Seller information and full price display needed strengthening | `ECOM-003` now requires supplier visibility and the total payable amount before payment, and blocks hidden mandatory fees. |
| Payment responsibility and user-protection flow were missing | `EFTA-004` now requires an accident and dispute runbook aligned with article 9 responsibility structure. |
| PG outsourcing and raw-card handling needed clearer language | `EFTA-001` and `EFTA-002` now make PG-hosted card entry the baseline and require truthful documentation that raw card credentials are handled through the PG rather than stored by the OTA. |
| Record-retention periods were over-generalized | `EFTA-003` and Appendix A now separate EFTA, E-Commerce, commercial, tax, and PIPA retention logic instead of using one blanket period. |
| Advertising label, unsubscribe, and nighttime-send rules were incomplete | `NET-002` and `NET-003` now require `[광고]` or equivalent labeling, no-cost unsubscribe, separate nighttime consent, and two-year reconfirmation handling. |
| Cookie rules overstated universal opt-in | `NET-005` now uses notice plus refusal by default, and requires prior consent only where tracking is tied to personal-data or targeted-advertising consent analysis. |
| Credit Information Act was being applied too broadly | `CORE-005` and `CREDIT-001` now require an applicability gate before invoking Credit Information Act duties. |
| Earlier internal encryption details disappeared in later drafts | `CORE-006`, `PIPA-001`, and `PIPA-002` now preserve those details as internal baselines instead of misstating them as statutes. |

## Appendix G. Internal baselines preserved from earlier drafts
These are **company security baselines**, not claims that the statutes themselves prescribe each exact implementation detail.

- **High-risk identifier encryption baseline:** For RRN, passport number, and comparable high-risk identifiers, prefer AES-256 or an equivalent strong cipher, managed key custody through KMS, HSM, or an equivalent control plane, and rotation at least every 180 days or under a stricter approved key policy.
- **Location minimization baseline:** Default to ephemeral exact-location handling and a 10-minute maximum transient cache unless a documented exception is approved.
- **Payment baseline:** Raw card entry must stay inside the approved PG-hosted field, iframe, or SDK path. OTA-controlled UI and backend must not carry PAN, CVV, or equivalent secrets.
- **Tracker baseline:** Default-load only essential trackers until consent analysis is complete and any required consent has been obtained.
- **Audit baseline:** Privileged reads of high-risk identifiers, exports, downloads, and consent changes should be logged in addition to state-changing mutations.

