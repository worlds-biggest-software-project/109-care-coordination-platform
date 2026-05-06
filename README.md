# Care Coordination Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source care coordination platform for multi-provider care plans, referral management, and transitions of care.

The Care Coordination Platform unifies clinical, claims, and social-determinant data into a single longitudinal view, then drives care plans, outreach, and transitions across primary care, specialty, behavioural health, and community providers. It is built for ACOs, health plans, health systems, and value-based physician groups that need to close care gaps, prevent readmissions, and reduce referral leakage without the multi-million-dollar price tag of enterprise incumbents.

---

## Why Care Coordination Platform?

- Enterprise incumbents (Innovaccer, ZeOmega, Optum, Health Catalyst) charge $500K–$5M+ per year and require 9–24 month implementations, putting modern care coordination out of reach for smaller ACOs and physician groups.
- Salesforce Health Cloud charges $300–$450 per user per month with $500K–$2M implementation costs, making broad care-team deployment prohibitively expensive.
- Industry-wide referral leakage rates of 55–65% persist because most platforms track referrals but do not predict abandonment risk or automate scheduling confirmation.
- Most platforms screen for social determinants of health but do not close the loop to community-based organisations, leaving identified social needs unresolved.
- AI capabilities across the incumbent landscape are uneven: Innovaccer and Navina lead on clinical AI, while ZeOmega, Persivia, and Medecision lag — leaving room for an AI-native open alternative built on open standards.

---

## Key Features

### Data Aggregation and Interoperability

- Multi-source patient data ingestion via HL7 FHIR R4, HL7 v2, and C-CDA from EHR, lab, and pharmacy systems
- Claims data ingestion using ANSI X12 837/835 for payer-provided attribution and utilisation history
- Compliance with the CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F) for payer-to-payer and patient access APIs
- HL7 Da Vinci CDex for structured provider-to-payer and provider-to-provider clinical data exchange
- USCDI-aligned patient record exchange across the care network

### Care Management Workflow

- Care plan creation, task assignment, and progress tracking with multi-disciplinary team access
- Care manager worklist with prioritised patient queue and recommended next actions
- ML-based risk stratification across chronic condition burden, recent utilisation, and SDOH-derived social risk
- Care gap identification mapped to CMS quality programme measures and evidence-based preventive care guidelines
- CMS Transitional Care Management (99495–99496) and Chronic Care Management (99490–99491) billing documentation with time tracking

### Patient Engagement and Outreach

- Automated member outreach across SMS, email, and phone with response capture and care plan update
- SDOH screening data capture using PRAPARE or AHC-HRSN with closed-loop referral to community-based organisations
- Patient engagement portal with care plan visibility, self-management resources, and secure messaging
- Configurable rules engine for outreach automation without custom development

### Transitions and Referrals

- Post-acute referral management with provider network directory, quality metrics, and referral tracking
- AI-generated clinical summaries for referral packets and transitions-of-care documents
- Readmission risk alerts at point of discharge integrated with hospital EHR workflow
- Predictive identification of referrals at risk of abandonment, with automated scheduling confirmation

### Quality, Analytics, and Reporting

- HEDIS measure calculation and reporting (NCQA licence required for commercial use)
- Population cohort analytics with drill-down to individual patient action panels
- Provider and care manager performance dashboards
- Predictive readmission and escalation models incorporating SDOH, claims, labs, and care plan adherence

---

## AI-Native Advantage

AI is embedded in the workflow rather than bolted on. Autonomous care gap closure agents identify gaps, generate outreach, capture responses, and update care plans without care manager intervention for low-complexity cases. Clinical NLP extracts structured diagnoses and HCC risk-adjustment codes from unstructured notes, while LLM-driven natural language interfaces draft, update, and reconcile multi-provider care plans — resolving conflicting orders across primary care, specialty, behavioural health, and social services. Automated transitions-of-care synthesis pushes structured discharge summaries via FHIR to receiving providers within 24 hours of discharge, addressing the most common failure point in care transitions.

---

## Tech Stack & Deployment

The platform targets cloud-native deployment with self-hosted and hybrid options for health plans with on-premise data residency requirements. Interoperability is built on open standards: HL7 FHIR R4 (Creative Commons), HL7 C-CDA, HL7 Da Vinci CDex, the HL7 FHIR SDOH Implementation Guide, and ONC USCDI. HAPI FHIR (Apache 2.0) provides an embeddable open-source FHIR server. EHR integration covers Epic, Oracle Health (Cerner), eClinicalWorks, and Meditech via certified FHIR APIs and HL7 v2 ADT feeds. NCQA HEDIS measure specifications require a separate commercial licence from NCQA.

---

## Market Context

The global care management solutions market was valued at $17–18 billion in 2025 and is projected to reach $34–60 billion by 2030–2034, growing at 13–15% CAGR (Precedence Research; Fortune Business Insights; Grand View Research; Mordor Intelligence). Enterprise platforms charge $500K–$5M+ per year for health systems and payers, with PMPM models ranging $1–$10 depending on programme intensity. Primary buyers are Chief Medical Officers and VP Care Management at health systems, Medical Directors and VP Population Health at health plans, ACO administrators, Medicaid managed care executives, and value-based care directors at physician groups and IPAs.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
