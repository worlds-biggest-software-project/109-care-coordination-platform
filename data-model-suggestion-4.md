# Data Model Suggestion 4: Graph-Relational

> Project: Care Coordination Platform · Created: 2026-05-19

## Philosophy

This model combines a **relational core** for operational CRUD with a **property graph layer** for relationship-intensive queries. Care coordination is fundamentally a graph problem: patients are connected to providers, care teams, organisations, referral networks, community-based organisations, and quality programmes through complex, many-to-many, temporally-scoped relationships. Queries like "find all providers who share patients with Dr. Smith across three ACO programmes," "trace the referral chain from PCP to specialist to post-acute to CBO for this patient," and "identify potential conflicts of interest in the care team" are natural graph traversals that are expensive or awkward in relational SQL but trivial in a graph query language.

The graph layer is implemented using two generic tables — `graph_node` and `graph_edge` — within PostgreSQL itself, rather than requiring a separate graph database (Neo4j, Amazon Neptune). This keeps the entire data model in a single database, avoids dual-write consistency issues, and leverages PostgreSQL's ACID transactions, row-level security, and existing operational tooling. For organisations that outgrow PostgreSQL's graph query performance, the graph tables can be replicated to a dedicated graph database without changing the relational core.

This pattern is used by Innovaccer's FHIR+ Data Activation Platform (which stores approximately 70 entities in a graph structure optimised for high-performance queries), social network platforms, fraud detection systems, and supply chain management tools — all domains where relationship traversal is a primary query pattern.

**Best for:** Platforms that need to answer relationship-intensive questions: referral network analysis, care team overlap, provider network optimisation, conflict-of-interest detection, and social network effects in patient engagement.

**Trade-offs:**
- (+) Relationship queries (multi-hop traversals) are natural and performant
- (+) Referral network analysis, provider overlap, and care pathway tracing are first-class capabilities
- (+) Graph edges can carry temporal and contextual properties (role, start/end date, weight)
- (+) Single database — no dual-write consistency problems between relational and graph stores
- (+) Graph structure supports AI/ML features like network-based risk scoring and referral leakage prediction
- (-) Developers must learn graph query patterns (recursive CTEs or graph extensions)
- (-) Graph traversal performance degrades for very deep traversals (>5 hops) without careful index design
- (-) Generic `graph_node`/`graph_edge` tables are less self-documenting than typed tables
- (-) Reporting queries that mix graph traversal with relational aggregation can be complex
- (-) PostgreSQL does not have native graph query language (requires recursive CTEs or Apache AGE extension)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | FHIR resources stored in relational tables; FHIR references (Patient→Practitioner, CarePlan→CareTeam) represented as typed graph edges |
| Gravity SDOH IG | SDOH closed-loop referral chain (Screening→Condition→ServiceRequest→Task→CBO Confirmation) modeled as a connected subgraph |
| NCQA HEDIS | Quality measure attribution relationships (patient→programme→payer→provider) tracked as graph edges |
| CMS-0057-F | Provider network relationships for Provider Access API modeled in the graph layer |
| HIPAA | Graph edge access logged in audit trail; graph queries filtered by tenant |
| Da Vinci CDex | Clinical data exchange relationships between providers modeled as graph edges with exchange metadata |

---

## Relational Core Tables

```sql
-- ============================================================
-- STANDARD RELATIONAL TABLES for operational CRUD
-- These are the same core tables as the normalised model,
-- but with a lighter footprint — relationship-heavy queries
-- are delegated to the graph layer.
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
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state_code      VARCHAR(2),
    postal_code     VARCHAR(10),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_tenant ON organisation(tenant_id);
CREATE INDEX idx_organisation_npi ON organisation(npi);

CREATE TABLE patient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    fhir_id             VARCHAR(64),
    mrn                 VARCHAR(50),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),
    race                VARCHAR(50),
    ethnicity           VARCHAR(50),
    preferred_language  VARCHAR(10),
    address_line1       VARCHAR(255),
    city                VARCHAR(100),
    state_code          VARCHAR(2),
    postal_code         VARCHAR(10),
    phone_mobile        VARCHAR(20),
    email               VARCHAR(255),
    risk_tier           VARCHAR(20),
    composite_risk_score DECIMAL(10,4),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_patient_tenant ON patient(tenant_id);
CREATE INDEX idx_patient_name ON patient(tenant_id, last_name, first_name);

CREATE TABLE provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organisation_id UUID REFERENCES organisation(id),
    npi             VARCHAR(10) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),
    specialty_code  VARCHAR(20),
    specialty_name  VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provider_tenant ON provider(tenant_id);
CREATE INDEX idx_provider_npi ON provider(npi);

CREATE TABLE care_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    fhir_id         VARCHAR(64),
    status          VARCHAR(20) NOT NULL,
    intent          VARCHAR(20) NOT NULL DEFAULT 'plan',
    title           VARCHAR(500),
    description     TEXT,
    period_start    DATE,
    period_end      DATE,
    author_id       UUID REFERENCES provider(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_plan_patient ON care_plan(tenant_id, patient_id);

CREATE TABLE condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    category        VARCHAR(50) NOT NULL,
    clinical_status VARCHAR(20) NOT NULL,
    code_system     VARCHAR(255),
    code_value      VARCHAR(20) NOT NULL,
    code_display    VARCHAR(500),
    onset_date      DATE,
    is_sdoh         BOOLEAN DEFAULT false,
    sdoh_category   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_condition_patient ON condition(tenant_id, patient_id);

CREATE TABLE referral (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    referral_type   VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    reason_display  VARCHAR(500),
    requested_date  DATE NOT NULL,
    scheduled_date  DATE,
    completed_date  DATE,
    abandonment_risk_score DECIMAL(5,4),
    sdoh_category   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_referral_patient ON referral(tenant_id, patient_id);
CREATE INDEX idx_referral_status ON referral(tenant_id, status);

CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    care_plan_id    UUID REFERENCES care_plan(id),
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(20) DEFAULT 'routine',
    task_type       VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    due_date        DATE,
    completed_date  TIMESTAMPTZ,
    time_spent_min  INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_patient ON task(tenant_id, patient_id);

CREATE TABLE care_gap (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    measure_name    VARCHAR(500) NOT NULL,
    measure_set     VARCHAR(50),
    gap_status      VARCHAR(20) NOT NULL,
    identified_date DATE NOT NULL,
    due_date        DATE,
    closed_date     DATE,
    closure_method  VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_care_gap_patient ON care_gap(tenant_id, patient_id);
CREATE INDEX idx_care_gap_status ON care_gap(tenant_id, gap_status);

CREATE TABLE sdoh_screening (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    instrument      VARCHAR(50) NOT NULL,
    screening_date  TIMESTAMPTZ NOT NULL,
    identified_needs TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sdoh_screening_patient ON sdoh_screening(tenant_id, patient_id);
```

## Graph Layer Tables

```sql
-- ============================================================
-- PROPERTY GRAPH LAYER
-- Generic graph tables that model all relationships
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    node_type       VARCHAR(50) NOT NULL,    -- 'patient', 'provider', 'organisation', 'care_plan', 'care_team',
                                             -- 'referral', 'condition', 'care_gap', 'sdoh_screening',
                                             -- 'vbc_programme', 'quality_measure', 'cbo'
    entity_id       UUID NOT NULL,           -- FK to the relational table (patient.id, provider.id, etc.)
    label           VARCHAR(255),            -- human-readable label for visualisation
    properties      JSONB NOT NULL DEFAULT '{}',  -- denormalised properties for graph queries
    -- Example (patient node):
    -- {"name": "John Smith", "risk_tier": "high", "age": 67, "sdoh_needs": ["food_insecurity"]}
    -- Example (provider node):
    -- {"name": "Dr. Jane Doe", "npi": "1234567890", "specialty": "Internal Medicine"}
    -- Example (organisation node):
    -- {"name": "Austin Regional Medical Center", "type": "hospital", "npi": "9876543210"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_node_tenant ON graph_node(tenant_id, node_type);
CREATE UNIQUE INDEX idx_graph_node_entity ON graph_node(node_type, entity_id);
CREATE INDEX idx_graph_node_properties ON graph_node USING GIN (properties jsonb_path_ops);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(50) NOT NULL,    -- relationship type (see catalogue below)
    properties      JSONB NOT NULL DEFAULT '{}',  -- edge-specific metadata
    -- Example (care_team_member edge):
    -- {"role": "primary_care_physician", "start_date": "2025-01-01", "end_date": null}
    -- Example (referred_to edge):
    -- {"referral_id": "uuid-...", "referral_type": "specialty", "status": "active", "risk_score": 0.35}
    -- Example (attributed_to edge):
    -- {"programme": "MSSP ACO", "attribution_method": "claims_based", "start_date": "2026-01-01"}
    weight          DECIMAL(10,4),           -- optional edge weight for scoring algorithms
    valid_from      TIMESTAMPTZ,             -- temporal validity start
    valid_to        TIMESTAMPTZ,             -- temporal validity end (NULL = current)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_graph_edge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_graph_edge_tenant ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_graph_edge_type ON graph_edge(edge_type);
CREATE INDEX idx_graph_edge_temporal ON graph_edge(valid_from, valid_to) WHERE valid_to IS NULL;
CREATE INDEX idx_graph_edge_properties ON graph_edge USING GIN (properties jsonb_path_ops);
```

## Edge Type Catalogue

```sql
-- ============================================================
-- EDGE TYPE CATALOGUE
-- Documents all relationship types in the graph
-- ============================================================

-- Patient → Provider relationships
-- 'has_pcp'              patient → provider (primary care physician)
-- 'seen_by'              patient → provider (any encounter)
-- 'care_team_member'     patient → provider (care team membership, with role in properties)
-- 'referred_by'          patient → provider (referring provider on a referral)

-- Patient → Organisation relationships
-- 'attributed_to'        patient → organisation (VBC programme attribution)
-- 'enrolled_in'          patient → organisation (health plan enrollment)
-- 'receives_care_at'     patient → organisation (facility where care is delivered)

-- Provider → Organisation relationships
-- 'employed_by'          provider → organisation
-- 'affiliated_with'      provider → organisation (non-employment affiliation)
-- 'credentialed_at'      provider → organisation

-- Organisation → Organisation relationships
-- 'parent_of'            organisation → organisation (hierarchy)
-- 'part_of_network'      organisation → organisation (provider network membership)
-- 'contracts_with'       organisation → organisation (payer-provider contract)

-- Referral relationships
-- 'referred_to'          patient → organisation/provider (the referral target)
-- 'referral_from'        provider → referral (who initiated)
-- 'referral_for'         referral → patient (the subject)

-- SDOH relationships
-- 'screened_for'         patient → sdoh_screening
-- 'has_sdoh_need'        patient → condition (SDOH condition)
-- 'referred_to_cbo'      patient → organisation (CBO referral)
-- 'served_by_cbo'        patient → organisation (CBO confirmed service)

-- Care plan relationships
-- 'has_care_plan'        patient → care_plan
-- 'addresses'            care_plan → condition
-- 'managed_by'           care_plan → care_team
-- 'has_goal'             care_plan → goal (if goals are separate nodes)
-- 'assigned_task'        care_plan → task

-- Quality measure relationships
-- 'has_care_gap'         patient → care_gap
-- 'measured_by'          care_gap → quality_measure
-- 'assigned_to_close'    care_gap → provider (assigned care manager)

-- Programme relationships
-- 'participates_in'      organisation → vbc_programme
-- 'contracted_under'     provider → vbc_programme
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

## Example Graph Queries

```sql
-- ============================================================
-- 1. Find all providers who share patients with a given provider
--    (for care team coordination and network analysis)
-- ============================================================
WITH target_patients AS (
    SELECT DISTINCT e1.target_node_id AS patient_node_id
    FROM graph_edge e1
    JOIN graph_node n ON n.id = e1.source_node_id
    WHERE n.node_type = 'provider'
      AND n.entity_id = 'target-provider-uuid'
      AND e1.edge_type IN ('has_pcp', 'seen_by', 'care_team_member')
      AND (e1.valid_to IS NULL OR e1.valid_to > now())
)
SELECT DISTINCT
    prov_node.entity_id AS provider_id,
    prov_node.properties->>'name' AS provider_name,
    prov_node.properties->>'specialty' AS specialty,
    COUNT(DISTINCT tp.patient_node_id) AS shared_patient_count
FROM target_patients tp
JOIN graph_edge e2 ON e2.target_node_id = tp.patient_node_id
    AND e2.edge_type IN ('has_pcp', 'seen_by', 'care_team_member')
    AND (e2.valid_to IS NULL OR e2.valid_to > now())
JOIN graph_node prov_node ON prov_node.id = e2.source_node_id
    AND prov_node.node_type = 'provider'
    AND prov_node.entity_id != 'target-provider-uuid'
GROUP BY prov_node.entity_id, prov_node.properties->>'name', prov_node.properties->>'specialty'
ORDER BY shared_patient_count DESC;

-- ============================================================
-- 2. Trace the complete referral chain for a patient
--    (PCP → Specialist → Post-Acute → CBO)
-- ============================================================
WITH RECURSIVE referral_chain AS (
    -- Start: patient's PCP
    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        e.properties,
        1 AS depth,
        ARRAY[e.source_node_id] AS path
    FROM graph_edge e
    JOIN graph_node patient_node ON patient_node.id = e.source_node_id
        AND patient_node.node_type = 'patient'
        AND patient_node.entity_id = 'patient-uuid-here'
    WHERE e.edge_type IN ('referred_to', 'referred_to_cbo', 'served_by_cbo')
    
    UNION ALL
    
    -- Follow referral chain
    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        e.properties,
        rc.depth + 1,
        rc.path || e.source_node_id
    FROM graph_edge e
    JOIN referral_chain rc ON e.source_node_id = rc.target_node_id
    WHERE e.edge_type IN ('referred_to', 'referred_to_cbo', 'served_by_cbo')
      AND rc.depth < 6  -- max 6 hops
      AND NOT (e.source_node_id = ANY(rc.path))  -- prevent cycles
)
SELECT
    rc.depth,
    rc.edge_type,
    src.node_type AS from_type,
    src.properties->>'name' AS from_name,
    tgt.node_type AS to_type,
    tgt.properties->>'name' AS to_name,
    rc.properties AS edge_details
FROM referral_chain rc
JOIN graph_node src ON src.id = rc.source_node_id
JOIN graph_node tgt ON tgt.id = rc.target_node_id
ORDER BY rc.depth;

-- ============================================================
-- 3. Find provider network gaps: providers with high referral
--    abandonment risk who share no in-network connections
-- ============================================================
SELECT
    ref_edge.properties->>'referral_type' AS referral_type,
    src_node.properties->>'name' AS referring_provider,
    tgt_node.properties->>'name' AS target_org,
    (ref_edge.properties->>'risk_score')::decimal AS abandonment_risk,
    ref_edge.properties->>'status' AS referral_status
FROM graph_edge ref_edge
JOIN graph_node src_node ON src_node.id = ref_edge.source_node_id
    AND src_node.node_type = 'provider'
JOIN graph_node tgt_node ON tgt_node.id = ref_edge.target_node_id
    AND tgt_node.node_type = 'organisation'
WHERE ref_edge.tenant_id = 'tenant-uuid'
  AND ref_edge.edge_type = 'referred_to'
  AND (ref_edge.properties->>'risk_score')::decimal > 0.5
  AND NOT EXISTS (
      SELECT 1 FROM graph_edge network
      JOIN graph_node prov_org ON prov_org.id = network.source_node_id
          AND prov_org.entity_id = (
              SELECT organisation_id FROM provider WHERE id = src_node.entity_id
          )
      WHERE network.edge_type = 'part_of_network'
        AND network.target_node_id = tgt_node.id
  )
ORDER BY abandonment_risk DESC;

-- ============================================================
-- 4. SDOH closed-loop referral completeness:
--    Find patients with SDOH needs but no CBO confirmation
-- ============================================================
SELECT
    p_node.entity_id AS patient_id,
    p_node.properties->>'name' AS patient_name,
    need_edge.properties->>'sdoh_category' AS need_category,
    need_edge.created_at AS need_identified_date,
    CASE WHEN cbo_edge.id IS NOT NULL THEN 'referred' ELSE 'no_referral' END AS referral_status,
    CASE WHEN confirm_edge.id IS NOT NULL THEN 'confirmed' ELSE 'pending' END AS confirmation_status
FROM graph_node p_node
JOIN graph_edge need_edge ON need_edge.source_node_id = p_node.id
    AND need_edge.edge_type = 'has_sdoh_need'
    AND (need_edge.valid_to IS NULL)
LEFT JOIN graph_edge cbo_edge ON cbo_edge.source_node_id = p_node.id
    AND cbo_edge.edge_type = 'referred_to_cbo'
    AND cbo_edge.properties->>'sdoh_category' = need_edge.properties->>'sdoh_category'
LEFT JOIN graph_edge confirm_edge ON confirm_edge.source_node_id = p_node.id
    AND confirm_edge.edge_type = 'served_by_cbo'
    AND confirm_edge.properties->>'sdoh_category' = need_edge.properties->>'sdoh_category'
WHERE p_node.tenant_id = 'tenant-uuid'
  AND p_node.node_type = 'patient'
  AND confirm_edge.id IS NULL  -- no CBO confirmation yet
ORDER BY need_edge.created_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organisation | 2 | Standard relational |
| Patient | 1 | Core demographics |
| Provider | 1 | Standard relational |
| Care Plan | 1 | Standard relational |
| Clinical Data | 1 | Conditions |
| Referral | 1 | Standard relational |
| Task | 1 | Standard relational |
| Care Gap | 1 | Standard relational |
| SDOH | 1 | Screenings |
| Graph Layer | 2 | graph_node + graph_edge (the core differentiator) |
| Audit | 1 | Audit log |
| RBAC | 3 | Users, roles, user-role |
| **Total** | **16** | Plus graph edges scale to any number of relationship types |

---

## Key Design Decisions

1. **Graph in PostgreSQL, not a separate database** — Using `graph_node` and `graph_edge` tables in PostgreSQL rather than Neo4j or Amazon Neptune eliminates dual-write consistency issues, simplifies deployment, and leverages existing PostgreSQL operational tooling (backup, replication, RLS). Recursive CTEs handle multi-hop traversals. For organisations that later need dedicated graph performance, the graph tables can be replicated to a graph database via CDC.

2. **Generic graph tables with typed edges** — Rather than a separate table for each relationship type, all relationships are stored in `graph_edge` with an `edge_type` discriminator. This allows new relationship types to be added without schema changes and supports generic graph traversal algorithms. The edge type catalogue documents all known relationship types.

3. **Denormalised properties on nodes and edges** — Graph nodes carry denormalised properties (e.g., patient name, risk tier, provider specialty) so that graph queries can return useful results without joining back to the relational tables. These properties are updated when the source relational record changes. This trades storage for query performance.

4. **Temporal edges** — `valid_from` and `valid_to` on graph edges support temporal queries: "who was on this patient's care team in January 2026?" Current relationships have `valid_to IS NULL`. This is essential for care team evolution, provider affiliation changes, and programme attribution periods.

5. **Edge weights for scoring** — The `weight` column on graph edges supports network analysis algorithms: closeness centrality for provider influence, referral volume weighting, and risk score propagation. This enables AI features like network-based readmission risk (patients connected to high-readmission providers or organisations).

6. **Relational tables remain the system of record** — The graph layer is a relationship index, not the system of record. CRUD operations write to relational tables first, then update graph nodes/edges. If the graph layer becomes inconsistent, it can be rebuilt from relational data. This separation keeps the operational CRUD path simple while providing graph capabilities for analytics and AI.

7. **SDOH closed-loop as graph traversal** — The referral chain from SDOH screening to CBO confirmation is naturally modeled as a graph path: `patient --has_sdoh_need--> condition --referred_to_cbo--> cbo_org --served_by_cbo--> patient`. Finding incomplete chains (screening without CBO confirmation) is a simple graph query for missing edges.

8. **Provider network analysis** — The graph layer enables queries that are impractical in a pure relational model: "find all providers within 2 hops of Dr. Smith who accept Medicaid and have availability within 7 days." This supports the AI-driven referral matching feature (matching patients to optimal in-network specialists by relationships, wait time, and outcomes).
