# Standards & API Reference

> Project: Care Coordination Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### HL7 FHIR Standards

**HL7 FHIR R4 (Release 4.0.1)**
- URL: https://www.hl7.org/fhir/R4/
- The foundational interoperability standard mandated across U.S. healthcare by CMS. All APIs required under CMS-0057-F (Interoperability and Prior Authorization Final Rule) must use FHIR R4. Care coordination platforms must expose CarePlan, CareTeam, Patient, Observation, Condition, Encounter, and related resources via FHIR R4 REST APIs. Normative content with backward-compatibility guarantees; the standard for all new integration development as of 2026.

**HL7 US Core Implementation Guide (STU8 / v8.0.0)**
- URL: https://hl7.org/fhir/us/core/
- Defines minimum FHIR R4 profiles and constraints for U.S.-specific interoperability, including mandatory data elements for patient demographics, clinical conditions, medications, laboratory results, vital signs, and smoking status. US Core is the baseline profile set required by ONC Health IT Certification and is the conformance target for CMS-mandated APIs. Required by all Da Vinci implementation guides.

**HL7 FHIR R6 (Build / Ballot)**
- URL: https://build.fhir.org/
- The in-development next-generation FHIR release. Platforms should monitor ballot progress; R4 remains the required production version through at least 2027.

### HL7 Da Vinci Project Implementation Guides

**Da Vinci Clinical Data Exchange (CDex) v2.1.0**
- URL: https://hl7.org/fhir/us/davinci-cdex/STU2/
- Defines FHIR-based interactions for provider-to-payer and provider-to-provider clinical data exchange, including Direct Query, Task-Based, and Attachments transaction patterns. Directly relevant to care coordination for claims management, prior authorization support, quality reporting, and care gap documentation. Required under CMS-0057-F for payer clinical data request workflows.

**Da Vinci Documentation Templates and Rules (DTR) v2.2.0**
- URL: https://build.fhir.org/ig/HL7/davinci-dtr/
- Specifies how payer documentation requirements can be expressed as FHIR Questionnaires and executed within a provider EHR context using Clinical Quality Language (CQL). Enables automated pre-population of documentation fields from EHR data — directly enabling the AI Copilot feature pattern (auto-populating care plan and protocol fields) from existing patient records.

**Da Vinci Prior Authorization Support (PAS) v2.1.0**
- URL: https://hl7.org/fhir/us/davinci-pas/
- Defines FHIR-based services for submitting prior authorization requests and receiving real-time responses from payers at the point of service. Complements CDex for utilization management workflows integrated within care coordination platforms. Bridges FHIR and ANSI X12 278 transaction sets via intermediary mapping.

**Gravity Project SDOH Clinical Care IG (HL7 FHIR SDOH IG)**
- URL: https://hl7.org/fhir/us/sdoh-clinicalcare/
- GitHub: https://github.com/HL7/fhir-sdoh-clinicalcare
- Defines how to exchange coded social determinants of health (SDOH) data, including screening assessments, SDOH diagnoses (Conditions), goals, referral requests (ServiceRequests), and tasks to community-based organisations. Supports the closed-loop SDOH referral pattern (screening → referral → community organisation confirmation) identified as an underserved opportunity. Recognised as a core interoperability standard by multiple state health IT programmes in 2025–2026.

**HL7 Da Vinci Data Exchange for Quality Measures (DEQM)**
- URL: https://blog.hl7.org/topic/deqm
- Defines how FHIR-based quality measure data is collected and reported for CMS value-based care programmes and HEDIS digital quality measures (dQMs). Relevant for care coordination platforms that need to report HEDIS and CMS quality measure performance.

### Clinical Document Standards

**HL7 C-CDA (Consolidated Clinical Document Architecture) R2.1 / R3.0**
- URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=492
- C-CDA R3.0 value sets: https://www.nlm.nih.gov/vsac/support/releasenotes/20240809.html
- XML document standard for structured clinical summaries including Continuity of Care Documents (CCDs), Referral Notes, and Discharge Summaries. Required for Promoting Interoperability (Meaningful Use) compliance. C-CDA R3.0 merges R2.1 and Companion Guides, adds USCDI v4 alignment. Critical for care transitions use cases — AI-generated discharge summaries must produce valid C-CDA or FHIR Document bundles accepted by receiving EHRs.

### ONC & CMS Regulatory Standards

**ONC United States Core Data for Interoperability (USCDI) v3 / v4**
- URL: https://isp.healthit.gov/united-states-core-data-interoperability-uscdi
- USCDI v3: https://www.healthit.gov/sites/default/files/facas/2022-08-17_USCDI_v3_Presentation_508.pdf
- Defines the minimum dataset of health data classes and data elements required for nationwide interoperable health information exchange. USCDI v3 becomes mandatory under HTI-1 Final Rule on January 1, 2026. Care coordination platforms must support all USCDI v3 data elements including social determinants of health screening data, sexual orientation and gender identity, and care team members. USCDI v4 adds care planning and medication reconciliation data elements, relevant to longitudinal care plan features.

**CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F)**
- URL: https://www.cms.gov/cms-interoperability-and-prior-authorization-final-rule-cms-0057-f
- Requires impacted payers (Medicare Advantage, Medicaid, CHIP, QHP) to implement and maintain four FHIR R4 APIs: Patient Access API, Provider Access API, Payer-to-Payer API, and Prior Authorization API. Full compliance deadline is January 1, 2027. Care coordination platforms targeting payer clients must comply with all four APIs; platforms targeting providers must integrate with payer APIs to exchange patient clinical history and prior authorization status.

**CMS APIs and Implementation Guides Index**
- URL: https://www.cms.gov/priorities/burden-reduction/overview/interoperability/implementation-guides-standards/application-programming-interfaces-apis-relevant-standards-implementation-guides-igs
- Official CMS index of all mandatory and recommended standards and IGs for CMS-0057-F compliance; authoritative reference for all required API standards.

### Security & Authentication Standards

**SMART App Launch v2.2.0 (SMART on FHIR)**
- URL: https://hl7.org/fhir/smart-app-launch/
- Build: https://build.fhir.org/ig/HL7/smart-app-launch/
- Defines OAuth 2.0 based authorization patterns for FHIR client applications: EHR launch (context-aware app launch within a clinician session), standalone launch, and backend service authorization (for headless system-to-system integrations without a user present). All EHR-embedded care coordination modules and API integrations with Epic, Oracle Health, and other EHRs require SMART App Launch compliance. Supports PKCE (RFC 7636) for security hardening.

**HIPAA Privacy Rule and Security Rule (45 CFR Parts 160, 162, and 164)**
- Privacy Rule summary: https://www.hhs.gov/hipaa/for-professionals/privacy/laws-regulations/index.html
- Security Rule summary: https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- Proposed 2025 Security Rule NPRM: https://www.federalregister.gov/documents/2025/01/06/2024-30983/
- Mandatory compliance framework for all U.S. healthcare software handling protected health information (PHI). The proposed 2025 HIPAA Security Rule NPRM (published January 2025) would eliminate the distinction between "required" and "addressable" safeguards, mandate encryption of ePHI at rest and in transit, require multi-factor authentication, biannual vulnerability scans, and annual penetration testing. Care coordination platforms are Business Associates and must execute BAAs with all covered entity clients.

### EDI & Claims Standards

**ANSI X12 Healthcare Transaction Standards (HIPAA EDI)**
- URL: https://x12.org/flow/health-care
- Mandatory HIPAA EDI transaction sets for claims ingestion and administrative workflows:
  - **X12 837** (Professional / Institutional / Dental): Health care claim submission; primary source for claims-based risk stratification and utilisation data ingestion
  - **X12 835**: Health care claim payment and remittance advice; needed for financial outcome tracking
  - **X12 278**: Health care services review (prior authorisation request and notification); bridged to FHIR via Da Vinci PAS IG
  - **X12 270/271**: Eligibility inquiry and response; used for real-time member eligibility verification at care coordination touchpoints
- All covered entities required to use HIPAA 5010A1 version of X12 transactions.

### Quality Measurement Standards

**NCQA HEDIS (Healthcare Effectiveness Data and Information Set)**
- URL: https://www.ncqa.org/hedis/
- Digital measures: https://www.ncqa.org/hedis/the-future-of-hedis/digital-measures/
- The dominant health plan quality measurement framework, covering 90+ measures across six domains. Care coordination directly impacts measures including Transitions of Care (TRC), Follow-Up After Hospitalization for Mental Illness (FUH), Follow-Up After Emergency Department Visit, and Comprehensive Diabetes Care. NCQA is transitioning HEDIS to FHIR-based digital quality measures (dQMs) delivered as computable FHIR packages. Commercial platforms calculating and reporting HEDIS measures require a formal NCQA licence.

**NCQA Case Management Accreditation Standards**
- URL: https://www.ncqa.org/programs/health-plans/case-management-cm/
- Standards framework covering identification and assessment, care planning, care monitoring, and care coordination for organisations providing case management to complex or high-risk populations. Relevant for care coordination platform workflow design and documentation requirements; health plan clients may require the platform to support NCQA CM accreditation workflows.

---

## Similar Products — Developer Documentation & APIs

### Epic on FHIR / open.epic

- **Description:** Epic is the most widely deployed EHR in the U.S. (>250 million patients). Epic's open.epic programme provides FHIR R4 APIs and developer resources for building applications that integrate with Epic EHR data and workflows. Essential integration target for care coordination platforms deployed at health systems.
- **API Documentation:** https://fhir.epic.com/Documentation
- **Technical Specifications:** https://open.epic.com/TechnicalSpecifications
- **Developer Resources:** https://open.epic.com/DeveloperResources
- **Specifications Index:** https://fhir.epic.com/Specifications
- **SDKs/Libraries:** No official SDK; FHIR R4 REST APIs consumed via standard HTTP clients; HAPI FHIR client library (Java) widely used
- **Standards:** FHIR R4 (US Core), SMART App Launch v2, USCDI, CMS-0057-F Patient Access/Provider Access/Payer-to-Payer APIs
- **Authentication:** SMART App Launch (OAuth 2.0), Backend Services (JWT client credentials)
- **Notes:** 40+ developer playbooks guide specific care coordination integration patterns (care team management, transitions of care, referral workflows). International Patient Summary (IPS) support added May 2025.

### Oracle Health (Cerner) Millennium Platform FHIR APIs

- **Description:** Oracle Health Millennium is the second-largest EHR platform in the U.S. The Millennium Platform provides FHIR R4 (Ignite APIs), HL7 v2, and CDS Hooks integration pathways. Priority integration for care coordination platforms targeting hospital, post-acute, and government health system deployments.
- **API Documentation:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/mfrap/r4_overview.html
- **Developer Portal:** https://docs.oracle.com/en/industries/health/millennium-platform-apis/index.html
- **Health Data Intelligence Docs:** https://docs.healtheintent.com/
- **SDKs/Libraries:** No official SDK; FHIR R4 REST via standard HTTP; HAPI FHIR client library used in practice
- **Standards:** FHIR R4 (Ignite APIs), SMART App Launch, CDS Hooks, HL7 v2 (COI), C-CDA
- **Authentication:** SMART App Launch (OAuth 2.0), M2M OAuth (backend services)
- **Notes:** New integrations in 2026 recommended to use FHIR R4 Ignite APIs as the primary method.

### Salesforce Health Cloud

- **Description:** CRM-based care coordination platform with FHIR R4 APIs and an extensive developer ecosystem. Health Cloud exposes clinical data as FHIR-compliant REST APIs alongside native Salesforce REST/SOAP APIs and platform customisation tooling. Key integration target for care coordination solutions building on or alongside Salesforce.
- **API Documentation (FHIR):** https://developer.salesforce.com/docs/atlas.en-us.health_cloud_object_reference.meta/health_cloud_object_reference/hc_fhir_apis_overview.htm
- **Developer Overview:** https://developer.salesforce.com/docs/industries/health/overview
- **REST Reference:** https://developer.salesforce.com/docs/atlas.en-us.health_cloud_object_reference.meta/health_cloud_object_reference/hc_business_apis_rest_reference.htm
- **Get Started:** https://developer.salesforce.com/docs/industries/health/guide/get-started.html
- **SDKs/Libraries:** Salesforce SDKs (JavaScript, Java, Python, Apex); AppExchange ecosystem; MuleSoft for complex integrations
- **Standards:** FHIR R4 (aligned clinical data model), REST/JSON, SMART on FHIR, CMS-0057-F compliance for payer use cases, Da Vinci CDex
- **Authentication:** OAuth 2.0 (Salesforce identity), SMART on FHIR for external FHIR API access

### Arcadia Population Health Platform

- **Description:** Population health analytics and care coordination platform for ACOs, health systems, and payers. Arcadia's Plug API and Signal API provide access to population health data, care gap analytics, and real-time data streams. Key reference for multi-payer claims and clinical data aggregation architecture.
- **API Documentation:** https://docs.arcadia.com/
- **Developer Resources:** https://dev.arcadia.io/resources
- **Standards:** FHIR R4 (for data ingestion and interoperability), ANSI X12 (claims ingestion), REST/JSON
- **Authentication:** API key / OAuth 2.0
- **Notes:** Arcadia announced FHIR-based cancer data-sharing API (2024) and AWS HealthLake integration using FHIR adapters for Beneficiary Claims Data (BCDA). New interoperability commitments in 2024 focused on faster standards-based data sharing.

### HAPI FHIR (Open Source FHIR Server and Client Library)

- **Description:** The most widely used open-source FHIR server and Java client library, licensed under Apache 2.0. HAPI FHIR provides a complete FHIR R4 server (JPA Server) and client API for building FHIR-compliant healthcare applications. The JPA Server starter project is the recommended foundation for building an open-source FHIR backend for a care coordination platform.
- **Official Website:** https://hapifhir.io/
- **API Documentation:** https://hapifhir.io/hapi-fhir/docs/server_jpa/get_started.html
- **GitHub (main):** https://github.com/hapifhir/hapi-fhir
- **GitHub (JPA Server Starter):** https://github.com/hapifhir/hapi-fhir-jpaserver-starter
- **SDKs/Libraries:** Java (primary); community ports in Python (fhirclient), JavaScript (fhir.js), .NET (Firely SDK)
- **Standards:** FHIR R4 (normative), SMART App Launch, US Core profiles
- **Authentication:** SMART on FHIR (OAuth 2.0); configurable interceptor-based auth
- **Licence:** Apache Software License 2.0 — fully embeddable in open-source and commercial products without IP concerns

### Google Cloud Healthcare API

- **Description:** Fully managed, HIPAA-compliant cloud service for ingesting, transforming, and storing healthcare data in FHIR (R4, STU3, DSTU2), HL7v2, and DICOM formats. Provides highly performant FHIR R4 REST APIs with SMART on FHIR authorization, bulk export, and native integration with Google Cloud AI/ML services. Relevant infrastructure layer for cloud-native care coordination platform deployments.
- **API Documentation:** https://docs.cloud.google.com/healthcare-api/docs/concepts/fhir
- **Overview:** https://docs.cloud.google.com/healthcare-api/docs/introduction
- **SMART on FHIR guide:** https://docs.cloud.google.com/healthcare-api/docs/smart-on-fhir
- **SDKs/Libraries:** Google Cloud client libraries (Python, Java, Go, Node.js, Ruby, PHP, C#); REST API; gRPC
- **Standards:** FHIR R4/STU3/DSTU2, HL7v2, DICOM, SMART App Launch, USCDI
- **Authentication:** Google Cloud IAM, SMART on FHIR (OAuth 2.0)
- **Notes:** Python code samples available on GitHub (GoogleCloudPlatform/python-docs-samples). HIPAA-compliant with BAA available from Google Cloud.

### AWS HealthLake

- **Description:** AWS managed FHIR R4 data store and analytics service. Provides FHIR R4 RESTful APIs, bulk import/export, SMART on FHIR authorization, and native integration with Amazon SageMaker for ML-based care management analytics. Relevant as cloud infrastructure for care coordination platforms built on AWS.
- **API Documentation:** https://docs.aws.amazon.com/healthlake/latest/devguide/managing-fhir-resources.html
- **Getting Started:** https://docs.aws.amazon.com/healthlake/latest/devguide/getting-started.html
- **FHIR Operations:** https://docs.aws.amazon.com/healthlake/latest/devguide/reference-fhir-operations.html
- **SMART on FHIR:** https://docs.aws.amazon.com/healthlake/latest/devguide/reference-smart-on-fhir-getting-started.html
- **GitHub Samples:** https://github.com/aws-samples/aws-healthlake-smart-on-fhir
- **SDKs/Libraries:** AWS SDKs (Python/boto3, Java, JavaScript, .NET, Go, Ruby, PHP); FHIR REST API
- **Standards:** FHIR R4, SMART App Launch, US Core profiles, USCDI
- **Authentication:** SMART on FHIR (OAuth 2.0), AWS IAM
- **Notes:** Integrates with Arcadia for BCDA (Beneficiary Claims Data) ingestion using FHIR adapters. HIPAA-eligible with BAA available from AWS.

### Innovaccer Developer Portal

- **Description:** Innovaccer's Healthcare Intelligence Cloud (FHIR+ Data Activation Platform) offers a developer portal and marketplace for building interoperable digital health applications on top of its unified patient data layer. Relevant as a reference architecture for AI-native care coordination API design.
- **Developer Portal:** https://nucleus.innovaccer.com/devPortal/documentation/api-overview
- **Platform Overview:** https://innovaccer.com/data-activation-platform
- **Standards:** FHIR R4, SMART App Launch, US Core, CMS-0057-F, ANSI X12
- **Authentication:** OAuth 2.0
- **Notes:** FHIR+ data model (Level 4) includes approximately 70 entities and 2,800 data elements in a graph structure optimised for high-performance queries. AI-enabled API framework uses NLP to map between disparate API documentation schemas.

---

## Notes

**Evolving Standard: FHIR R5 / R6 Transition**
FHIR R5 was published in 2023 and FHIR R6 is in active ballot as of 2026. However, CMS-0057-F mandates FHIR R4.0.1 as the required version through the 2027 compliance deadlines. New platforms should build on R4 with an architecture that supports profile-based extensibility to ease future migration.

**USCDI v3 Mandatory in 2026**
USCDI v3 becomes the required minimum dataset for ONC-certified health IT under the HTI-1 Final Rule effective January 1, 2026. Care coordination platforms certified or integrated with certified EHR technology must support all USCDI v3 data elements, including expanded SDOH and equity-related data elements.

**HEDIS Digital Quality Measures (dQMs)**
NCQA is actively transitioning HEDIS to FHIR-based digital quality measures published as computable FHIR packages. Most HEDIS measures are now available in dQM format. Platforms calculating HEDIS measures should align with NCQA's dQM packages and the Da Vinci DEQM IG for reporting.

**HIPAA Security Rule NPRM (2025)**
The proposed HIPAA Security Rule update (NPRM published January 2025) would significantly strengthen mandatory technical controls for all platforms handling ePHI. Final rule status is pending as of mid-2026. Platforms should design to the proposed requirements (encryption at rest and in transit, MFA, penetration testing, incident response within 72 hours) regardless of final rule status, as these represent security best practice and are increasingly required by enterprise health system clients contractually.

**Gravity SDOH IG — Growing State Adoption**
Washington State named the Gravity SDOH Clinical Care IG as a core interoperability standard in its 2025 FHIR Infrastructure Roadmap. Multiple states are mandating SDOH closed-loop referral workflows using the Gravity IG. Care coordination platforms should implement the full Gravity referral task pattern (not just SDOH screening) to support state-mandated workflows and the closed-loop SDOH referral opportunity.
