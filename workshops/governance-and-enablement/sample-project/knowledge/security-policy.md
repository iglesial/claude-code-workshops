# Acme Corp — Information Security Policy

**Version:** 3.1  
**Last reviewed:** January 2025  
**Owner:** Head of Engineering & CISO (shared)  
**Applies to:** All employees, contractors, and systems processing Acme Corp data or customer data

---

## 1. Data Classification

All data handled by Acme Corp is classified into one of four tiers:

| Tier | Label | Examples | Handling |
|------|-------|----------|----------|
| 1 | Public | Marketing materials, product documentation | No restrictions |
| 2 | Internal | Process docs, org charts, roadmaps | Internal access only, no external sharing without approval |
| 3 | Confidential | Customer contracts, pricing, personal data | Encrypted at rest and in transit, access logged |
| 4 | Restricted | Credentials, encryption keys, audit logs | Access by named individuals only, MFA required |

Customer data is always classified as Confidential (Tier 3) or Restricted (Tier 4).

---

## 2. Access Control

- All systems require multi-factor authentication (MFA). SMS-based MFA is not accepted; TOTP or hardware keys only.
- Access follows the principle of least privilege. Role-based access control (RBAC) is enforced in all production systems.
- Employee access is reviewed quarterly. Accounts are deprovisioned within 24 hours of departure.
- Contractors receive time-limited access scoped to the minimum required for their engagement.
- Shared credentials are prohibited. All access must be attributable to an individual.
- Production database access requires a written request, manager approval, and is logged with a ticket reference.

---

## 3. Encryption

- **At rest:** AES-256 for all customer data stored on Acme Corp infrastructure.
- **In transit:** TLS 1.2 minimum; TLS 1.3 preferred. SSLv3 and TLS 1.0/1.1 are disabled.
- **Key management:** AWS KMS. Key rotation is enforced annually. Master keys are never exported.
- **Backups:** Encrypted with a separate key from the primary data. Backup keys are stored in a separate AWS account.

---

## 4. Data Retention and Deletion

| Data type | Retention period | Deletion method |
|-----------|-----------------|-----------------|
| Customer contracts | 7 years from contract end | Secure deletion via NIST 800-88 |
| Personal data (GDPR) | As long as the contract is active + 2 years, or as required by law | Cryptographic erasure |
| Application logs | 12 months | Automated purge |
| Security / audit logs | 3 years | Immutable log store (no manual deletion) |
| Support tickets | 5 years | Soft delete with annual hard purge |

Customer data deletion requests are processed within 30 days. Proof of deletion is provided on request.

---

## 5. Incident Response

Acme Corp follows a four-phase incident response process:

**Phase 1 — Detection (target: < 4 hours)**  
Automated monitoring (AWS GuardDuty, Datadog) triggers alerts. The on-call engineer confirms or dismisses within 30 minutes.

**Phase 2 — Containment (target: < 2 hours after confirmation)**  
Affected systems are isolated. No data is deleted during containment. All actions are logged.

**Phase 3 — Notification**  
- Customers whose data may be affected are notified within 72 hours (GDPR Article 33/34 compliance).
- Regulatory authorities (CNIL for France, ICO for UK) are notified within 72 hours if required.
- Internal leadership is notified within 1 hour of confirmation.

**Phase 4 — Post-incident review**  
A written post-mortem is produced within 5 business days. Root cause analysis and remediation steps are shared with affected customers on request.

---

## 6. Subprocessors

Acme Corp uses the following subprocessors to deliver its service. All are under Data Processing Agreements (DPAs) compliant with GDPR:

| Subprocessor | Purpose | Location |
|-------------|---------|----------|
| Amazon Web Services (AWS) | Cloud infrastructure (EU-WEST-1, Paris) | EU |
| Stripe | Payment processing | US (Standard Contractual Clauses in place) |
| Intercom | Customer support chat | US (Standard Contractual Clauses in place) |
| Datadog | Application monitoring and logging | US (Standard Contractual Clauses in place) |
| DocuSign | E-signature (used by customers via the platform) | US (Standard Contractual Clauses in place) |

Customers are notified of new subprocessors with 30 days' notice. Objection rights are respected.

---

## 7. Penetration Testing and Vulnerability Management

- An independent third-party penetration test is conducted annually. The most recent test was completed in October 2024 by Netsecure Consulting.
- Critical and high vulnerabilities are remediated within 72 hours. Medium within 30 days. Low within 90 days.
- Vulnerability scan results and remediation status are available to enterprise customers under NDA on request.
- A responsible disclosure programme is in place at security@acmecorp.io.

---

## 8. Business Continuity

- **Recovery Point Objective (RPO):** 4 hours — automated backups run every 4 hours.
- **Recovery Time Objective (RTO):** 8 hours — full failover to secondary region tested bi-annually.
- **Uptime SLA:** 99.9% monthly (measured excluding scheduled maintenance windows).
- Business continuity plans are tested twice per year with full simulation exercises.

---

## 9. Employee Security

- All employees complete security awareness training annually (mandatory, tracked in the HR system).
- New joiners complete training within their first week before receiving production access.
- Background checks are conducted for all employees with access to Restricted data.
- A clean desk policy applies in all Acme Corp offices.

---

## 10. Physical Security

All customer data is hosted in AWS data centres (EU-WEST-1, Paris region). Acme Corp does not operate its own data centres. AWS maintains ISO 27001, SOC 2 Type II, and CSA STAR Level 2 certifications for these facilities.

Acme Corp office access is controlled by key card. Visitor logs are maintained. No customer data is stored on office-based hardware.

---

## Certifications

- **ISO 27001:2022** — certified since 2022, last audit November 2024
- **SOC 2 Type II** — last report covers May 2023 – May 2024, available under NDA

To request a copy of our ISO 27001 certificate or SOC 2 report, contact security@acmecorp.io.
