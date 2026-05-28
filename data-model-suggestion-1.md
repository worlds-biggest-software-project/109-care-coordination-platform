# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Care Coordination Platform · Created: 2026-05-19

## Philosophy

This model maps every domain concept to its own table with strict foreign keys, constraints, and junction tables. It mirrors the FHIR R4 resource model (CarePlan, CareTeam, Patient, Condition, Goal, ServiceRequest, Task, Observation) as closely as possible while translating FHIR's reference-based linking into relational foreign keys. Every FHIR resource type that the platform must support becomes a first-class table, and FHIR coding/identifier patterns are represented as child tables (e.g., `condition_coding`, `patient_identifier`).

This approach prioritises data integrity, regulatory auditability, and query predictability. Complex cross-entity queries — "find all patients attributed to ACO X whose diabetes care gap is open and who have an unresolved SDOH food-insecurity screening" — are expressed as standard SQL joins without JSONB containment operators or event replay. The schema is self-documenting: a new developer can read the DDL and understand every relationship.

Real-world systems using this pattern include Epic's Chronicles relational core, Oracle Health's Millennium database, and HAPI FHIR's JPA persistence layer (which maps FHIR resources to a normalized relational schema in PostgreSQL or SQL Server).

**Best for:** Organisations that need strict referential integrity, complex multi-table reporting, and direct alignment with FHIR R4 resource definitions for regulatory compliance.

**Trade-offs:**
- (+) Strong referential integrity enforced at the database level
- (+) Direct mapping to FHIR R4 resources simplifies API layer
- (+) Complex cross-entity queries are straightforward SQL joins
- (+) Schema is self-documenting and auditable
- (-) High table count (~60-70 tables) increases migration complexity
- (-) Schema changes for new fields require ALTER TABLE migrations
- (-) Jurisdiction-specific or programme-specific fields require either nullable columns or additional junction tables
- (-) Write performance under high concurrency requires careful indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Each core FHIR resource (Patient, CarePlan, CareTeam, Condition, Goal, Task, ServiceRequest, Observation, Encounter) maps to a dedicated table; FHIR references become foreign keys |
| HL7 US Core STU8 | US Core mandatory data elements (demographics, conditions, medications, labs, vitals, smoking status) are explicit columns, not optional JSONB |
| Gravity SDOH Clinical Care IG | SDOH screenings, conditions, goals, service requests, and tasks modeled as typed rows in their respective tables with SDOH-specific category codes |
| HL7 Da Vinci CDex | Clinical data exchange tracked via `data_exchange_request` and `data_exchange_response` tables |
| CMS-0057-F | Patient Access, Provider Access, Payer-to-Payer, and Prior Auth API compliance supported by the FHIR-aligned table structure |
| USCDI v3/v4 | All USCDI data classes (clinical notes, SDOH, care team, care plan) have dedicated tables |
| NCQA HEDIS | Quality measures, care gaps, and measure results tracked in `quality_measure`, `care_gap`, and `measure_result` tables |
| ANSI X12 837/835 | Claims data ingested into `claim` and `claim_line` tables with X12-aligned fields |
| ISO 3166 | Jurisdiction codes stored as ISO 3166-1/3166-2 in address and organisation tables |
| HIPAA | Audit log table captures all PHI access; encryption metadata tracked per record |

---

## Core Identity & Organisation Tables

```sql
-- ============================================================
-- TENANT & ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL CHECK (type IN ('health_system', 'health_plan', 'aco', 'physician_group', 'cbo')),
    npi             VARCHAR(10),          -- National Provider Identifier (if applicable)
    tax_id          VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES organisation(id),  -- hierarchy: system > hospital > department
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL CHECK (type IN ('health_system', 'hospital', 'clinic', 'practice', 'department', 'cbo', 'snf', 'home_health', 'hospice')),
    npi             VARCHAR(10),
    fhir_id         VARCHAR(64),          -- FHIR Organization resource id
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_code      VARCHAR(2),           -- ISO 3166-2:US
    postal_code     VARCHAR(10),
    country_code    CHAR(2) DEFAULT 'US', -- ISO 3166-1 alpha-2
    phone           VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);
CREATE INDEX idx_organisation_parent ON organisation(parent_id);
CREATE INDEX idx_organisation_npi ON organisation(npi);
CREATE INDEX idx_organisation_type ON organisation(tenant_id, type);
```

## Patient Tables

```sql
CREATE TABLE patient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    fhir_id             VARCHAR(64),
    mrn                 VARCHAR(50),           -- Medical Record Number
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    middle_name         VARCHAR(100),
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),           -- FHIR AdministrativeGender
    sex_at_birth        VARCHAR(20),           -- USCDI v3
    sexual_orientation  VARCHAR(50),           -- USCDI v3
    gender_identity     VARCHAR(50),           -- USCDI v3
    race                VARCHAR(50),           -- OMB race categories
    ethnicity           VARCHAR(50),           -- OMB ethnicity categories
    preferred_language  VARCHAR(10),           -- BCP-47 language tag
    deceased_flag       BOOLEAN DEFAULT false,
    deceased_date       DATE,
    address_line1       VARCHAR(255),
    address_line2       VARCHAR(255),
    city                VARCHAR(100),
    state_code          VARCHAR(2),
    postal_code         VARCHAR(10),
    country_code        CHAR(2) DEFAULT 'US',
    phone_home          VARCHAR(20),
    phone_mobile        VARCHAR(20),
    email               VARCHAR(255),
    primary_pcp_id      UUID,                  -- FK to provider (set after provider table created)
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_tenant ON patient(tenant_id);
CREATE INDEX idx_patient_name ON patient(tenant_id, last_name, first_name);
CREATE INDEX idx_patient_dob ON patient(tenant_id, date_of_birth);
CREATE INDEX idx_patient_mrn ON patient(tenant_id, mrn);

CREATE TABLE patient_identifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id) ON DELETE CASCADE,
    system          VARCHAR(255) NOT NULL,    -- e.g., 'http://hl7.org/fhir/sid/us-medicare'
    value           VARCHAR(100) NOT NULL,
    type_code       VARCHAR(20),              -- MR, SS, MB, etc.
    assigner        VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_identifier_patient ON patient_identifier(patient_id);
CREATE UNIQUE INDEX idx_patient_identifier_system_value ON patient_identifier(system, value);

CREATE TABLE patient_attribution (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    programme_id    UUID NOT NULL,            -- FK to vbc_programme
    payer_id        UUID,                     -- FK to organisation (payer)
    attributed_pcp_id UUID,                   -- FK to provider
    attribution_start DATE NOT NULL,
    attribution_end   DATE,
    attribution_method VARCHAR(50),           -- 'claims_based', 'voluntary', 'prospective'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_attribution_patient ON patient_attribution(patient_id);
CREATE INDEX idx_patient_attribution_programme ON patient_attribution(tenant_id, programme_id);
```

## Provider & Care Team Tables

```sql
CREATE TABLE provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    fhir_id         VARCHAR(64),
    npi             VARCHAR(10) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),             -- MD, DO, NP, RN, LCSW, etc.
    specialty_code  VARCHAR(20),             -- NUCC taxonomy code
    specialty_name  VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provider_tenant ON provider(tenant_id);
CREATE INDEX idx_provider_npi ON provider(npi);
CREATE INDEX idx_provider_org ON provider(organisation_id);

-- Now add FK for patient.primary_pcp_id
ALTER TABLE patient ADD CONSTRAINT fk_patient_pcp FOREIGN KEY (primary_pcp_id) REFERENCES provider(id);

CREATE TABLE care_team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    fhir_id         VARCHAR(64),
    name            VARCHAR(255),
    status          VARCHAR(20) NOT NULL CHECK (status IN ('proposed', 'active', 'suspended', 'inactive', 'entered-in-error')),
    category_code   VARCHAR(50),             -- e.g., 'longitudinal', 'encounter', 'episode'
    period_start    DATE,
    period_end      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_team_patient ON care_team(tenant_id, patient_id);

CREATE TABLE care_team_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    care_team_id    UUID NOT NULL REFERENCES care_team(id) ON DELETE CASCADE,
    provider_id     UUID REFERENCES provider(id),
    patient_id      UUID REFERENCES patient(id),        -- patient can be a member of their own care team
    role_code       VARCHAR(50) NOT NULL,                -- FHIR CareTeam participant role
    role_display    VARCHAR(255),
    period_start    DATE,
    period_end      DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_team_member_team ON care_team_member(care_team_id);
CREATE INDEX idx_care_team_member_provider ON care_team_member(provider_id);
```

## Clinical Data Tables

```sql
CREATE TABLE condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    fhir_id         VARCHAR(64),
    category        VARCHAR(50) NOT NULL,    -- 'problem-list-item', 'encounter-diagnosis', 'sdoh'
    clinical_status VARCHAR(20) NOT NULL CHECK (clinical_status IN ('active', 'recurrence', 'relapse', 'inactive', 'remission', 'resolved')),
    verification_status VARCHAR(20),
    severity_code   VARCHAR(20),
    code_system     VARCHAR(255),            -- e.g., 'http://snomed.info/sct', 'http://hl7.org/fhir/sid/icd-10-cm'
    code_value      VARCHAR(20) NOT NULL,    -- ICD-10-CM or SNOMED CT code
    code_display    VARCHAR(500),
    onset_date      DATE,
    abatement_date  DATE,
    recorded_date   DATE,
    recorder_id     UUID REFERENCES provider(id),
    is_sdoh         BOOLEAN DEFAULT false,   -- Gravity SDOH flag
    sdoh_category   VARCHAR(100),            -- Gravity SDOH category (food_insecurity, housing, transportation, etc.)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_patient ON condition(tenant_id, patient_id);
CREATE INDEX idx_condition_code ON condition(code_system, code_value);
CREATE INDEX idx_condition_sdoh ON condition(tenant_id, is_sdoh) WHERE is_sdoh = true;

CREATE TABLE observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    fhir_id         VARCHAR(64),
    category_code   VARCHAR(50) NOT NULL,    -- 'vital-signs', 'laboratory', 'social-history', 'sdoh', 'survey'
    status          VARCHAR(20) NOT NULL,    -- 'final', 'preliminary', 'amended', 'corrected'
    code_system     VARCHAR(255),
    code_value      VARCHAR(20) NOT NULL,    -- LOINC code
    code_display    VARCHAR(500),
    value_quantity  DECIMAL(18,4),
    value_unit      VARCHAR(50),
    value_string    VARCHAR(1000),
    value_code      VARCHAR(50),
    effective_date  TIMESTAMPTZ,
    issued_date     TIMESTAMPTZ,
    performer_id    UUID REFERENCES provider(id),
    encounter_id    UUID,                    -- FK to encounter
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_observation_patient ON observation(tenant_id, patient_id);
CREATE INDEX idx_observation_code ON observation(code_system, code_value);
CREATE INDEX idx_observation_category ON observation(tenant_id, category_code);
CREATE INDEX idx_observation_date ON observation(tenant_id, patient_id, effective_date);

CREATE TABLE encounter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL,
    class_code      VARCHAR(20) NOT NULL,    -- 'AMB', 'IMP', 'EMER', 'HH', 'VR'
    type_code       VARCHAR(20),
    type_display    VARCHAR(255),
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    reason_code     VARCHAR(20),
    discharge_disposition VARCHAR(50),
    organisation_id UUID REFERENCES organisation(id),
    provider_id     UUID REFERENCES provider(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounter_patient ON encounter(tenant_id, patient_id);
CREATE INDEX idx_encounter_date ON encounter(tenant_id, period_start);
CREATE INDEX idx_encounter_class ON encounter(tenant_id, class_code);

ALTER TABLE observation ADD CONSTRAINT fk_observation_encounter FOREIGN KEY (encounter_id) REFERENCES encounter(id);
```

## Care Plan & Goal Tables

```sql
CREATE TABLE care_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    care_team_id    UUID REFERENCES care_team(id),
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'active', 'on-hold', 'revoked', 'completed', 'entered-in-error', 'unknown')),
    intent          VARCHAR(20) NOT NULL DEFAULT 'plan' CHECK (intent IN ('proposal', 'plan', 'order', 'option')),
    category_code   VARCHAR(50),             -- e.g., 'assess-plan', 'longitudinal', 'encounter'
    title           VARCHAR(500),
    description     TEXT,
    period_start    DATE,
    period_end      DATE,
    author_id       UUID REFERENCES provider(id),
    replaces_id     UUID REFERENCES care_plan(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_plan_patient ON care_plan(tenant_id, patient_id);
CREATE INDEX idx_care_plan_status ON care_plan(tenant_id, status);

CREATE TABLE care_plan_condition (
    care_plan_id    UUID NOT NULL REFERENCES care_plan(id) ON DELETE CASCADE,
    condition_id    UUID NOT NULL REFERENCES condition(id),
    PRIMARY KEY (care_plan_id, condition_id)
);

CREATE TABLE goal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    care_plan_id    UUID REFERENCES care_plan(id),
    fhir_id         VARCHAR(64),
    lifecycle_status VARCHAR(20) NOT NULL CHECK (lifecycle_status IN ('proposed', 'planned', 'accepted', 'active', 'on-hold', 'completed', 'cancelled', 'entered-in-error', 'rejected')),
    achievement_status VARCHAR(20),          -- 'in-progress', 'improving', 'worsening', 'no-change', 'achieved', 'sustaining', 'not-achieved', 'no-progress', 'not-attainable'
    category_code   VARCHAR(50),
    priority        VARCHAR(20),             -- 'high-priority', 'medium-priority', 'low-priority'
    description     TEXT NOT NULL,
    target_date     DATE,
    target_value    DECIMAL(18,4),
    target_unit     VARCHAR(50),
    is_sdoh         BOOLEAN DEFAULT false,
    sdoh_category   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_goal_patient ON goal(tenant_id, patient_id);
CREATE INDEX idx_goal_care_plan ON goal(care_plan_id);

CREATE TABLE care_plan_activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    care_plan_id    UUID NOT NULL REFERENCES care_plan(id) ON DELETE CASCADE,
    activity_type   VARCHAR(50) NOT NULL,    -- 'task', 'service_request', 'medication_request', 'appointment', 'communication'
    reference_id    UUID,                    -- FK to the referenced resource table
    status          VARCHAR(20) NOT NULL,
    description     TEXT,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    performer_id    UUID REFERENCES provider(id),
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_plan_activity_plan ON care_plan_activity(care_plan_id);
```

## Task & Workflow Tables

```sql
CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    care_plan_id    UUID REFERENCES care_plan(id),
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'requested', 'received', 'accepted', 'rejected', 'ready', 'cancelled', 'in-progress', 'on-hold', 'failed', 'completed', 'entered-in-error')),
    intent          VARCHAR(20) NOT NULL DEFAULT 'order',
    priority        VARCHAR(20) DEFAULT 'routine' CHECK (priority IN ('routine', 'urgent', 'asap', 'stat')),
    task_type       VARCHAR(50) NOT NULL,    -- 'outreach_call', 'care_gap_closure', 'sdoh_referral', 'med_reconciliation', 'follow_up', 'documentation', 'review'
    description     TEXT NOT NULL,
    requester_id    UUID REFERENCES provider(id),
    owner_id        UUID REFERENCES provider(id),      -- assigned care manager
    due_date        DATE,
    completed_date  TIMESTAMPTZ,
    time_spent_min  INTEGER DEFAULT 0,       -- for CCM/TCM billing
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_patient ON task(tenant_id, patient_id);
CREATE INDEX idx_task_owner ON task(tenant_id, owner_id, status);
CREATE INDEX idx_task_due ON task(tenant_id, due_date) WHERE status NOT IN ('completed', 'cancelled', 'failed');
CREATE INDEX idx_task_type ON task(tenant_id, task_type, status);
```

## SDOH & Referral Tables

```sql
CREATE TABLE sdoh_screening (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    encounter_id    UUID REFERENCES encounter(id),
    instrument      VARCHAR(50) NOT NULL,    -- 'PRAPARE', 'AHC-HRSN', 'custom'
    screening_date  TIMESTAMPTZ NOT NULL,
    screener_id     UUID REFERENCES provider(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sdoh_screening_patient ON sdoh_screening(tenant_id, patient_id);

CREATE TABLE sdoh_screening_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_id    UUID NOT NULL REFERENCES sdoh_screening(id) ON DELETE CASCADE,
    question_code   VARCHAR(20) NOT NULL,    -- LOINC code for the question
    question_text   VARCHAR(500),
    answer_code     VARCHAR(20),             -- LOINC answer code
    answer_text     VARCHAR(500),
    risk_flag       BOOLEAN DEFAULT false,
    sdoh_category   VARCHAR(100),            -- Gravity category: food_insecurity, housing_instability, transportation, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sdoh_response_screening ON sdoh_screening_response(screening_id);

CREATE TABLE referral (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    care_plan_id    UUID REFERENCES care_plan(id),
    fhir_id         VARCHAR(64),
    referral_type   VARCHAR(50) NOT NULL,    -- 'specialty', 'post_acute', 'sdoh_cbo', 'behavioral_health', 'home_health'
    status          VARCHAR(20) NOT NULL CHECK (status IN ('draft', 'active', 'on-hold', 'revoked', 'completed', 'entered-in-error')),
    priority        VARCHAR(20) DEFAULT 'routine',
    reason_code     VARCHAR(20),
    reason_display  VARCHAR(500),
    referring_provider_id   UUID NOT NULL REFERENCES provider(id),
    receiving_org_id        UUID REFERENCES organisation(id),
    receiving_provider_id   UUID REFERENCES provider(id),
    requested_date  DATE NOT NULL,
    scheduled_date  DATE,
    completed_date  DATE,
    clinical_summary_doc_id UUID,            -- FK to document
    abandonment_risk_score  DECIMAL(5,4),    -- ML-predicted risk 0.0-1.0
    sdoh_category   VARCHAR(100),            -- if SDOH referral, Gravity category
    cbo_confirmation_date   DATE,            -- closed-loop: CBO confirmed service delivery
    cbo_confirmation_notes  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_referral_patient ON referral(tenant_id, patient_id);
CREATE INDEX idx_referral_status ON referral(tenant_id, status);
CREATE INDEX idx_referral_type ON referral(tenant_id, referral_type, status);
CREATE INDEX idx_referral_receiving_org ON referral(receiving_org_id);
```

## Quality Measure & Care Gap Tables

```sql
CREATE TABLE quality_measure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    measure_id_ext  VARCHAR(50) NOT NULL,    -- e.g., 'NQF-0059', 'HEDIS-CDC'
    measure_name    VARCHAR(500) NOT NULL,
    measure_set     VARCHAR(50) NOT NULL,    -- 'HEDIS', 'CMS_STARS', 'MIPS', 'CUSTOM'
    domain          VARCHAR(100),            -- 'effectiveness_of_care', 'access_availability', etc.
    description     TEXT,
    numerator_desc  TEXT,
    denominator_desc TEXT,
    measure_year    INTEGER NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quality_measure_set ON quality_measure(tenant_id, measure_set, measure_year);

CREATE TABLE care_gap (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    measure_id      UUID NOT NULL REFERENCES quality_measure(id),
    gap_status      VARCHAR(20) NOT NULL CHECK (gap_status IN ('open', 'pending_closure', 'closed', 'excluded', 'not_applicable')),
    identified_date DATE NOT NULL,
    due_date        DATE,
    closed_date     DATE,
    closure_method  VARCHAR(50),             -- 'clinical_action', 'outreach', 'ai_agent', 'supplemental_data'
    closure_evidence_type VARCHAR(50),       -- 'claim', 'ehr_data', 'patient_report', 'lab_result'
    assigned_to_id  UUID REFERENCES provider(id),
    care_plan_id    UUID REFERENCES care_plan(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_gap_patient ON care_gap(tenant_id, patient_id);
CREATE INDEX idx_care_gap_status ON care_gap(tenant_id, gap_status);
CREATE INDEX idx_care_gap_measure ON care_gap(measure_id, gap_status);
CREATE INDEX idx_care_gap_assigned ON care_gap(tenant_id, assigned_to_id, gap_status);

CREATE TABLE measure_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    measure_id      UUID NOT NULL REFERENCES quality_measure(id),
    reporting_period_start DATE NOT NULL,
    reporting_period_end   DATE NOT NULL,
    organisation_id UUID REFERENCES organisation(id),
    provider_id     UUID REFERENCES provider(id),
    denominator_count INTEGER NOT NULL,
    numerator_count   INTEGER NOT NULL,
    exclusion_count   INTEGER DEFAULT 0,
    rate            DECIMAL(7,4),            -- percentage
    benchmark_50th  DECIMAL(7,4),
    benchmark_90th  DECIMAL(7,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_measure_result_period ON measure_result(tenant_id, measure_id, reporting_period_start);
```

## Claims & Billing Tables

```sql
CREATE TABLE claim (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    claim_number    VARCHAR(50) NOT NULL,
    claim_type      VARCHAR(20) NOT NULL,    -- 'professional', 'institutional', 'dental'  (X12 837P/837I)
    status          VARCHAR(20) NOT NULL,    -- 'submitted', 'accepted', 'denied', 'paid', 'adjusted'
    payer_id        UUID REFERENCES organisation(id),
    provider_id     UUID REFERENCES provider(id),
    facility_id     UUID REFERENCES organisation(id),
    service_date_from DATE NOT NULL,
    service_date_to   DATE,
    billed_amount   DECIMAL(12,2),
    allowed_amount  DECIMAL(12,2),
    paid_amount     DECIMAL(12,2),
    primary_dx_code VARCHAR(10),             -- ICD-10-CM
    admission_date  DATE,
    discharge_date  DATE,
    drg_code        VARCHAR(5),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_patient ON claim(tenant_id, patient_id);
CREATE INDEX idx_claim_date ON claim(tenant_id, service_date_from);
CREATE INDEX idx_claim_payer ON claim(tenant_id, payer_id);

CREATE TABLE claim_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claim(id) ON DELETE CASCADE,
    line_number     INTEGER NOT NULL,
    cpt_code        VARCHAR(10),
    hcpcs_code      VARCHAR(10),
    modifier1       VARCHAR(2),
    modifier2       VARCHAR(2),
    dx_pointer      VARCHAR(10),
    units           DECIMAL(8,2),
    billed_amount   DECIMAL(12,2),
    allowed_amount  DECIMAL(12,2),
    paid_amount     DECIMAL(12,2),
    place_of_service VARCHAR(2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_line_claim ON claim_line(claim_id);
CREATE INDEX idx_claim_line_cpt ON claim_line(cpt_code);
```

## Risk Stratification & Outreach Tables

```sql
CREATE TABLE risk_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    score_type      VARCHAR(50) NOT NULL,    -- 'composite', 'readmission_30d', 'ed_utilisation', 'hcc_raf', 'sdoh_risk'
    score_value     DECIMAL(10,4) NOT NULL,
    risk_tier       VARCHAR(20),             -- 'low', 'rising', 'moderate', 'high', 'very_high'
    model_version   VARCHAR(50),
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_through   DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_score_patient ON risk_score(tenant_id, patient_id, score_type);
CREATE INDEX idx_risk_score_tier ON risk_score(tenant_id, risk_tier, score_type);

CREATE TABLE outreach (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    task_id         UUID REFERENCES task(id),
    care_gap_id     UUID REFERENCES care_gap(id),
    channel         VARCHAR(20) NOT NULL,    -- 'phone', 'sms', 'email', 'portal', 'mail'
    direction       VARCHAR(10) NOT NULL,    -- 'outbound', 'inbound'
    status          VARCHAR(20) NOT NULL,    -- 'scheduled', 'attempted', 'reached', 'completed', 'failed', 'no_answer'
    attempt_number  INTEGER DEFAULT 1,
    scheduled_at    TIMESTAMPTZ,
    attempted_at    TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    duration_sec    INTEGER,
    notes           TEXT,
    performed_by_id UUID REFERENCES provider(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outreach_patient ON outreach(tenant_id, patient_id);
CREATE INDEX idx_outreach_status ON outreach(tenant_id, status);
CREATE INDEX idx_outreach_care_gap ON outreach(care_gap_id);
```

## Value-Based Care Programme & Billing Tables

```sql
CREATE TABLE vbc_programme (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    programme_type  VARCHAR(50) NOT NULL,    -- 'mssp_aco', 'reach_aco', 'ma_vbc', 'medicaid_managed', 'commercial_vbc', 'bundled_payment'
    payer_id        UUID REFERENCES organisation(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vbc_programme_tenant ON vbc_programme(tenant_id);

ALTER TABLE patient_attribution ADD CONSTRAINT fk_attribution_programme FOREIGN KEY (programme_id) REFERENCES vbc_programme(id);

CREATE TABLE ccm_tcm_billing_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    billing_type    VARCHAR(10) NOT NULL CHECK (billing_type IN ('CCM', 'TCM')),
    cpt_code        VARCHAR(10) NOT NULL,    -- 99490, 99491, 99495, 99496
    service_month   DATE NOT NULL,           -- first of month for CCM; discharge date for TCM
    total_time_min  INTEGER NOT NULL,
    provider_id     UUID NOT NULL REFERENCES provider(id),
    claim_submitted BOOLEAN DEFAULT false,
    claim_id        UUID REFERENCES claim(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ccm_tcm_patient ON ccm_tcm_billing_log(tenant_id, patient_id, service_month);
```

## Audit & Security Tables

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,    -- 'create', 'read', 'update', 'delete', 'export', 'login'
    resource_type   VARCHAR(50) NOT NULL,    -- table name / FHIR resource type
    resource_id     UUID,
    patient_id      UUID,                    -- for PHI access tracking
    ip_address      INET,
    user_agent      VARCHAR(500),
    details         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_log_patient ON audit_log(patient_id, created_at);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);

-- Partition audit_log by month for performance
-- CREATE TABLE audit_log (...) PARTITION BY RANGE (created_at);
```

## RBAC Tables

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider_id     UUID REFERENCES provider(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_app_user_email ON app_user(email);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenant(id),          -- NULL = global role
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource_type   VARCHAR(50) NOT NULL,
    action          VARCHAR(20) NOT NULL,                -- 'create', 'read', 'update', 'delete', 'export'
    description     TEXT,
    UNIQUE (resource_type, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    organisation_id UUID REFERENCES organisation(id),    -- scope role to specific org
    PRIMARY KEY (user_id, role_id, organisation_id)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organisation | 2 | Multi-tenant with org hierarchy |
| Patient & Attribution | 3 | Patient demographics, identifiers, programme attribution |
| Provider & Care Team | 3 | Providers, care teams, team members |
| Clinical Data | 3 | Conditions, observations, encounters |
| Care Plan & Goals | 4 | Care plans, plan-condition links, goals, plan activities |
| Tasks & Workflow | 1 | Unified task table for all workflow types |
| SDOH & Referrals | 3 | Screenings, screening responses, referrals |
| Quality & Care Gaps | 3 | Measures, gaps, measure results |
| Claims & Billing | 2 | Claims and claim lines |
| Risk & Outreach | 2 | Risk scores, outreach tracking |
| VBC & Billing | 2 | Programmes, CCM/TCM billing logs |
| Audit & Security | 1 | Immutable audit log |
| RBAC | 4 | Users, roles, permissions, assignments |
| **Total** | **33** | Core tables; ~50-60 with future additions (medications, allergies, documents, notifications) |

---

## Key Design Decisions

1. **Tenant-scoped row-level security** — Every clinical table includes `tenant_id` as the first column in composite indexes. PostgreSQL RLS policies should enforce `tenant_id = current_setting('app.tenant_id')` on all queries, providing database-enforced multi-tenancy without schema-per-tenant overhead.

2. **FHIR resource alignment** — Core tables (patient, condition, observation, care_plan, care_team, goal, task, encounter) map 1:1 to FHIR R4 resources. The `fhir_id` column on each table stores the FHIR resource identifier, enabling bidirectional sync between the relational database and a FHIR server (HAPI FHIR).

3. **Explicit SDOH columns** — Rather than relying on generic category codes alone, SDOH-related tables include explicit `is_sdoh` and `sdoh_category` columns to support the Gravity SDOH Clinical Care IG closed-loop referral pattern. This makes SDOH queries fast and explicit.

4. **Care gap as first-class entity** — Care gaps are modeled as their own table rather than computed views, because care gap lifecycle management (identification, assignment, outreach, closure) is a core workflow. The `closure_method` column tracks whether AI agents or human care managers closed the gap.

5. **Referral with abandonment risk** — The referral table includes `abandonment_risk_score` to support the AI-driven referral leakage prevention feature. The closed-loop SDOH fields (`cbo_confirmation_date`, `cbo_confirmation_notes`) are on the same table rather than a separate CBO interaction table, keeping the referral lifecycle in one place.

6. **CCM/TCM billing time tracking** — Dedicated `ccm_tcm_billing_log` table tracks cumulative time per patient per month for CMS CCM (99490-99491) and TCM (99495-99496) billing codes, as these have specific documentation requirements for reimbursement.

7. **Audit log designed for HIPAA** — The audit log includes `patient_id` to track all PHI access, meeting HIPAA access logging requirements. It should be partitioned by month for performance and retention management.

8. **Organisation hierarchy via self-referencing FK** — `organisation.parent_id` supports health system hierarchies (system > hospital > department) using adjacency list pattern. For deep hierarchies, a materialised path or recursive CTE can be added.
