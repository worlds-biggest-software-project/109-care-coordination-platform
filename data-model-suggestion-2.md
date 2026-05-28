# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Care Coordination Platform · Created: 2026-05-19

## Philosophy

This model treats the **event store** as the single source of truth. Every state change — a care plan being created, a task being assigned, a care gap being closed, an SDOH screening being completed — is recorded as an immutable event in a single append-only `event_store` table. The current state of any entity is derived by replaying its events in sequence. Materialised read models (projections) are maintained in denormalised tables optimised for the specific query patterns care managers, analysts, and AI agents need.

This is the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from projections. The event store provides a complete, immutable audit trail that satisfies HIPAA access logging, CMS documentation requirements for CCM/TCM billing, and temporal queries ("what was this patient's care plan on March 15th?"). Event replay enables point-in-time reconstruction of any entity's state, which is invaluable for regulatory audits, dispute resolution, and AI training on historical care patterns.

Real-world healthcare systems using event-sourced patterns include clinical trial management systems (where regulatory audit trails are mandatory), claims adjudication engines, and EHR systems that maintain complete amendment histories. Financial services platforms (banking, trading) have proven this pattern at scale for regulatory compliance.

**Best for:** Organisations that require complete audit trails for regulatory compliance, temporal queries for care plan evolution, and AI/ML training on historical care coordination patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is recorded forever
- (+) Temporal queries: reconstruct any entity's state at any point in time
- (+) Natural fit for HIPAA audit logging — audit is the architecture, not an add-on
- (+) Rich event stream for AI/ML model training and analytics
- (+) Schema evolution is easier — new event types don't require ALTER TABLE on existing data
- (-) Higher storage requirements — events accumulate indefinitely
- (-) Read model consistency is eventual, not immediate (typically milliseconds, but not zero)
- (-) Debugging requires understanding event replay, not just inspecting current state
- (-) Projection rebuild after schema changes can be slow for large event volumes
- (-) More complex development model — developers must think in events, not CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | FHIR resources are projected from events; each resource type has a read-model projection table aligned with FHIR field structures |
| HIPAA Security Rule | The event store IS the audit trail — every data access and modification is an event; no separate audit logging needed |
| CMS TCM/CCM Billing | Time-tracking events (care_manager_time_logged) provide granular, immutable documentation of time spent per patient |
| Gravity SDOH IG | SDOH screening, referral, and closed-loop confirmation events form a traceable chain from screening to resolution |
| HL7 Da Vinci CDex | Clinical data exchange requests and responses are events, enabling replay of the full exchange history |
| NCQA HEDIS | Care gap events (identified, assigned, outreach_sent, closed) provide measure-ready evidence chains |
| USCDI v3/v4 | Read-model projections include all USCDI-mandated data elements |
| CMS-0057-F | FHIR API responses are served from read-model projections |

---

## Event Store (Write Side)

```sql
-- ============================================================
-- CORE EVENT STORE — Single append-only table
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,           -- aggregate/entity ID (e.g., patient ID, care plan ID)
    stream_type     VARCHAR(50) NOT NULL,    -- aggregate type: 'patient', 'care_plan', 'referral', 'care_gap', 'task', etc.
    event_type      VARCHAR(100) NOT NULL,   -- e.g., 'care_plan_created', 'task_assigned', 'care_gap_closed'
    event_version   INTEGER NOT NULL,        -- sequential version within this stream
    event_data      JSONB NOT NULL,          -- full event payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- actor, IP, correlation ID, causation ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Optimistic concurrency: no two events can have the same version for a stream
    UNIQUE (stream_id, event_version)
);

-- Primary query pattern: replay all events for an entity
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Query by event type across all streams (for projections)
CREATE INDEX idx_event_type ON event_store(event_type, created_at);

-- Tenant-scoped queries
CREATE INDEX idx_event_tenant ON event_store(tenant_id, stream_type, created_at);

-- Temporal queries: find events in a time range
CREATE INDEX idx_event_time ON event_store(created_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store (...) PARTITION BY RANGE (created_at);

-- ============================================================
-- EVENT TYPE EXAMPLES
-- ============================================================

-- Example: care_plan_created event
-- {
--   "event_type": "care_plan_created",
--   "stream_type": "care_plan",
--   "event_data": {
--     "patient_id": "uuid-...",
--     "care_team_id": "uuid-...",
--     "status": "draft",
--     "intent": "plan",
--     "title": "Diabetes Management Plan",
--     "category_code": "longitudinal",
--     "conditions": ["uuid-cond-1", "uuid-cond-2"],
--     "author_id": "uuid-provider-..."
--   },
--   "metadata": {
--     "actor_id": "uuid-user-...",
--     "actor_type": "care_manager",
--     "ip_address": "10.0.1.50",
--     "correlation_id": "uuid-corr-...",
--     "source": "web_app"
--   }
-- }

-- Example: care_gap_closed event
-- {
--   "event_type": "care_gap_closed",
--   "stream_type": "care_gap",
--   "event_data": {
--     "patient_id": "uuid-...",
--     "measure_id": "uuid-...",
--     "closure_method": "ai_agent",
--     "closure_evidence_type": "lab_result",
--     "evidence_reference": "uuid-observation-...",
--     "closed_date": "2026-05-19"
--   },
--   "metadata": {
--     "actor_id": "system-ai-agent-001",
--     "actor_type": "ai_agent",
--     "correlation_id": "uuid-corr-...",
--     "source": "autonomous_gap_closure_agent"
--   }
-- }

-- Example: sdoh_screening_completed event
-- {
--   "event_type": "sdoh_screening_completed",
--   "stream_type": "patient",
--   "event_data": {
--     "screening_id": "uuid-...",
--     "instrument": "AHC-HRSN",
--     "responses": [
--       {"question_code": "96777-8", "answer_code": "LA31993-1", "risk_flag": true, "sdoh_category": "food_insecurity"},
--       {"question_code": "71802-3", "answer_code": "LA31976-6", "risk_flag": false, "sdoh_category": "housing_instability"}
--     ],
--     "identified_needs": ["food_insecurity"],
--     "screener_id": "uuid-provider-..."
--   }
-- }
```

## Snapshot Store (Performance Optimisation)

```sql
-- ============================================================
-- SNAPSHOTS — periodic state captures to avoid full replay
-- ============================================================

CREATE TABLE event_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,       -- event_version at time of snapshot
    state_data      JSONB NOT NULL,          -- full current state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, snapshot_version DESC);

-- Snapshots are taken every N events (e.g., every 50 events per stream)
-- To reconstruct state: load latest snapshot + replay events after snapshot_version
```

## Read Model Projections (Query Side)

```sql
-- ============================================================
-- PROJECTION: Patient Summary (denormalised for care manager worklist)
-- ============================================================

CREATE TABLE proj_patient_summary (
    patient_id          UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    mrn                 VARCHAR(50),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),
    primary_pcp_name    VARCHAR(255),
    primary_pcp_npi     VARCHAR(10),
    -- Risk scores (latest)
    composite_risk_score    DECIMAL(10,4),
    composite_risk_tier     VARCHAR(20),
    readmission_risk_score  DECIMAL(10,4),
    sdoh_risk_score         DECIMAL(10,4),
    -- Care gap counts
    open_care_gaps      INTEGER DEFAULT 0,
    pending_care_gaps   INTEGER DEFAULT 0,
    -- Active care plan
    active_care_plan_id UUID,
    active_care_plan_title VARCHAR(500),
    -- SDOH flags
    has_sdoh_needs      BOOLEAN DEFAULT false,
    sdoh_categories     TEXT[],              -- array of active SDOH categories
    -- Open tasks
    open_task_count     INTEGER DEFAULT 0,
    next_task_due_date  DATE,
    -- Attribution
    attributed_programme VARCHAR(255),
    attributed_payer     VARCHAR(255),
    -- Timestamps
    last_encounter_date DATE,
    last_outreach_date  DATE,
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_patient_tenant ON proj_patient_summary(tenant_id);
CREATE INDEX idx_proj_patient_risk ON proj_patient_summary(tenant_id, composite_risk_tier);
CREATE INDEX idx_proj_patient_gaps ON proj_patient_summary(tenant_id, open_care_gaps DESC);

-- ============================================================
-- PROJECTION: Care Plan Current State
-- ============================================================

CREATE TABLE proj_care_plan (
    care_plan_id    UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    patient_id      UUID NOT NULL,
    care_team_id    UUID,
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL,
    intent          VARCHAR(20) NOT NULL,
    title           VARCHAR(500),
    description     TEXT,
    category_code   VARCHAR(50),
    period_start    DATE,
    period_end      DATE,
    author_id       UUID,
    author_name     VARCHAR(255),
    condition_codes TEXT[],                 -- denormalised array of ICD-10/SNOMED codes
    condition_displays TEXT[],              -- denormalised condition names
    goal_count      INTEGER DEFAULT 0,
    active_goals    INTEGER DEFAULT 0,
    completed_goals INTEGER DEFAULT 0,
    activity_count  INTEGER DEFAULT 0,
    version         INTEGER NOT NULL,       -- current event version
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_care_plan_patient ON proj_care_plan(tenant_id, patient_id);
CREATE INDEX idx_proj_care_plan_status ON proj_care_plan(tenant_id, status);

-- ============================================================
-- PROJECTION: Care Gap Worklist
-- ============================================================

CREATE TABLE proj_care_gap (
    care_gap_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    patient_id      UUID NOT NULL,
    patient_name    VARCHAR(255),           -- denormalised for worklist display
    measure_id      UUID NOT NULL,
    measure_name    VARCHAR(500),
    measure_set     VARCHAR(50),
    gap_status      VARCHAR(20) NOT NULL,
    identified_date DATE NOT NULL,
    due_date        DATE,
    closed_date     DATE,
    closure_method  VARCHAR(50),
    assigned_to_id  UUID,
    assigned_to_name VARCHAR(255),
    outreach_count  INTEGER DEFAULT 0,
    last_outreach_date DATE,
    version         INTEGER NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_care_gap_tenant ON proj_care_gap(tenant_id, gap_status);
CREATE INDEX idx_proj_care_gap_assigned ON proj_care_gap(tenant_id, assigned_to_id, gap_status);
CREATE INDEX idx_proj_care_gap_due ON proj_care_gap(tenant_id, due_date) WHERE gap_status = 'open';

-- ============================================================
-- PROJECTION: Task Worklist
-- ============================================================

CREATE TABLE proj_task (
    task_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    patient_id      UUID NOT NULL,
    patient_name    VARCHAR(255),
    care_plan_id    UUID,
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(20),
    task_type       VARCHAR(50) NOT NULL,
    description     TEXT,
    requester_name  VARCHAR(255),
    owner_id        UUID,
    owner_name      VARCHAR(255),
    due_date        DATE,
    completed_date  TIMESTAMPTZ,
    time_spent_min  INTEGER DEFAULT 0,
    version         INTEGER NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_task_owner ON proj_task(tenant_id, owner_id, status);
CREATE INDEX idx_proj_task_due ON proj_task(tenant_id, due_date) WHERE status NOT IN ('completed', 'cancelled', 'failed');

-- ============================================================
-- PROJECTION: Referral Tracker
-- ============================================================

CREATE TABLE proj_referral (
    referral_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    patient_id      UUID NOT NULL,
    patient_name    VARCHAR(255),
    referral_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(20),
    reason_display  VARCHAR(500),
    referring_provider_name VARCHAR(255),
    receiving_org_name      VARCHAR(255),
    receiving_provider_name VARCHAR(255),
    requested_date  DATE,
    scheduled_date  DATE,
    completed_date  DATE,
    abandonment_risk_score DECIMAL(5,4),
    sdoh_category   VARCHAR(100),
    cbo_confirmed   BOOLEAN DEFAULT false,
    cbo_confirmation_date DATE,
    version         INTEGER NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_referral_tenant ON proj_referral(tenant_id, status);
CREATE INDEX idx_proj_referral_risk ON proj_referral(tenant_id, abandonment_risk_score DESC) WHERE status = 'active';

-- ============================================================
-- PROJECTION: SDOH Dashboard
-- ============================================================

CREATE TABLE proj_sdoh_patient (
    patient_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    sdoh_category   VARCHAR(100) NOT NULL,   -- food_insecurity, housing_instability, transportation, etc.
    screening_date  TIMESTAMPTZ,
    screening_instrument VARCHAR(50),
    risk_level      VARCHAR(20),             -- 'identified', 'active', 'resolved'
    referral_id     UUID,
    referral_status VARCHAR(20),
    cbo_name        VARCHAR(255),
    cbo_confirmed   BOOLEAN DEFAULT false,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (patient_id, sdoh_category)
);

CREATE INDEX idx_proj_sdoh_tenant ON proj_sdoh_patient(tenant_id, sdoh_category, risk_level);

-- ============================================================
-- PROJECTION: Quality Measure Performance
-- ============================================================

CREATE TABLE proj_measure_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    measure_id      UUID NOT NULL,
    measure_name    VARCHAR(500),
    measure_set     VARCHAR(50),
    reporting_period DATE NOT NULL,          -- first of month
    organisation_id UUID,
    denominator     INTEGER NOT NULL,
    numerator       INTEGER NOT NULL,
    exclusions      INTEGER DEFAULT 0,
    rate            DECIMAL(7,4),
    open_gaps       INTEGER DEFAULT 0,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_measure_perf ON proj_measure_performance(tenant_id, measure_id, reporting_period);
```

## Reference Data Tables (Shared, Not Event-Sourced)

```sql
-- ============================================================
-- REFERENCE DATA — these are static/slowly-changing and NOT event-sourced
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL,
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    npi             VARCHAR(10) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),
    specialty_code  VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quality_measure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    measure_id_ext  VARCHAR(50) NOT NULL,
    measure_name    VARCHAR(500) NOT NULL,
    measure_set     VARCHAR(50) NOT NULL,
    domain          VARCHAR(100),
    measure_year    INTEGER NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RBAC (not event-sourced — standard relational)
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider_id     UUID REFERENCES provider(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (user_id, role_id)
);
```

## Projection Rebuild Infrastructure

```sql
-- ============================================================
-- PROJECTION TRACKING — tracks which events each projection has processed
-- ============================================================

CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_rebuilt_at  TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example rows:
-- ('proj_patient_summary', 'uuid-...', '2026-05-19 10:00:00Z', 1500000, '2026-05-01 00:00:00Z', ...)
-- ('proj_care_plan',       'uuid-...', '2026-05-19 10:00:00Z', 500000,  '2026-05-01 00:00:00Z', ...)
```

## Example Temporal Query

```sql
-- Reconstruct a patient's care plan state as of March 15, 2026
SELECT event_data
FROM event_store
WHERE stream_id = 'care-plan-uuid-here'
  AND stream_type = 'care_plan'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- The application replays these events to build the care plan state at that point in time.

-- Alternative: if a snapshot exists before the target date, start from there
SELECT state_data, snapshot_version
FROM event_snapshot
WHERE stream_id = 'care-plan-uuid-here'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY snapshot_version DESC
LIMIT 1;

-- Then replay only events after the snapshot:
SELECT event_data
FROM event_store
WHERE stream_id = 'care-plan-uuid-here'
  AND event_version > [snapshot_version]
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;
```

## Example Event Stream for CCM Billing Documentation

```sql
-- Retrieve all time-tracking events for a patient in a given month for CCM billing
SELECT 
    event_type,
    event_data->>'duration_minutes' AS minutes,
    event_data->>'activity_description' AS activity,
    metadata->>'actor_id' AS care_manager_id,
    created_at
FROM event_store
WHERE stream_type = 'patient'
  AND stream_id = 'patient-uuid-here'
  AND event_type IN ('care_manager_time_logged', 'outreach_completed', 'care_plan_reviewed')
  AND created_at >= '2026-05-01T00:00:00Z'
  AND created_at < '2026-06-01T00:00:00Z'
ORDER BY created_at ASC;

-- Sum for billing determination
SELECT 
    SUM((event_data->>'duration_minutes')::integer) AS total_minutes
FROM event_store
WHERE stream_type = 'patient'
  AND stream_id = 'patient-uuid-here'
  AND event_type IN ('care_manager_time_logged', 'outreach_completed', 'care_plan_reviewed')
  AND created_at >= '2026-05-01T00:00:00Z'
  AND created_at < '2026-06-01T00:00:00Z';
-- If >= 20 minutes: qualifies for CPT 99490 (CCM)
-- If >= 40 minutes: qualifies for CPT 99491 (complex CCM)
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event table (partitioned by month) |
| Snapshots | 1 | Periodic state snapshots for replay performance |
| Projection: Patient | 1 | Denormalised patient summary for worklist |
| Projection: Care Plan | 1 | Current care plan state |
| Projection: Care Gap | 1 | Care gap worklist |
| Projection: Task | 1 | Task worklist |
| Projection: Referral | 1 | Referral tracker |
| Projection: SDOH | 1 | SDOH dashboard per patient per category |
| Projection: Quality | 1 | Measure performance aggregates |
| Reference Data | 4 | Tenant, organisation, provider, quality measure |
| RBAC | 3 | Users, roles, user-role assignments |
| Infrastructure | 1 | Projection checkpoint tracking |
| **Total** | **17** | Plus projections added as needed for new views |

---

## Key Design Decisions

1. **Single event store table** — All events go to one table regardless of aggregate type. The `stream_type` column enables filtered queries. This simplifies infrastructure (one table to partition, backup, replicate) while supporting any number of aggregate types. An alternative is one event table per aggregate type, but that increases operational complexity without meaningful performance gain when properly indexed.

2. **JSONB event payload** — Event data is stored as JSONB rather than typed columns because event schemas evolve over time. A `care_plan_created` event in v1 may have different fields than v2. JSONB allows coexistence of event versions without ALTER TABLE migrations. The application's event deserialiser handles version differences.

3. **Optimistic concurrency via unique constraint** — The `UNIQUE (stream_id, event_version)` constraint prevents concurrent writes to the same aggregate from creating conflicting events. This is the standard optimistic concurrency mechanism for event stores.

4. **Snapshots for replay performance** — For aggregates with many events (e.g., a patient with years of history), full replay from event 1 is slow. Periodic snapshots (every 50-100 events) allow reconstruction from the latest snapshot plus a small number of subsequent events.

5. **Projections are disposable** — Read-model tables can be dropped and rebuilt from the event store at any time. This means schema changes to projections are non-destructive: update the projection code, rebuild the table, and the new schema is populated from the complete event history.

6. **Audit trail is the architecture** — In the normalised relational model, audit logging is an add-on (`audit_log` table). In this model, the event store IS the audit trail. Every state change, including who made it, when, from what IP, and with what correlation ID, is captured in the event metadata. HIPAA audit requirements are satisfied by the core architecture.

7. **Reference data is not event-sourced** — Tenants, organisations, providers, and quality measure definitions are slowly-changing reference data. Event-sourcing them adds complexity without benefit. They use standard relational tables.

8. **CCM/TCM billing from event stream** — Rather than a separate billing log, CCM and TCM time is computed from the event stream. Every time-logged event contributes to the monthly total. This is more accurate than a separate table because it includes the complete context (what activity was performed, by whom, for how long) and cannot be retroactively edited without a new corrective event.
