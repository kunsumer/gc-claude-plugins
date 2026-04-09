# Review Checklists by Type

Quick-reference checklists derived from `rules.md`. Each item maps to a rule ID.
Use these as a scan checklist — refer to `rules.md` for full rule text.

---

## PRD Review Checklist

### Classification (do first)
- [ ] Business role identified per product line (CORE-001)
- [ ] Data categories mapped: what personal data is collected, from whom, why (CORE-002)
- [ ] Payment role documented: PG-only vs MOR vs regulated (CORE-004)
- [ ] Credit Information Act applicability assessed if scoring/profiling mentioned (CORE-005)

### Privacy & Consent
- [ ] Collection legal basis stated per data item — contract-necessary vs consent (PIPA-004)
- [ ] No forced consent for contract-necessary data (PIPA-004)
- [ ] Marketing consent separated from service consent (NET-001)
- [ ] Optional consents (marketing, profiling, 3rd-party, overseas) listed separately (PIPA-004)
- [ ] Retention periods specified per dataset, not blanket 5-year (PIPA-009)
- [ ] Pseudonymization plan for analytics/audience if applicable (PIPA-010)

### Third-party & Cross-border
- [ ] Each supplier/vendor classified as 3rd-party provision or entrustment (PIPA-005, CORE-002)
- [ ] Overseas transfers identified with country, recipient, basis, disclosure (PIPA-006, CORE-003)
- [ ] Direct foreign collection by another controller distinguished from OTA transfer (CORE-003)

### E-Commerce & Consumer
- [ ] OTA intermediary role disclosed pre-contract (ECOM-002)
- [ ] Supplier identity exposed before payment (ECOM-003)
- [ ] Total payable amount shown before payment authorization (ECOM-003)
- [ ] Cancellation/refund terms product-specific, no generic 7-day badge (ECOM-004)
- [ ] Post-purchase confirmation requirements addressed (ECOM-005)

### Location (if applicable)
- [ ] Location collection purpose and consent model specified (LIA-001)
- [ ] Business classification and filing requirement assessed (LIA-002)
- [ ] Storage minimization approach documented (LIA-003)

### Automated Decisions (if applicable)
- [ ] Scope of article 37-2 assessed — not every sort/filter qualifies (PIPA-008)
- [ ] User-rights workflow for significant automated decisions planned (PIPA-008)

### Breach & Incident
- [ ] Breach notification plan referenced (PIPA-011)

### Tourism (if package travel)
- [ ] Travel-business registration and disclosures addressed (TOUR-001, TOUR-002)

### Watchlist
- [ ] E-Commerce Act amendments (2026-07-21) impact assessed if launch ≥ that date (ECOM-006)
- [ ] EFTA amendments (2026-12-17) impact assessed if launch ≥ that date (EFTA-007)

---

## UX Review Checklist

### Consent Flows
- [ ] Contract-necessary fields use notice, not forced consent checkbox (PIPA-004)
- [ ] Optional consents have separate, unchecked toggles (PIPA-004)
- [ ] No pre-checked boxes for marketing, profiling, 3rd-party, overseas (PIPA-004, NET-001)
- [ ] Marketing consent clearly says "advertising" — no vague "혜택 알림" (NET-001)
- [ ] Nighttime ad consent (21:00–08:00) is separate toggle if applicable (NET-002)

### Disclosures
- [ ] OTA intermediary notice visible pre-contract (ECOM-002)
- [ ] Supplier identity visible before payment (ECOM-003)
- [ ] Total price including all fees shown before payment button (ECOM-003)
- [ ] Cancellation terms are product-specific, no generic 7-day claim (ECOM-004)
- [ ] Cybermall operator footer has required fields (ECOM-001)

### Location UX
- [ ] Pre-prompt notice before browser/OS permission dialog (LIA-001)
- [ ] Pre-prompt states: collector, purpose, one-time vs continuous, retention (LIA-001)

### Cookie/Tracker Banner
- [ ] Essential-only trackers load by default (NET-005 internal baseline)
- [ ] No blanket opt-in banner claim — use notice + refusal, consent where required (NET-005)

### Children
- [ ] Under-14 guardian consent flow if service is accessible to children (PIPA-007, LIA-006)
- [ ] Child-readable language for notices (PIPA-007)

### Automated Decisions
- [ ] Explanation/refusal/review workflow reachable for significant automated decisions (PIPA-008)

### Overseas Transfer Notice
- [ ] Consent screen shows country, recipient, items, purpose, retention, refusal consequence (PIPA-006)

### Advertising Messages
- [ ] `[광고]` label or equivalent on ad messages (NET-002)
- [ ] Easy no-cost unsubscribe (NET-002)
- [ ] Opt-out doesn't require login or complex navigation (NET-002)

---

## Solution Architecture Review Checklist

### Classification (do first)
- [ ] Business role per product line documented (CORE-001)
- [ ] Data flow diagram shows personal data categories at each hop (CORE-002)
- [ ] Cross-border transfers mapped: country, recipient entity, transfer type (CORE-003)
- [ ] Payment role documented (CORE-004)

### Data Protection
- [ ] RRN handling: blocked unless specific law permits; if stored, AES-256 + KMS + rotation (PIPA-001)
- [ ] Passport numbers: encrypted at rest, masked in UI/logs, access-logged (PIPA-002)
- [ ] No biometric templates on server unless separately approved (PIPA-003)
- [ ] High-risk identifiers not in logs, crash reports, analytics, test fixtures (PIPA-001)

### Third-party & Cross-border Architecture
- [ ] Each vendor/supplier classified as 3rd-party provision vs entrustment (PIPA-005)
- [ ] Overseas transfer basis documented per integration (PIPA-006)
- [ ] TLS 1.2+ for all overseas transit (PIPA-006)
- [ ] Region-aware system inventory maintained (PIPA-006)

### Security Baselines (ISMS)
- [ ] No hardcoded secrets in code/config/IaC (ISMS-001)
- [ ] RBAC on all sensitive routes (ISMS-002)
- [ ] Tamper-resistant audit logs for mutations, consent changes, privileged reads (ISMS-003)
- [ ] HTTPS for all external comms (ISMS-004)
- [ ] Encryption at rest for high-risk fields (ISMS-005)
- [ ] Secure session cookies: Secure, HttpOnly, SameSite (ISMS-006)
- [ ] SCA in CI pipeline (ISMS-007)
- [ ] Backup/recovery for destructive migrations (ISMS-008)
- [ ] No public DB/cache port exposure (ISMS-009)
- [ ] Security event logging + rate limiting on auth endpoints (ISMS-010)

### Payments
- [ ] Raw card data stays inside PG-hosted iframe/SDK (EFTA-001)
- [ ] No PAN/CVV in OTA frontend state, backend DTOs, logs, DB (EFTA-001)
- [ ] Payment role truthfully documented in privacy notice and architecture (EFTA-002)
- [ ] Transaction log schema covers required fields (EFTA-003)
- [ ] Record retention is record-class-specific (EFTA-003, Appendix A)
- [ ] Incident/dispute runbook exists (EFTA-004)
- [ ] Fraud/abuse controls documented (EFTA-005)
- [ ] Refund path wired end-to-end (EFTA-006)

### Location (if applicable)
- [ ] Location business classification and filing assessed (LIA-002)
- [ ] Ephemeral storage, 10-min max transient cache default (LIA-003)
- [ ] Confirmation-data recording implemented (LIA-004)
- [ ] No exact coords in permanent tables/analytics/backups without approval (LIA-003)

### Breach Readiness
- [ ] 72-hour regulator reporting path built (PIPA-011)
- [ ] User notification templates ready (PIPA-011)

### Retention
- [ ] Retention schedule per dataset/record-class, not blanket (PIPA-009, Appendix A)
- [ ] Destruction automation for purpose-achieved data (PIPA-009)

---

## Code Review Checklist

### Secrets & Config
- [ ] No hardcoded API keys, tokens, DB creds, private keys (ISMS-001)
- [ ] No .env files with real secrets in commits (ISMS-001)
- [ ] Secrets come from vault/env injection (ISMS-001)

### High-Risk Data Handling
- [ ] RRN-like patterns not in logs, analytics, test data (`/\b\d{6}-?\d{7}\b/`) (PIPA-001)
- [ ] Passport/foreigner-reg fields encrypted, masked, access-logged (PIPA-002)
- [ ] No plaintext storage of high-risk identifiers (ISMS-005)
- [ ] High-risk field names flagged: `residentRegistration|rrn|passport|foreignerRegistration|cardNumber|cvv` (heuristic)

### Payment Code
- [ ] No `<input>` for card number or CVV outside PG SDK/iframe (EFTA-001)
- [ ] No PAN/CVV in React state, Redux, Java DTOs, logs, DB columns (EFTA-001)
- [ ] PG tokenization SDK used for card entry (EFTA-001)

### Auth & Access
- [ ] Sensitive endpoints have authorization check, not just authentication (ISMS-002)
- [ ] Session cookies: Secure, HttpOnly, SameSite (ISMS-006)
- [ ] Rate limits on login, password reset, payment initiation (ISMS-010)

### Audit Logging
- [ ] CUD operations generate audit records (ISMS-003)
- [ ] Consent changes logged (ISMS-003)
- [ ] Privileged reads of high-risk data logged (ISMS-003, Appendix G)

### Network & Transport
- [ ] No `http://` external URLs in production config (ISMS-004)
- [ ] No public-facing DB/cache ports (ISMS-009)

### Consent & Privacy in Frontend
- [ ] No pre-checked consent checkboxes (`defaultChecked={true}` on consent) (PIPA-004)
- [ ] Geolocation API preceded by custom pre-prompt, not just browser dialog (LIA-001)
- [ ] Marketing checkboxes separate from service terms (NET-001)
- [ ] Cookie/tracker script loads only after consent where required (NET-005)

### Data Retention
- [ ] No retain-forever defaults for personal data (PIPA-009)
- [ ] Destructive migrations have rollback/backup plan (ISMS-008)

### Dependencies
- [ ] No known high/critical CVEs in dependencies (ISMS-007)
