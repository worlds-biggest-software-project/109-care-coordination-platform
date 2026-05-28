# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Care Coordination Platform · Created: 2026-05-19

## Philosophy

This model uses relational tables with typed columns for core, universally-required fields (patient demographics, care plan status, care gap lifecycle) while storing variable, jurisdiction-specific, programme-specific, and extensible data in JSONB columns. The result is a moderate table count with maximum flexibility: new fields for a Medicaid managed care programme in Texas do not require ALTER TABLE — they are added to the JSONB `extensions` column and validated at the application layer.

This approach is inspired by how FHIR itself handles extensibility. FHIR resources have a core set of defined fields plus an `extension` array for profile-specific additions. Similarly, this model has typed relational columns for the FHIR "core" and JSONB for the "extensions." PostgreSQL's JSONB support — including GIN indexes, containment queries (`@>`), and jsonpath — makes this pattern performant for the query patterns care coordination platforms need.

The hybrid pattern is battle-tested in healthcare SaaS. Innovaccer's FHIR+ Data Activation Platform stores approximately 70 entities with 2,800 data elements, many of which vary by customer deployment. Salesforce Health Cloud uses a similar pattern with its metadata-driven object model. This approach lets a single codebase serve ACOs, health plans, health systems, and physician groups — each with different programme requirements — without schema fragmentation.

**Best for:** Multi-customer SaaS deployments where different clients (health plans, ACOs, health systems) require different fields, programmes, and workflows but share a common core data model.

**Trade-offs:**
- (+) Rapid development — new fields don't require migrations
- (+) Multi-programme flexibility — each VBC programme can define custom fields without schema changes
- (+) Moderate table count (~25-30 tables) — easier to understand and maintain than fully normalised
- (+) FHIR extension pattern alignment — natural mapping to FHIR's extensibility model
- (+) GIN-indexed JSONB queries are fast for containment and key-value lookups
- (-) JSONB fields are not self-documenting — schema knowledge lives in application code, not DDL
- (-) No foreign key constraints within JSONB — referential integrity for extended fields is application-enforced
- (-) Complex JSONB queries (deep nesting, array element matching) can be slower than typed column queries
- (-) Data validation for JSONB fields requires application-layer JSON Schema validation
- (-) Reporting/analytics queries on JSONB fields require extraction functions, adding complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Core FHIR fields are relational columns; FHIR extensions map to JSONB `extensions` column; FHIR coding patterns stored as JSONB arrays |
| HL7 US Core STU8 | US Core mandatory elements are typed columns; US Core extensions (race/ethnicity detail) are in JSONB |
| Gravity SDOH IG | SDOH screening responses stored as JSONB arrays within the screening record; SDOH categories are typed columns for indexed querying |
| USCDI v3/v4 | USCDI-mandated elements are typed columns; USCDI v4 additions (care planning extensions) stored in JSONB until promoted |
| NCQA HEDIS | Measure definitions include JSONB `criteria` for measure-specific numerator/denominator logic |
| CMS-0057-F | FHIR API responses assembled from typed columns plus JSONB extensions |
| ANSI X12 | Claims stored with core typed fields plus JSONB `segments` for full X12 segment preservation |
| HIPAA | Audit log with JSONB `details` for flexible access event recording |

---

## Core Tables

```sql
-- ============================================================
-- TENANT & ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',  -- tenant-specific configuration
    -- Example config:
    -- {
    --   "features": {"sdoh_screening": true, "ai_copilot": true, "hedis_reporting": false},
    --   "default_screening_instrument": "AHC-HRSN",
    --   "ccm_billing_enabled": true,
    --   "custom_risk_tiers": ["low", "rising", "moderate", "high", "critical"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    npi             VARCHAR(10),
    address         JSONB,                    -- full address as structured JSON
    -- Example address:
    -- {
    --   "line1": "123 Main St", "line2": "Suite 400",
    --   "city": "Austin", "state": "TX", "postal_code": "78701",
    --   "country": "US", "geo": {"lat": 30.2672, "lng": -97.7431}
    -- }
    contact         JSONB,                    -- phone, fax, email, website
    extensions      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);
CREATE INDEX idx_organisation_parent ON organisation(parent_id);
CREATE INDEX idx_organisation_npi ON organisation(npi);
```

## Patient Tables

```sql
CREATE TABLE patient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    fhir_id             VARCHAR(64),
    -- Core demographics (typed columns for indexed querying)
    mrn                 VARCHAR(50),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),
    -- USCDI v3 fields (typed — mandatory under HTI-1)
    sex_at_birth        VARCHAR(20),
    sexual_orientation  VARCHAR(50),
    gender_identity     VARCHAR(50),
    race                VARCHAR(50),
    ethnicity           VARCHAR(50),
    preferred_language  VARCHAR(10),
    -- Contact (structured JSONB for flexibility)
    address             JSONB,
    -- Example: {"line1": "456 Oak Ave", "city": "Houston", "state": "TX", "postal_code": "77001", "country": "US"}
    phones              JSONB,
    -- Example: [{"type": "home", "number": "+1-713-555-0100"}, {"type": "mobile", "number": "+1-713-555-0200"}]
    email               VARCHAR(255),
    -- Identifiers (JSONB array for multiple ID systems)
    identifiers         JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"system": "http://hl7.org/fhir/sid/us-medicare", "value": "1EG4-TE5-MK72", "type": "MB"},
    --   {"system": "http://hospital.example.org/mrn", "value": "MRN-12345", "type": "MR"},
    --   {"system": "http://hl7.org/fhir/sid/us-ssn", "value": "***-**-1234", "type": "SS"}
    -- ]
    -- Current risk (denormalised for fast worklist queries)
    risk_tier           VARCHAR(20),
    composite_risk_score DECIMAL(10,4),
    -- SDOH summary (denormalised)
    active_sdoh_needs   TEXT[],              -- e.g., ARRAY['food_insecurity', 'transportation']
    -- Extensions for programme-specific fields
    extensions          JSONB NOT NULL DEFAULT '{}',
    -- Example (Medicaid managed care):
    -- {
    --   "medicaid_id": "TX-MC-123456",
    --   "managed_care_plan": "Superior HealthPlan",
    --   "eligibility_category": "SSI",
    --   "enrollment_date": "2025-01-01",
    --   "assigned_health_home": "uuid-org-..."
    -- }
    -- Example (Medicare ACO):
    -- {
    --   "hic_number": "1EG4TE5MK72A",
    --   "aco_attribution_tin": "12-3456789",
    --   "prospective_hcc_raf": 1.245,
    --   "benchmark_amount": 12500.00
    -- }
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_tenant ON patient(tenant_id);
CREATE INDEX idx_patient_name ON patient(tenant_id, last_name, first_name);
CREATE INDEX idx_patient_dob ON patient(tenant_id, date_of_birth);
CREATE INDEX idx_patient_mrn ON patient(tenant_id, mrn);
CREATE INDEX idx_patient_risk ON patient(tenant_id, risk_tier);
CREATE INDEX idx_patient_sdoh ON patient(tenant_id) WHERE active_sdoh_needs IS NOT NULL AND array_length(active_sdoh_needs, 1) > 0;
CREATE INDEX idx_patient_identifiers ON patient USING GIN (identifiers jsonb_path_ops);
CREATE INDEX idx_patient_extensions ON patient USING GIN (extensions jsonb_path_ops);
```

## Provider & Care Team Tables

```sql
CREATE TABLE provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    npi             VARCHAR(10) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),
    specialties     JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"code": "207R00000X", "display": "Internal Medicine", "primary": true}]
    contact         JSONB,
    extensions      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provider_tenant ON provider(tenant_id);
CREATE INDEX idx_provider_npi ON provider(npi);

CREATE TABLE care_team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    fhir_id         VARCHAR(64),
    name            VARCHAR(255),
    status          VARCHAR(20) NOT NULL,
    category_code   VARCHAR(50),
    period_start    DATE,
    period_end      DATE,
    members         JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"provider_id": "uuid-...", "role": "primary_care_physician", "display": "Dr. Smith, MD", "period_start": "2025-01-01"},
    --   {"provider_id": "uuid-...", "role": "care_manager", "display": "Jane Doe, RN", "period_start": "2025-03-15"},
    --   {"provider_id": "uuid-...", "role": "behavioral_health", "display": "Dr. Johnson, LCSW", "period_start": "2025-06-01"},
    --   {"patient_id": "uuid-...", "role": "patient", "display": "John Patient"}
    -- ]
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_team_patient ON care_team(tenant_id, patient_id);
CREATE INDEX idx_care_team_members ON care_team USING GIN (members jsonb_path_ops);
```

## Clinical Data Tables

```sql
CREATE TABLE condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    fhir_id         VARCHAR(64),
    category        VARCHAR(50) NOT NULL,
    clinical_status VARCHAR(20) NOT NULL,
    -- Primary coding (typed for indexed querying)
    code_system     VARCHAR(255),
    code_value      VARCHAR(20) NOT NULL,
    code_display    VARCHAR(500),
    -- Additional codings (e.g., ICD-10 + SNOMED for same condition)
    codings         JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"system": "http://snomed.info/sct", "code": "73211009", "display": "Diabetes mellitus"}]
    onset_date      DATE,
    abatement_date  DATE,
    is_sdoh         BOOLEAN DEFAULT false,
    sdoh_category   VARCHAR(100),
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_patient ON condition(tenant_id, patient_id);
CREATE INDEX idx_condition_code ON condition(code_system, code_value);
CREATE INDEX idx_condition_sdoh ON condition(tenant_id) WHERE is_sdoh = true;

CREATE TABLE encounter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL,
    class_code      VARCHAR(20) NOT NULL,
    type_code       VARCHAR(20),
    period_start    TIMESTAMPTZ,
    period_end      TIMESTAMPTZ,
    organisation_id UUID REFERENCES organisation(id),
    provider_id     UUID REFERENCES provider(id),
    discharge_disposition VARCHAR(50),
    diagnoses       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"code": "E11.9", "system": "icd-10-cm", "display": "Type 2 diabetes", "rank": 1}]
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounter_patient ON encounter(tenant_id, patient_id);
CREATE INDEX idx_encounter_date ON encounter(tenant_id, period_start);

CREATE TABLE observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    fhir_id         VARCHAR(64),
    category_code   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    code_system     VARCHAR(255),
    code_value      VARCHAR(20) NOT NULL,
    code_display    VARCHAR(500),
    -- Value (typed for common patterns, JSONB for complex)
    value_quantity  DECIMAL(18,4),
    value_unit      VARCHAR(50),
    value_string    VARCHAR(1000),
    value_complex   JSONB,                   -- for component observations (e.g., blood pressure systolic + diastolic)
    -- Example: {"systolic": {"value": 120, "unit": "mmHg"}, "diastolic": {"value": 80, "unit": "mmHg"}}
    effective_date  TIMESTAMPTZ,
    encounter_id    UUID REFERENCES encounter(id),
    performer_id    UUID REFERENCES provider(id),
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_observation_patient ON observation(tenant_id, patient_id);
CREATE INDEX idx_observation_code ON observation(code_system, code_value);
CREATE INDEX idx_observation_date ON observation(tenant_id, patient_id, effective_date);
```

## Care Plan & Workflow Tables

```sql
CREATE TABLE care_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    care_team_id    UUID REFERENCES care_team(id),
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL,
    intent          VARCHAR(20) NOT NULL DEFAULT 'plan',
    category_code   VARCHAR(50),
    title           VARCHAR(500),
    description     TEXT,
    period_start    DATE,
    period_end      DATE,
    author_id       UUID REFERENCES provider(id),
    -- Conditions addressed (JSONB array of references)
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"condition_id": "uuid-...", "code": "E11.9", "display": "Type 2 diabetes"}]
    -- Goals (JSONB array — keeps goals co-located with care plan)
    goals           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"id": "uuid-goal-1", "status": "active", "description": "HbA1c < 7%", "target_date": "2026-09-01",
    --    "target_value": 7.0, "target_unit": "%", "achievement": "in-progress", "is_sdoh": false},
    --   {"id": "uuid-goal-2", "status": "active", "description": "Resolve food insecurity",
    --    "target_date": "2026-06-30", "achievement": "improving", "is_sdoh": true, "sdoh_category": "food_insecurity"}
    -- ]
    -- Activities (JSONB array)
    activities      JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"type": "medication_review", "status": "completed", "scheduled": "2026-05-01", "performer_id": "uuid-..."},
    --   {"type": "sdoh_referral", "status": "in-progress", "referral_id": "uuid-...", "description": "Food bank referral"}
    -- ]
    -- Programme-specific extensions
    extensions      JSONB NOT NULL DEFAULT '{}',
    -- Example (TCM care plan):
    -- {
    --   "tcm_discharge_date": "2026-05-10",
    --   "tcm_complexity": "high",
    --   "tcm_follow_up_within_7_days": true,
    --   "tcm_medication_reconciliation_date": "2026-05-12"
    -- }
    replaces_id     UUID REFERENCES care_plan(id),
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_plan_patient ON care_plan(tenant_id, patient_id);
CREATE INDEX idx_care_plan_status ON care_plan(tenant_id, status);
CREATE INDEX idx_care_plan_goals ON care_plan USING GIN (goals jsonb_path_ops);

CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    care_plan_id    UUID REFERENCES care_plan(id),
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(20) DEFAULT 'routine',
    task_type       VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    requester_id    UUID REFERENCES provider(id),
    owner_id        UUID REFERENCES provider(id),
    due_date        DATE,
    completed_date  TIMESTAMPTZ,
    time_spent_min  INTEGER DEFAULT 0,
    -- Flexible metadata for task-type-specific fields
    task_data       JSONB NOT NULL DEFAULT '{}',
    -- Example (outreach task):
    -- {"channel": "phone", "attempt_number": 2, "last_attempt": "2026-05-18", "outcome": "no_answer"}
    -- Example (med reconciliation):
    -- {"medications_reviewed": 8, "discrepancies_found": 2, "discrepancies_resolved": 1}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_owner ON task(tenant_id, owner_id, status);
CREATE INDEX idx_task_due ON task(tenant_id, due_date) WHERE status NOT IN ('completed', 'cancelled');
CREATE INDEX idx_task_type ON task(tenant_id, task_type, status);
```

## SDOH & Referral Tables

```sql
CREATE TABLE sdoh_screening (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    encounter_id    UUID REFERENCES encounter(id),
    instrument      VARCHAR(50) NOT NULL,
    screening_date  TIMESTAMPTZ NOT NULL,
    screener_id     UUID REFERENCES provider(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
    -- All responses in a single JSONB array (avoids a separate response table)
    responses       JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"question_code": "96777-8", "question_text": "Within the past 12 months, you worried food would run out",
    --    "answer_code": "LA31993-1", "answer_text": "Often true", "risk_flag": true, "sdoh_category": "food_insecurity"},
    --   {"question_code": "71802-3", "question_text": "Problems with housing",
    --    "answer_code": "LA31976-6", "answer_text": "No", "risk_flag": false, "sdoh_category": "housing_instability"}
    -- ]
    identified_needs TEXT[],                 -- summary: ARRAY['food_insecurity']
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sdoh_screening_patient ON sdoh_screening(tenant_id, patient_id);
CREATE INDEX idx_sdoh_screening_needs ON sdoh_screening USING GIN (identified_needs);

CREATE TABLE referral (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    care_plan_id    UUID REFERENCES care_plan(id),
    fhir_id         VARCHAR(64),
    referral_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(20) DEFAULT 'routine',
    reason_code     VARCHAR(20),
    reason_display  VARCHAR(500),
    referring_provider_id   UUID REFERENCES provider(id),
    receiving_org_id        UUID REFERENCES organisation(id),
    receiving_provider_id   UUID REFERENCES provider(id),
    requested_date  DATE NOT NULL,
    scheduled_date  DATE,
    completed_date  DATE,
    abandonment_risk_score DECIMAL(5,4),
    -- SDOH closed-loop tracking
    sdoh_category   VARCHAR(100),
    cbo_tracking    JSONB,
    -- Example:
    -- {
    --   "cbo_name": "Central Texas Food Bank",
    --   "cbo_contact": "Jane Smith",
    --   "service_type": "food_assistance",
    --   "confirmation_date": "2026-05-15",
    --   "confirmation_method": "portal",
    --   "services_provided": ["monthly food box", "SNAP application assistance"],
    --   "follow_up_date": "2026-06-15"
    -- }
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_referral_patient ON referral(tenant_id, patient_id);
CREATE INDEX idx_referral_status ON referral(tenant_id, referral_type, status);
CREATE INDEX idx_referral_risk ON referral(tenant_id, abandonment_risk_score DESC) WHERE status = 'active';
```

## Quality Measure & Care Gap Tables

```sql
CREATE TABLE quality_measure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    measure_id_ext  VARCHAR(50) NOT NULL,
    measure_name    VARCHAR(500) NOT NULL,
    measure_set     VARCHAR(50) NOT NULL,
    domain          VARCHAR(100),
    measure_year    INTEGER NOT NULL,
    -- Measure criteria as JSONB (enables programme-specific custom measures)
    criteria        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "denominator": {"conditions": ["E11.*"], "age_range": [18, 75], "continuous_enrollment_months": 12},
    --   "numerator": {"observation_code": "4548-4", "value_threshold": 9.0, "comparison": "less_than"},
    --   "exclusions": [{"conditions": ["O24.*"]}, {"hospice": true}]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quality_measure_set ON quality_measure(tenant_id, measure_set, measure_year);

CREATE TABLE care_gap (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    measure_id      UUID NOT NULL REFERENCES quality_measure(id),
    gap_status      VARCHAR(20) NOT NULL,
    identified_date DATE NOT NULL,
    due_date        DATE,
    closed_date     DATE,
    closure_method  VARCHAR(50),
    assigned_to_id  UUID REFERENCES provider(id),
    -- Evidence and history as JSONB
    evidence        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "closure_evidence_type": "lab_result",
    --   "evidence_observation_id": "uuid-...",
    --   "evidence_date": "2026-05-10",
    --   "outreach_history": [
    --     {"date": "2026-04-01", "channel": "sms", "status": "sent"},
    --     {"date": "2026-04-15", "channel": "phone", "status": "reached", "notes": "Scheduled lab appointment"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_gap_patient ON care_gap(tenant_id, patient_id);
CREATE INDEX idx_care_gap_status ON care_gap(tenant_id, gap_status);
CREATE INDEX idx_care_gap_assigned ON care_gap(tenant_id, assigned_to_id, gap_status);
```

## Claims, Risk, and Billing Tables

```sql
CREATE TABLE claim (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    claim_number    VARCHAR(50) NOT NULL,
    claim_type      VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    payer_id        UUID REFERENCES organisation(id),
    provider_id     UUID REFERENCES provider(id),
    service_date_from DATE NOT NULL,
    service_date_to   DATE,
    billed_amount   DECIMAL(12,2),
    allowed_amount  DECIMAL(12,2),
    paid_amount     DECIMAL(12,2),
    primary_dx_code VARCHAR(10),
    -- Claim lines and full X12 detail in JSONB (avoids separate claim_line table)
    lines           JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"line": 1, "cpt": "99214", "modifier": "25", "units": 1, "billed": 175.00, "allowed": 120.00, "paid": 96.00, "pos": "11"},
    --   {"line": 2, "cpt": "83036", "units": 1, "billed": 45.00, "allowed": 30.00, "paid": 24.00, "pos": "11"}
    -- ]
    -- Raw X12 segments preserved for reprocessing
    x12_segments    JSONB,
    extensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_patient ON claim(tenant_id, patient_id);
CREATE INDEX idx_claim_date ON claim(tenant_id, service_date_from);

CREATE TABLE risk_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL,
    score_type      VARCHAR(50) NOT NULL,
    score_value     DECIMAL(10,4) NOT NULL,
    risk_tier       VARCHAR(20),
    model_version   VARCHAR(50),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "contributing_factors": [
    --     {"factor": "hcc_count", "value": 4, "weight": 0.35},
    --     {"factor": "ed_visits_12m", "value": 3, "weight": 0.25},
    --     {"factor": "sdoh_food_insecurity", "value": true, "weight": 0.15}
    --   ]
    -- }
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_score_patient ON risk_score(tenant_id, patient_id, score_type);
```

## Audit & RBAC Tables

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    patient_id      UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ip": "10.0.1.50", "user_agent": "Mozilla/5.0...", "fields_accessed": ["ssn", "dob"], "query_type": "patient_search"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_patient ON audit_log(patient_id, created_at);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider_id     UUID REFERENCES provider(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    roles           JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"role": "care_manager", "org_id": "uuid-...", "org_name": "Austin Primary Care"}]
    preferences     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_app_user_roles ON app_user USING GIN (roles jsonb_path_ops);
```

## Example JSONB Queries

```sql
-- Find all patients with a specific Medicaid ID in extensions
SELECT * FROM patient
WHERE tenant_id = 'tenant-uuid'
  AND extensions @> '{"medicaid_id": "TX-MC-123456"}';

-- Find care plans with an active SDOH goal
SELECT * FROM care_plan
WHERE tenant_id = 'tenant-uuid'
  AND goals @> '[{"is_sdoh": true, "status": "active"}]';

-- Find care teams where a specific provider is a member
SELECT * FROM care_team
WHERE tenant_id = 'tenant-uuid'
  AND members @> '[{"provider_id": "provider-uuid-here"}]';

-- Find patients with food insecurity needs
SELECT * FROM patient
WHERE tenant_id = 'tenant-uuid'
  AND 'food_insecurity' = ANY(active_sdoh_needs);

-- Find claims with a specific CPT code in any line
SELECT * FROM claim
WHERE tenant_id = 'tenant-uuid'
  AND lines @> '[{"cpt": "99214"}]';

-- Find users with care_manager role at a specific org
SELECT * FROM app_user
WHERE tenant_id = 'tenant-uuid'
  AND roles @> '[{"role": "care_manager", "org_id": "org-uuid-here"}]';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organisation | 2 | Organisation address/contact in JSONB |
| Patient | 1 | Identifiers, phones, extensions in JSONB |
| Provider & Care Team | 2 | Team members as JSONB array; specialties as JSONB |
| Clinical Data | 3 | Conditions, encounters, observations; diagnoses/codings in JSONB |
| Care Plan & Tasks | 2 | Goals and activities embedded as JSONB arrays in care plan |
| SDOH & Referrals | 2 | Screening responses as JSONB array; CBO tracking as JSONB |
| Quality & Care Gaps | 2 | Measure criteria as JSONB; gap evidence/history as JSONB |
| Claims & Risk | 2 | Claim lines as JSONB array; risk factors as JSONB |
| Audit & RBAC | 2 | User roles as JSONB array; audit details as JSONB |
| **Total** | **18** | Significantly fewer tables than normalised model |

---

## Key Design Decisions

1. **JSONB for variable/extensible data, typed columns for indexed/queried data** — The rule of thumb: if a field is used in WHERE clauses, JOINs, or ORDER BY, it is a typed column. If it varies by client, programme, or jurisdiction, it goes in JSONB. This gives performance where it matters and flexibility everywhere else.

2. **Care plan goals and activities embedded as JSONB** — Rather than separate `goal` and `care_plan_activity` tables (as in the normalised model), goals and activities are JSONB arrays within the care plan. This reduces table count and simplifies the common query pattern of "load a care plan with all its goals and activities." For platforms where goals are heavily queried independently (e.g., "find all patients with unmet HbA1c goals"), a materialised view or extracted column can be added.

3. **Care team members as JSONB array** — Care team composition changes infrequently and is almost always loaded as a complete list. Embedding members as a JSONB array avoids a junction table and a JOIN for the most common access pattern.

4. **Claim lines as JSONB array** — A claim and its lines are always loaded together. Embedding lines as JSONB eliminates the `claim_line` table. If analytical queries across individual claim lines are needed (e.g., "find all claims containing CPT 99214"), the JSONB containment operator (`@>`) with a GIN index handles this efficiently.

5. **SDOH screening responses as JSONB** — A screening and its responses are a single conceptual unit. Embedding responses as a JSONB array keeps them co-located and avoids a JOIN for the standard access pattern. The `identified_needs` array column provides a fast indexed path for population-level SDOH queries.

6. **Tenant configuration in JSONB** — Each tenant has a `config` JSONB column that controls feature flags, default instruments, billing settings, and custom risk tier definitions. This allows per-tenant customisation without schema changes.

7. **GIN indexes on all JSONB columns used in queries** — Every JSONB column that supports query patterns has a `jsonb_path_ops` GIN index. This enables fast containment queries (`@>`) for the JSONB search patterns shown above.

8. **Extensions column pattern** — Every major entity has an `extensions` JSONB column, mirroring FHIR's extension mechanism. This is where programme-specific, jurisdiction-specific, or client-specific fields are stored. Application-layer JSON Schema validation ensures data quality for known extension schemas.
