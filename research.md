# Care Coordination Platform

> Candidate #109 · Researched: 2026-05-01

## Existing Products and Software Packages

| Name | Description | Model | Pricing |
|------|-------------|-------|---------|
| Innovaccer Care Management | AI-powered care management with Copilot; automated documentation, care protocol auto-fill, risk stratification, EHR integration; reduces care manager documentation from 4.5 hours to under 1 hour/day | SaaS / Enterprise | Custom enterprise pricing |
| ZeOmega Jiva | Payer-focused care management and population health platform; care management, benefits administration, population health analytics; serves health plans, health systems, and ACOs | SaaS / On-premise | Custom enterprise pricing |
| Health Catalyst ACE | AI-enabled care management with population analytics; demonstrated $32.2M in reduced healthcare spending for a large health system over 30 months | SaaS / Enterprise | Custom enterprise pricing |
| HealthEdge GuidingCare | Care coordination and utilization management platform for payers; AI-generated clinical summaries from referral packets; prior authorization workflows | SaaS | Custom enterprise pricing |
| Optum One / Optum Care Management | Integrated analytics and care management platform from UnitedHealth Group subsidiary; risk stratification, care gap closure, care plan management | SaaS / Enterprise | Custom enterprise pricing |
| WellSky CarePort | Referral management and care transitions platform; AI-powered clinical summaries, post-acute network management, readmission risk alerts | SaaS | Custom enterprise pricing |
| Salesforce Health Cloud | CRM-based care coordination; patient 360 views, referral tracking, care team collaboration, provider network management | SaaS | $300–$450/user/month |
| Neon Health | Care coordination software for U.S. healthcare; referral management, task automation, care team communication | SaaS | Custom pricing |
| Persivia CareSpace | Population health and care management platform; predictive analytics, outreach automation, care gap management | SaaS | Custom enterprise pricing |
| Medecision Aerial | Cloud-based care management and utilization management for health plans; condition management, care pathway automation | SaaS | Custom enterprise pricing |

## Relevant Industry Standards or Protocols

| Standard | Relevance |
|----------|-----------|
| HL7 FHIR R4 (CMS Interoperability Rules) | API-based data exchange standard; mandated under CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F) for payer-to-payer data exchange and patient access APIs; essential for multi-provider care plan sharing |
| HL7 C-CDA (Consolidated Clinical Document Architecture) | Standard for Continuity of Care Documents (CCDs) and Referral Notes; required for Promoting Interoperability / Meaningful Use compliance; governs electronic clinical summaries at care transitions |
| HL7 Da Vinci CDex (Clinical Data Exchange) | FHIR-based IG enabling provider-to-payer and provider-to-provider clinical data exchange; superior to CCD for structured prior authorization and quality reporting workflows |
| HL7 FHIR SDOH IG (Social Determinants of Health) | Implementation guide for capturing and exchanging social needs data (housing, food insecurity, transportation) within care coordination workflows |
| CMS Transitional Care Management (TCM) Codes (99495–99496) | Billing requirements for care coordination during 30-day post-discharge period; platforms must document required components of TCM services |
| CMS Chronic Care Management (CCM) Codes (99490–99491) | Monthly billing for care coordination for patients with 2+ chronic conditions; platforms must track time and document required CCM activities |
| NCQA HEDIS Measures | Health plan quality measures including care coordination metrics (e.g., transitions of care, follow-up after hospitalization); platforms often optimize workflows to improve HEDIS scores |
| ONC United States Core Data for Interoperability (USCDI) | Minimum dataset required for health IT interoperability; defines data elements care coordination platforms must support for patient record exchange |

## Available Research Materials

| Citation | Type |
|----------|------|
| Coleman, E.A. et al. (2006). "The care transitions intervention: results of a randomized controlled trial." *Archives of Internal Medicine*, 166(17), 1822–1828. | Randomized controlled trial |
| Naylor, M.D. et al. (2011). "The care span: the importance of transitional care in achieving health reform." *Health Affairs*, 30(4), 746–754. | Policy review |
| Bodenheimer, T. (2008). "Coordinating care — a perilous journey through the health care system." *New England Journal of Medicine*, 358(10), 1064–1071. | Commentary |
| McDonald, K.M. et al. (2007). "Closing the quality gap: a critical analysis of quality improvement strategies (Vol. 7: Care Coordination)." AHRQ Technical Review 9.7. Rockville, MD: AHRQ. | Technical review |
| Schulman-Green, D. et al. (2012). "Processes and patterns of coping and self-management in chronic illness: a qualitative meta-synthesis." *Research in Nursing and Health*, 35(4), 394–411. | Qualitative meta-synthesis |
| Pham, H.H. et al. (2007). "Care patterns in Medicare and their implications for pay for performance." *New England Journal of Medicine*, 356(11), 1130–1139. | Empirical study |
| Jha, A.K. et al. (2024). "Digital Information Ecosystems in Modern Care Coordination and Patient Care Pathways and the Challenges and Opportunities for AI Solutions." *Journal of Medical Internet Research*, 26, e60258. | Perspective article |
| McAlister, F.A. et al. (2001). "Multidisciplinary strategies for the management of heart failure patients at high risk for admission." *Journal of the American College of Cardiology*, 38(5), 1496–1502. | Systematic review |
| Peikes, D. et al. (2009). "Effects of care coordination on hospitalization, quality of care, and health care expenditures among Medicare beneficiaries." *JAMA*, 301(6), 603–618. | Randomized trial |

## Market Research

**Market Size:** The global care management solutions market was valued at $17–18 billion in 2025, projected to reach $34–60 billion by 2030–2034, growing at a CAGR of 13–15% (Precedence Research; Fortune Business Insights; Grand View Research; Mordor Intelligence).

**Pricing Landscape:**

| Segment | Typical Pricing |
|---------|----------------|
| Enterprise health system / payer (Innovaccer, ZeOmega, Optum) | Custom; typically $500K–$5M+ per year depending on covered lives |
| Mid-market care management (HealthEdge, Persivia) | $200K–$1M per year |
| Post-acute / referral management (WellSky CarePort) | Per-facility or per-referral transaction pricing |
| CRM-based (Salesforce Health Cloud) | $300–$450/user/month; implementation typically $500K–$2M |
| Per-member-per-month (PMPM) models | $1–$10 PMPM depending on program intensity |

**Key Buyer Personas:**
- Chief Medical Officers and VP of Care Management at health systems
- Health plan Medical Directors and VP Population Health
- Accountable Care Organization (ACO) administrators
- Medicaid managed care plan executives
- Value-based care program directors at physician groups and IPAs

**Notable Funding / Acquisitions:**
- Innovaccer raised $150M Series E at $3.2B valuation (2022); total funding $375M+; positioned as "agentic cloud for healthcare"
- Health Catalyst IPO'd in 2019 (NASDAQ: HCAT); ~$250M annual revenue; announced acquisition of Lumeon care pathway platform (2023)
- WellSky (private equity, TPG Capital) is a large post-acute software consolidator; acquired CarePort from Epic in 2023
- Salesforce Health Cloud launched in 2016; dedicated healthcare CRM with significant care coordination investment
- IDC MarketScape published its U.S. Care Coordination Technology 2024–2025 Vendor Assessment (US52720924), covering major platform vendors

## AI-Native Opportunity

- **Autonomous care gap closure:** AI agents can continuously scan patient records against evidence-based care guidelines, generate and assign outreach tasks, draft personalized member communications, and close the loop without care manager intervention for the large proportion of low-complexity gaps
- **Intelligent referral management:** With referral leakage rates of 55–65% industry-wide, AI can predict referral abandonment risk, match patients to in-network specialists by wait time, location, and outcomes data, and automate scheduling confirmation, reducing leakage and improving network retention
- **Predictive readmission and escalation prevention:** ML models incorporating social determinants of health, claims history, lab trends, and care plan adherence can identify patients likely to be readmitted 7–14 days before the event, enabling targeted transitional care interventions
- **Automated transitions of care documentation:** At discharge, AI can synthesize inpatient records into structured transition of care summaries, push them via FHIR to the receiving provider within 24 hours, and initiate a follow-up care plan automatically, addressing the most common failure point in care transitions
- **Natural language care plan management:** LLMs can draft, update, and reconcile multi-provider care plans through conversational interfaces, resolving conflicting orders and aligning goals of care across primary care, specialty, behavioral health, and social services — a coordination task currently requiring hours of human phone and fax work
