# Care Coordination Platform — Feature & Functionality Survey

> Candidate #109 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Innovaccer Health Cloud | Commercial SaaS | Proprietary (private, $3.2B valuation) | innovaccer.com |
| ZeOmega Jiva | Commercial SaaS / On-premise | Proprietary (private) | zeomega.com |
| Health Catalyst ACE | Commercial SaaS | Proprietary (NASDAQ: HCAT) | healthcatalyst.com |
| WellSky CarePort | Commercial SaaS | Proprietary (PE-backed, TPG Capital) | wellsky.com |
| Salesforce Health Cloud | Commercial SaaS | Proprietary (NYSE: CRM) | salesforce.com/health |
| Arcadia | Commercial SaaS | Proprietary (private) | arcadia.io |
| Philips Wellcentive | Commercial SaaS | Proprietary (Philips, NYSE: PHG) | philips.com |
| Navina | Commercial SaaS (AI-native) | Proprietary (private) | navina.ai |
| Persivia CareSpace | Commercial SaaS | Proprietary (private) | persivia.com |
| Medecision Aerial | Commercial SaaS | Proprietary (private) | medecision.com |

## Feature Analysis by Solution

### Innovaccer Health Cloud

**Core features**
- AI-powered care management with Copilot assistant for care managers
- Automated care protocol population and documentation from AI-analysed patient records
- Risk stratification: ML-based risk scoring across chronic conditions, gaps in care, and social determinants
- Care plan creation, assignment, and monitoring across multi-provider care teams
- Population health analytics: cohort identification, gap closure tracking, quality measure reporting
- EHR integration hub: 50+ EHR connectors aggregating data into unified patient record
- Patient engagement: automated outreach campaigns, portal access, remote monitoring data ingestion
- HEDIS measure reporting and analytics

**Differentiating features**
- Documented reduction in care manager documentation time from 4.5 hours to under 1 hour/day via AI Copilot
- "Agentic cloud for healthcare" positioning: AI agents performing autonomous care gap closure tasks
- Innovaccer Data Activation Platform: unified data layer across disparate EHRs for a single patient view
- Clinical-grade NLP to extract structured data from unstructured clinical notes at scale
- Pre-built value-based care programme templates (MSSP, REACH ACO, state Medicaid managed care)

**UX patterns**
- Care manager worklist with AI-prioritised patient tasks and recommended next actions
- Copilot assistant embedded in care plan workflow: drafts care plan sections and protocol-required fields
- Population health dashboard with cohort drill-down to individual patient action panels
- Risk stratification heat maps by geography, provider panel, and condition

**Integration points**
- 50+ EHR connectors via HL7 FHIR R4, HL7 v2, C-CDA, and proprietary APIs
- CMS Interoperability and Prior Authorization Rule compliance (FHIR-based payer-to-payer data exchange)
- Surescripts medication history
- Claims data ingestion from payers (ANSI X12 837/835)
- Social determinants of health screening tool integrations (PRAPARE, AHC-HRSN)
- NCQA HEDIS measure calculations
- State Medicaid managed care data feeds

**Known gaps**
- Implementation complexity is high; data normalisation across multiple EHRs requires significant technical services engagement
- AI Copilot requires structured data quality; sites with poor documentation practices see lower benefit
- Premium pricing limits accessibility for smaller ACOs and physician groups

**Licence / IP notes**
- Fully proprietary; Innovaccer Inc. (private company, San Francisco, $375M+ total funding)
- No open-source components

---

### ZeOmega Jiva

**Core features**
- Comprehensive care management platform for health plans, ACOs, and health systems
- Care management: care plan creation, task management, inter-disciplinary team communication
- Utilisation management (UM): prior authorisation, concurrent review, discharge planning
- Population health analytics: risk stratification, gap identification, HEDIS measure tracking
- Benefits administration integration for health plan workflows
- Member portal with care plan access and health education content
- Disease management and condition-specific care pathways
- Regulatory reporting for CMS, NCQA, and URAC

**Differentiating features**
- Deep health plan functionality — both care management and utilisation management in one system
- NCQA Case Management Accreditation support built into workflow
- Longitudinal member record spanning care management, UM, and benefits data
- Configurable care pathway engine for condition-specific and programme-specific protocols

**UX patterns**
- Case manager workflow with prioritised worklist and task queue
- Integrated UM review workflow within the same care manager interface
- Member 360 view with care history, benefit information, and social risk flags

**Integration points**
- HL7 FHIR R4 and C-CDA for EHR and payer data exchange
- CMS Da Vinci CDex for provider-to-payer clinical data exchange
- Claims data feeds (ANSI X12)
- NCQA HEDIS reporting engines
- Benefits administration system connectors (Facets, HealthEdge)

**Known gaps**
- UI is functional but less modern than newer cloud-native competitors
- AI and ML capabilities are less prominent than Innovaccer or Navina
- On-premise deployment options increase implementation and maintenance burden for health plans

**Licence / IP notes**
- Proprietary; ZeOmega Inc. (private company)
- No open-source components

---

### Health Catalyst ACE (Analytics-Enabled Care Management)

**Core features**
- AI-enabled care management with population analytics backbone
- Predictive risk stratification using ML models trained on clinical and claims data
- Care gap identification and closure workflow
- Outcomes tracking: demonstrated $32.2M savings over 30 months for a large health system
- Population cohort analysis and performance benchmarking
- Lumeon care pathway automation (acquired 2023): automated care protocol execution
- Quality measure reporting (HEDIS, CMS quality programmes)

**Differentiating features**
- Health Catalyst Data Operating System (DOS): scalable cloud data warehouse underpins analytics
- Lumeon integration: care pathway automation engine orchestrates multi-step protocols automatically without manual triggering
- Proven financial outcomes published by health system clients (publicly traded company with audited metrics)
- Analytics depth across the full care continuum from raw clinical data to quality reporting

**UX patterns**
- Analytics-first interface; care managers start from population analytics views and drill to individual patients
- Lumeon care pathway visualiser for protocol design and execution monitoring
- Executive dashboards with outcome and financial impact metrics

**Integration points**
- HL7 FHIR R4, HL7 v2, C-CDA for EHR data ingestion
- Claims data warehouse feeds
- NCQA HEDIS measure reporting
- CMS value-based care programme data submission

**Known gaps**
- Primary strength is analytics and care pathway automation; direct care manager UX is less refined than dedicated care management platforms
- Lumeon integration (2023) is still maturing within the Health Catalyst ecosystem
- Deployment complexity; Health Catalyst implementations typically require 12–24 months

**Licence / IP notes**
- Proprietary; Health Catalyst Inc. (NASDAQ: HCAT)
- No open-source components in the commercial platform; Health Catalyst has published open-source data tools (Catalyst.AI open-source utilities under Apache 2.0) but not the core platform

---

### WellSky CarePort

**Core features**
- Referral management and care transitions platform connecting hospitals to post-acute care (PAC)
- AI-powered clinical summaries generated from inpatient records for referral packets
- Post-acute provider network management: SNF, home health, hospice, IRF directories
- Readmission risk alerts at point of discharge
- Care transition tracking: referral accepted/declined, placement confirmed, follow-up status
- Discharge planning workflow integrated with hospital EHR
- PAC analytics: readmission rates by receiving facility, length of stay in PAC, network performance

**Differentiating features**
- Largest post-acute referral network in the U.S. (acquired from Epic in 2023); 500,000+ post-acute providers connected
- AI-generated clinical summaries reduce manual referral packet preparation time
- CarePort Insights: analytics on post-acute network performance and readmission patterns
- Integration with Epic, Cerner, and Meditech for hospital discharge workflow

**UX patterns**
- Hospital discharge planner view: patient list with readmission risk flag, pending referrals, and PAC options sorted by quality metrics
- Post-acute provider portal: referral acceptance, clinical summary review, care confirmation
- Analytics dashboard for care transitions coordinators and health system executives

**Integration points**
- Epic, Oracle Health (Cerner), Meditech via HL7 FHIR R4 and v2 ADT
- HL7 C-CDA for transition of care documents
- CMS Transitional Care Management (TCM) billing documentation support
- State health information exchange (HIE) connections

**Known gaps**
- Focused on post-acute transitions; upstream care management (chronic disease, population health) is outside scope
- AI clinical summaries require high-quality inpatient documentation to be accurate
- Limited social determinants of health screening integration

**Licence / IP notes**
- Proprietary; WellSky Corporation (PE-backed, TPG Capital)
- No open-source components

---

### Salesforce Health Cloud

**Core features**
- CRM-based care coordination with patient 360 view
- Care team collaboration workspace: tasks, notes, calls, cases
- Referral and prior authorisation tracking
- Provider network management and directory
- Automated outreach and engagement workflows
- Care programme management with configurable care plans
- Einstein AI (Salesforce AI): care gap scoring, next best action recommendations
- FHIR R4 integration via Health Cloud FHIR APIs

**Differentiating features**
- Salesforce platform strength: world-class CRM, workflow automation, and AppExchange ecosystem
- Highly configurable without custom code; configurable care plans, workflows, and analytics
- Marketing Cloud integration for sophisticated member outreach campaigns (segmentation, multi-channel)
- Massive partner ecosystem: hundreds of healthcare ISV apps in AppExchange
- Einstein AI embedded across care plan recommendations, risk scoring, and documentation

**UX patterns**
- CRM-style interface familiar to healthcare staff with sales or service backgrounds
- Lightning App Builder for no-code care coordination workflow customisation
- Mobile app for care team field workers

**Integration points**
- HL7 FHIR R4 via Health Cloud FHIR APIs
- CMS Interoperability rules compliance for payer use cases
- HL7 Da Vinci CDex for provider-to-payer exchange
- EHR connectors via MuleSoft integration platform
- HRIS and payer claims data via Salesforce Data Cloud

**Known gaps**
- High per-user cost ($300–$450/user/month) makes broad care team deployment expensive
- Requires significant Salesforce expertise to configure for complex clinical workflows; implementation cost often $500K–$2M+
- Not an EHR; clinical documentation is supplementary, not primary
- Einstein AI healthcare use cases are less specialised than purpose-built clinical AI platforms

**Licence / IP notes**
- Proprietary; Salesforce Inc. (NYSE: CRM)
- No open-source components

---

### Arcadia

**Core features**
- Population health analytics and care coordination for ACOs, health systems, and payers
- Multi-source data aggregation: EHR, claims, lab, pharmacy, SDOH
- Risk stratification and predictive analytics for population management
- Care gap identification and closure workflow
- Quality measure calculation and reporting (HEDIS, CAHPS, CMS Stars)
- Provider and care manager performance dashboards
- Attribution management for value-based care programme populations

**Differentiating features**
- Deep multi-payer claims and clinical data integration specialisation
- Attribution management for complex value-based care programme populations across multiple payer contracts
- HEDIS measure production with audit trail — used by health plans for NCQA submissions
- Arcadia Analytics: self-service analytics platform on top of the population health data layer

**UX patterns**
- Analytics-forward interface; users start with population cohorts and drill to individual care gaps
- Care gap closure worklist with automation for high-volume outreach
- Provider performance scorecards for network management

**Integration points**
- HL7 FHIR R4, HL7 v2, C-CDA for clinical data
- Multi-payer claims ingestion (ANSI X12 837/835)
- Lab and pharmacy data feeds
- SDOH screening data integration
- NCQA HEDIS measure reporting

**Known gaps**
- Primary strength is analytics; direct care management workflow (care plans, case notes) is less developed than Innovaccer or ZeOmega
- Implementation data normalisation is complex; typical timelines 9–18 months
- Less suited to direct patient engagement features (portal, outreach)

**Licence / IP notes**
- Proprietary; Arcadia.io Inc. (private company)
- No open-source components

---

### Philips Wellcentive

**Core features**
- Population health management for physician practices and health systems
- Risk stratification and care gap identification
- Quality measure tracking (HEDIS, CMS Stars)
- Care coordination workflow: outreach, follow-up task management
- Registry management for chronic disease populations
- Provider performance dashboards
- Patient engagement: automated outreach for preventive care gaps

**Differentiating features**
- Integration with Philips clinical monitoring and remote patient monitoring (RPM) devices for high-acuity populations
- Wellcentive Analytics: physician-level performance dashboards for value-based care programme management
- Physician practice-focused UX designed for ambulatory quality improvement workflows

**UX patterns**
- Practice-level population health dashboard with payer programme performance tracking
- Care gap closure worklist with automated outreach queue
- Provider scorecard for physician group leadership

**Integration points**
- HL7 FHIR R4 and v2 for EHR connectivity
- Claims data ingestion for payer-provided attribution
- Philips RPM and patient monitoring device data integration
- Quality measure reporting (HEDIS, CMS)

**Known gaps**
- Smaller market presence compared to Innovaccer, Arcadia, and Health Catalyst
- Limited AI and predictive analytics depth
- Philips' strategic focus on medical devices means population health software investment is secondary

**Licence / IP notes**
- Proprietary; Philips (Koninklijke Philips N.V., NYSE: PHG)
- No open-source components

---

### Navina

**Core features**
- AI-powered patient portrait for primary care and value-based care
- Real-time clinical synthesis: aggregates EHR, claims, labs, notes into a structured patient summary before a visit
- NLP-driven HCC risk adjustment: identifies undocumented or suspect diagnosis codes from unstructured notes
- Care gap closure recommendations surfaced at point of care within EHR workflow
- Chronic disease management dashboard with attribution and quality gap visibility
- Prior authorisation documentation support

**Differentiating features**
- AI generates a pre-visit patient portrait in real time (30–60 seconds) from all available clinical data
- Among the most advanced clinical NLP for HCC coding and risk adjustment in primary care
- Embedded in EHR workflow (Epic, Cerner, eClinicalWorks) as a sidebar — no separate application required
- Clinician-facing rather than care manager-facing; differentiates from population health platforms that target care management staff

**UX patterns**
- EHR sidebar showing pre-visit AI patient portrait: active problems, recent events, care gaps, open HCC suspect codes
- One-click care gap closure and documentation within the EHR encounter
- Care gap and HCC analytics dashboard for value-based care programme managers

**Integration points**
- Epic, Oracle Health (Cerner), eClinicalWorks via FHIR R4 and certified EHR API
- Claims data from payers for risk adjustment and attribution
- HL7 Da Vinci CDex for payer clinical data exchange

**Known gaps**
- Primarily point-of-care clinical tool; less suited to between-visit population-level care management workflow
- Care plan management and longitudinal care coordination are not core features
- Requires EHR integration for full value; standalone deployment is limited

**Licence / IP notes**
- Proprietary; Navina Inc. (private company, Israeli-founded, U.S. market focus)
- No open-source components

---

### Persivia CareSpace

**Core features**
- Population health and care management platform
- Predictive analytics: risk stratification using ML across chronic conditions and social risk
- Care gap identification and automated outreach
- Care plan management with inter-disciplinary team collaboration
- Chronic disease management protocols
- Patient engagement: portal, automated SMS/email outreach
- HEDIS measure calculation and quality reporting
- Utilisation management integration

**Differentiating features**
- End-to-end platform spanning population analytics through to direct patient engagement
- Pre-built condition-specific care programmes (diabetes, CHF, COPD, behavioural health)
- Configurable rules engine for outreach automation without custom development
- Lower total cost of ownership compared to enterprise platforms like Innovaccer for mid-market ACOs

**UX patterns**
- Care manager dashboard with risk-stratified worklist and recommended actions
- Patient 360 view with care plan, gap status, and outreach history
- Analytics dashboard for programme directors

**Integration points**
- HL7 FHIR R4 and v2 for EHR connectivity
- Claims data ingestion
- Social determinants screening data integration
- NCQA HEDIS reporting

**Known gaps**
- Less brand recognition than tier-1 competitors; fewer published outcomes studies
- AI capabilities are less prominent than Innovaccer or Navina
- Implementation services depth is smaller than enterprise competitors

**Licence / IP notes**
- Proprietary; Persivia Inc. (private company)
- No open-source components

---

### Medecision Aerial

**Core features**
- Cloud-based care management and utilisation management for health plans
- Condition management with evidence-based care pathways
- Care plan automation and care pathway execution
- Member engagement: portal, outreach, health education
- Utilisation review: prior authorisation, concurrent review, discharge planning
- HEDIS measure reporting and NCQA accreditation support
- SDOH screening integration
- Predictive analytics for member risk stratification

**Differentiating features**
- Aerial platform positioned as a digital health companion for health plan members
- Condition management pathways with automated member touchpoints
- Integrated UM and care management in one workflow for payer operations

**UX patterns**
- Case manager workflow with care plan and UM review in unified interface
- Member-facing digital companion with personalised content and health coaching content
- HEDIS reporting dashboard for quality programme managers

**Integration points**
- HL7 FHIR R4 and C-CDA for provider data exchange
- Claims data integration (ANSI X12)
- NCQA HEDIS measure calculation engines
- SDOH screening tool integration

**Known gaps**
- Less prominent than ZeOmega or Innovaccer in recent market assessments
- AI capabilities are less advanced than leading competitors
- Implementation timelines can be long for large health plan deployments

**Licence / IP notes**
- Proprietary; Medecision Inc. (private company)
- No open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-source patient data aggregation: EHR (FHIR R4, HL7 v2, C-CDA), claims (ANSI X12), labs, pharmacy
- Risk stratification using ML or rules-based scoring across chronic condition burden, utilisation, and social risk
- Care gap identification mapped to HEDIS measures, CMS quality programmes, and evidence-based guidelines
- Care plan creation, assignment, and progress tracking with multi-disciplinary team collaboration
- Task management and care manager worklist with priority ordering
- Member/patient outreach: automated call, SMS, and portal communication
- HEDIS measure calculation and quality reporting
- CMS TCM and CCM billing code documentation support (99490–99496)
- HL7 FHIR R4 compliance for CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F)
- SDOH screening data capture and integration (HL7 FHIR SDOH IG)

### Differentiating Features
- AI Copilot for care managers: autonomous care protocol completion and documentation from patient record (Innovaccer)
- Pre-visit AI patient portrait: real-time clinical synthesis at the point of care within EHR workflow (Navina)
- Care pathway automation: autonomous multi-step protocol execution without manual triggers (Health Catalyst Lumeon)
- AI-generated clinical summaries for referral packets (WellSky CarePort)
- HCC risk adjustment NLP identifying underdocumented diagnoses in unstructured notes (Navina)
- Post-acute network management with referral leakage analytics and provider quality benchmarking (WellSky)
- Salesforce platform configurability with AppExchange ecosystem and Marketing Cloud outreach sophistication
- Integrated UM and care management in a single workflow for payer operations (ZeOmega, Medecision)

### Underserved Areas / Opportunities
- Autonomous care gap closure: AI agents that complete outreach, capture responses, and update care plans without care manager intervention for low-complexity gaps
- Referral leakage prevention: predictive identification of referrals at risk of abandonment, with automated scheduling confirmation
- Automated transitions-of-care documentation: AI-synthesised discharge summaries pushed via FHIR to receiving provider within 24 hours of discharge
- SDOH closed-loop referral: connection from SDOH screening through to community-based organisation confirmation of resource provision (most platforms screen but do not close the loop)
- Cross-provider longitudinal care plan: single reconciled care plan shared across primary care, specialty, behavioural health, and social services with conflict resolution
- Natural language care coordination: LLM-driven conversational interface for updating care plans, documenting calls, and resolving order conflicts

### AI-Augmentation Candidates
- Autonomous care gap closure agents: identify gap, generate outreach, capture member response, update care plan and measure status
- Predictive readmission and escalation models 7–14 days ahead incorporating SDOH, claims, labs, and care plan adherence
- AI pre-visit patient portrait: real-time synthesis of all available data into structured clinical summary for care team
- NLP extraction of structured diagnoses and HCC codes from unstructured clinical notes
- Automated transitions-of-care documentation: AI-generated structured discharge summary with FHIR push to receiving provider
- Referral management AI: predict referral abandonment risk, match to optimal in-network specialist, automate scheduling confirmation
- Natural language care plan management: LLM interface for drafting, updating, and reconciling multi-provider care plans

## Legal & IP Summary

| Tool | Licence Type | Embedding Risk |
|------|-------------|----------------|
| Innovaccer Health Cloud | Proprietary (private) | Cannot embed; commercial licence required |
| ZeOmega Jiva | Proprietary (private) | Cannot embed; commercial licence required |
| Health Catalyst ACE | Proprietary (NASDAQ: HCAT) | Cannot embed; commercial licence required; open-source utilities (Apache 2.0) available separately |
| WellSky CarePort | Proprietary (PE-backed) | Cannot embed; commercial licence required |
| Salesforce Health Cloud | Proprietary (NYSE: CRM) | Cannot embed; commercial SaaS subscription required |
| Arcadia | Proprietary (private) | Cannot embed; commercial licence required |
| Philips Wellcentive | Proprietary (NYSE: PHG) | Cannot embed; commercial licence required |
| Navina | Proprietary (private) | Cannot embed; commercial licence required |
| Persivia CareSpace | Proprietary (private) | Cannot embed; commercial licence required |
| Medecision Aerial | Proprietary (private) | Cannot embed; commercial licence required |

No GPL or AGPL components are identified in the primary competitor set. Open standards used in this space — HL7 FHIR (Creative Commons licence), NCQA HEDIS measure specifications (licensed from NCQA; commercial users must obtain licence), and CMS quality measure specifications (public domain) — have different IP profiles. HEDIS measure specifications require a formal licence from NCQA for commercial use; this is a notable IP obligation for any platform calculating and reporting HEDIS measures. HAPI FHIR (Apache 2.0) provides an open-source FHIR server that can be embedded without licence concerns.

## Recommended Feature Scope

**Must-have (MVP)**:
- Multi-source patient data aggregation via HL7 FHIR R4, HL7 v2, and C-CDA from EHR, claims, and lab systems
- ML-based risk stratification: chronic condition burden, recent utilisation, and SDOH-derived social risk scores
- Care gap identification mapped to CMS quality programme measures and evidence-based preventive care guidelines
- Care plan creation, task assignment, and progress tracking with multi-disciplinary team access
- Care manager worklist with prioritised patient queue and recommended next actions
- Automated member outreach: SMS, email, and phone with response tracking and care plan update
- SDOH screening data capture (PRAPARE or AHC-HRSN) and referral to community resources
- CMS TCM (99495–99496) and CCM (99490–99491) billing documentation with time tracking
- HL7 FHIR R4 compliant APIs for CMS Interoperability and Prior Authorization Final Rule

**Should-have (v1.1)**:
- AI Copilot for care managers: auto-populates care plan fields, documentation, and protocol checklists from patient record
- HEDIS measure calculation and reporting (NCQA licence required for commercial use)
- Predictive readmission and escalation risk model incorporating SDOH, claims, labs, and care plan adherence
- AI-generated clinical summaries for referral and transitions-of-care packets
- Post-acute referral management: provider network directory with quality metrics, referral tracking, and readmission analytics
- Patient engagement portal with care plan access, self-management resources, and secure messaging

**Nice-to-have (backlog)**:
- Autonomous care gap closure agents: end-to-end gap closure without care manager intervention for low-complexity gaps
- Referral leakage prevention AI: predict abandonment risk and automate scheduling confirmation
- NLP extraction of structured diagnoses and HCC risk adjustment codes from unstructured clinical notes
- Natural language care plan management interface for conversational plan updates and multi-provider reconciliation
- SDOH closed-loop referral tracking: community-based organisation confirmation of resource provision completing the care loop
