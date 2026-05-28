# Care Coordination Platform -- Development Plan

> Project: Care Coordination Platform (Candidate #109)
> Created: 2026-05-25
> Status: Planning

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation -- Data Model, Auth, and Multi-Tenancy](#phase-1-foundation----data-model-auth-and-multi-tenancy)
5. [Phase 2: FHIR Interoperability Layer](#phase-2-fhir-interoperability-layer)
6. [Phase 3: Patient Data Aggregation and Clinical Data Pipeline](#phase-3-patient-data-aggregation-and-clinical-data-pipeline)
7. [Phase 4: Care Management Workflow](#phase-4-care-management-workflow)
8. [Phase 5: Risk Stratification and Care Gap Engine](#phase-5-risk-stratification-and-care-gap-engine)
9. [Phase 6: Patient Outreach and Engagement](#phase-6-patient-outreach-and-engagement)
10. [Phase 7: Referral Management and Transitions of Care](#phase-7-referral-management-and-transitions-of-care)
11. [Phase 8: SDOH Screening and Closed-Loop Referral](#phase-8-sdoh-screening-and-closed-loop-referral)
12. [Phase 9: Quality Measurement and Reporting](#phase-9-quality-measurement-and-reporting)
13. [Phase 10: AI Copilot and Autonomous Agents](#phase-10-ai-copilot-and-autonomous-agents)
14. [Phase 11: Billing, CCM/TCM, and Value-Based Care Programmes](#phase-11-billing-ccmtcm-and-value-based-care-programmes)
15. [Phase 12: Analytics Dashboards and Population Health Views](#phase-12-analytics-dashboards-and-population-health-views)
16. [Definition of Done (Global)](#definition-of-done-global)

---

## Technology Decisions

### Data Model: Hybrid Relational + JSONB (Data Model Suggestion 3)

**Rationale:** The hybrid model strikes the best balance for a multi-tenant SaaS care coordination platform. It uses typed relational columns for fields that appear in WHERE clauses, JOINs, and ORDER BY (patient demographics, care plan status, care gap lifecycle, risk tiers), while storing variable, programme-specific, and jurisdiction-specific data in JSONB `extensions` columns. This mirrors FHIR's own extensibility pattern and keeps the table count manageable (~18-25 core tables) while supporting diverse customer requirements (health plans, ACOs, health systems, physician groups) without per-client schema changes.

The fully normalised model (Suggestion 1, ~33+ tables) was rejected for excessive migration complexity and inflexibility to programme-specific fields. The event-sourced model (Suggestion 2) adds significant development complexity that is unnecessary for MVP; however, its audit-trail ideas are incorporated via an immutable audit_log table partitioned by month. The graph-relational model (Suggestion 4) adds powerful relationship queries but introduces a learning curve and complexity that can be deferred; provider network graph analysis is a post-MVP enhancement that can be layered on top of the hybrid model.

Selected elements from each suggestion are incorporated:
- **From Suggestion 1:** Explicit SDOH columns (`is_sdoh`, `sdoh_category`), care gap as first-class entity, referral with `abandonment_risk_score`, CCM/TCM billing log table.
- **From Suggestion 2:** Immutable audit log design with `patient_id` for HIPAA compliance, partitioned by month. Projection-style denormalised fields on patient for worklist queries (`risk_tier`, `composite_risk_score`, `active_sdoh_needs`).
- **From Suggestion 3:** Core JSONB extension pattern on all major entities, care team members as JSONB array, claim lines as JSONB array, SDOH screening responses as JSONB array, tenant configuration in JSONB.
- **From Suggestion 4:** Edge type catalogue concept adopted for documentation. Graph layer deferred to a post-MVP enhancement phase and can be added as two tables (`graph_node`, `graph_edge`) layered on top.

### Backend: TypeScript + Node.js (NestJS)

**Rationale:** NestJS provides a structured, enterprise-grade framework with built-in support for modular architecture, dependency injection, guards (for RBAC), interceptors (for audit logging), and OpenAPI/Swagger generation. TypeScript provides type safety across the FHIR data model. The Node.js ecosystem has mature libraries for FHIR (fhir.js, fhirclient), OAuth 2.0/SMART on FHIR, and ANSI X12 parsing. NestJS's module system maps naturally to the domain-driven boundaries of this platform (patients, care plans, referrals, quality measures).

### Database: PostgreSQL 16+

**Rationale:** PostgreSQL is the standard for HIPAA-compliant healthcare SaaS. It provides JSONB with GIN indexes for the hybrid model's extension columns, Row-Level Security (RLS) for database-enforced multi-tenancy, table partitioning for the audit log, and a mature ecosystem for managed deployment (AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL). HAPI FHIR's JPA server also runs on PostgreSQL, enabling a shared infrastructure layer.

### FHIR Server: HAPI FHIR (Apache 2.0)

**Rationale:** HAPI FHIR is the most widely used open-source FHIR server, licensed under Apache 2.0 with no IP concerns for commercial embedding. It provides a complete FHIR R4 server with US Core profile support, SMART on FHIR authorisation, and bulk data export. It will serve as the FHIR API layer for CMS-0057-F compliance while the NestJS application manages the operational workflow database. Bidirectional sync between the operational database and the HAPI FHIR server is maintained via `fhir_id` columns on each clinical entity.

### Frontend: React + Next.js

**Rationale:** React is the dominant frontend framework for healthcare SaaS (Epic's MyChart, Cerner's PowerChart, Salesforce Lightning). Next.js adds server-side rendering for initial load performance (important for care manager worklists with hundreds of patients), API routes for BFF (Backend for Frontend) patterns, and built-in support for authentication. A component library built on Radix UI (shadcn/ui) provides accessible, WCAG-compliant UI primitives.

### Message Queue: Apache Kafka (or AWS MSK / Confluent Cloud)

**Rationale:** Care coordination requires reliable asynchronous processing for: EHR data ingestion (HL7 v2 ADT feeds), claims data pipeline (X12 837/835), risk score computation, care gap identification, outreach campaign execution, and AI agent task queues. Kafka provides exactly-once delivery semantics, event replay for reprocessing, and the throughput needed for health plan-scale populations (millions of members).

### AI/ML: Python microservices + LLM API (Claude/GPT-4)

**Rationale:** Risk stratification models (readmission prediction, HCC risk adjustment) are best implemented in Python (scikit-learn, XGBoost, PyTorch) where the healthcare ML ecosystem is strongest. LLM features (care plan drafting, clinical summary generation, NLP extraction) use API-based LLMs (Claude, GPT-4) via a dedicated AI service gateway that enforces PHI handling policies. The AI services communicate with the NestJS backend via Kafka topics and REST APIs.

### Infrastructure: Kubernetes on AWS (EKS) or GCP (GKE)

**Rationale:** Healthcare SaaS requires SOC 2 Type II, HIPAA BAA, and data residency controls. Both AWS and GCP offer HIPAA-eligible managed Kubernetes with BAA coverage. Kubernetes provides the deployment flexibility for the mixed-language architecture (TypeScript backend, Python ML services, Java HAPI FHIR server) and supports auto-scaling for variable workloads (batch claims processing, real-time care gap computation).

### Authentication: SMART on FHIR (OAuth 2.0) + Auth0/Keycloak

**Rationale:** SMART on FHIR is mandated for EHR-embedded applications (Epic, Oracle Health sidebar launches). Auth0 or Keycloak provides the identity platform for standalone application access, care manager login, and patient portal authentication. Both support multi-tenancy, MFA (required by proposed HIPAA Security Rule NPRM), and SAML/OIDC federation for health system SSO.

---

## Project Structure

```
care-coordination-platform/
├── apps/
│   ├── api/                          # NestJS backend application
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/             # SMART on FHIR + OAuth 2.0 auth
│   │   │   │   ├── tenant/           # Multi-tenancy management
│   │   │   │   ├── patient/          # Patient demographics, identifiers
│   │   │   │   ├── provider/         # Provider registry, NPI lookup
│   │   │   │   ├── organisation/     # Organisation hierarchy
│   │   │   │   ├── care-team/        # Care team composition
│   │   │   │   ├── care-plan/        # Care plan CRUD, goals, activities
│   │   │   │   ├── condition/        # Clinical conditions (ICD-10, SNOMED)
│   │   │   │   ├── encounter/        # Encounters (ADT)
│   │   │   │   ├── observation/      # Labs, vitals, screenings
│   │   │   │   ├── task/             # Task management, worklist
│   │   │   │   ├── referral/         # Referral lifecycle management
│   │   │   │   ├── care-gap/         # Care gap identification & closure
│   │   │   │   ├── quality-measure/  # HEDIS, CMS measure definitions
│   │   │   │   ├── risk/             # Risk stratification scores
│   │   │   │   ├── outreach/         # Member outreach campaigns
│   │   │   │   ├── sdoh/             # SDOH screening & CBO referrals
│   │   │   │   ├── claim/            # Claims data ingestion
│   │   │   │   ├── billing/          # CCM/TCM billing, VBC programmes
│   │   │   │   ├── fhir-sync/        # Bidirectional FHIR server sync
│   │   │   │   ├── data-ingestion/   # HL7 v2, C-CDA, X12 pipelines
│   │   │   │   └── audit/            # HIPAA audit logging
│   │   │   ├── common/
│   │   │   │   ├── guards/           # RBAC guards, tenant guards
│   │   │   │   ├── interceptors/     # Audit, logging, tenant context
│   │   │   │   ├── decorators/       # @CurrentTenant, @CurrentUser
│   │   │   │   ├── filters/          # Exception filters
│   │   │   │   └── pipes/            # Validation pipes
│   │   │   ├── database/
│   │   │   │   ├── migrations/       # Knex/TypeORM migrations
│   │   │   │   ├── seeds/            # Reference data seeds
│   │   │   │   └── rls-policies/     # PostgreSQL RLS policy scripts
│   │   │   └── config/               # Environment configuration
│   │   └── test/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── e2e/
│   ├── web/                          # Next.js care manager web app
│   │   ├── src/
│   │   │   ├── app/                  # Next.js App Router pages
│   │   │   │   ├── (auth)/           # Login, SSO callback
│   │   │   │   ├── (dashboard)/      # Main layout
│   │   │   │   │   ├── worklist/     # Care manager worklist
│   │   │   │   │   ├── patients/     # Patient 360 views
│   │   │   │   │   ├── care-plans/   # Care plan management
│   │   │   │   │   ├── referrals/    # Referral management
│   │   │   │   │   ├── care-gaps/    # Care gap closure
│   │   │   │   │   ├── sdoh/         # SDOH screenings & referrals
│   │   │   │   │   ├── quality/      # Quality measure dashboards
│   │   │   │   │   ├── analytics/    # Population health analytics
│   │   │   │   │   └── admin/        # Tenant, user, role management
│   │   │   │   └── api/              # BFF API routes
│   │   │   ├── components/           # Shared UI components
│   │   │   ├── hooks/                # Custom React hooks
│   │   │   └── lib/                  # Utilities, API client
│   │   └── test/
│   ├── portal/                       # Next.js patient engagement portal
│   └── fhir-server/                  # HAPI FHIR JPA server (Docker)
├── services/
│   ├── risk-engine/                  # Python ML risk stratification
│   ├── nlp-service/                  # Python NLP (HCC coding, note extraction)
│   ├── ai-copilot/                   # LLM-powered care plan assistant
│   ├── data-pipeline/                # Kafka consumers (HL7 v2, X12, C-CDA)
│   └── outreach-engine/              # Outreach campaign execution
├── packages/
│   ├── shared-types/                 # Shared TypeScript type definitions
│   ├── fhir-models/                  # FHIR R4 resource type definitions
│   ├── ui-components/                # Shared React component library
│   └── validation/                   # Shared validation schemas (Zod)
├── infrastructure/
│   ├── terraform/                    # IaC for AWS/GCP
│   ├── kubernetes/                   # K8s manifests / Helm charts
│   ├── docker/                       # Dockerfiles
│   └── scripts/                      # Database setup, migration scripts
├── docs/
│   ├── architecture/                 # Architecture decision records
│   ├── api/                          # API documentation
│   ├── fhir-profiles/                # Custom FHIR profiles
│   └── compliance/                   # HIPAA, CMS-0057-F documentation
└── turbo.json                        # Turborepo configuration
```

---

## Phase Dependency Graph

```
Phase 1: Foundation (Data Model, Auth, Multi-Tenancy)
  |
  +---> Phase 2: FHIR Interoperability Layer
  |       |
  |       +---> Phase 3: Patient Data Aggregation & Clinical Data Pipeline
  |               |
  |               +---> Phase 4: Care Management Workflow
  |               |       |
  |               |       +---> Phase 6: Patient Outreach & Engagement
  |               |       |       |
  |               |       |       +---> Phase 10: AI Copilot & Autonomous Agents
  |               |       |
  |               |       +---> Phase 7: Referral Management & Transitions of Care
  |               |       |       |
  |               |       |       +---> Phase 8: SDOH Screening & Closed-Loop Referral
  |               |       |
  |               |       +---> Phase 11: Billing, CCM/TCM, VBC Programmes
  |               |
  |               +---> Phase 5: Risk Stratification & Care Gap Engine
  |                       |
  |                       +---> Phase 9: Quality Measurement & Reporting
  |                       |       |
  |                       |       +---> Phase 12: Analytics Dashboards & Population Health
  |                       |
  |                       +---> Phase 10: AI Copilot & Autonomous Agents
  |
  (Phase 12 depends on Phases 9, 11)
  (Phase 10 depends on Phases 5, 6)
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 4 -> Phase 7 -> Phase 8

**Parallel tracks after Phase 3:**
- Track A: Phase 4 -> Phase 6 -> Phase 10
- Track B: Phase 5 -> Phase 9 -> Phase 12
- Track C: Phase 4 -> Phase 7 -> Phase 8
- Track D: Phase 4 -> Phase 11

---

## Phase 1: Foundation -- Data Model, Auth, and Multi-Tenancy

**Goal:** Establish the core database schema, multi-tenant isolation, authentication/authorisation framework, HIPAA audit logging, and the NestJS application skeleton with CI/CD pipeline.

**Duration estimate:** 4-6 weeks

### Task 1.1: Monorepo Setup and Build Infrastructure

**What:** Initialise a Turborepo monorepo with the project structure defined above. Configure TypeScript, ESLint, Prettier, and Jest across all packages. Set up Docker Compose for local development (PostgreSQL 16, Kafka, Redis). Configure CI/CD pipeline (GitHub Actions) with lint, type-check, test, and build stages.

**Design:**

```typescript
// turbo.json
{
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false }
  }
}
```

```yaml
# docker-compose.yml (core services for local dev)
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: care_coordination
      POSTGRES_USER: app
      POSTGRES_PASSWORD: local_dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    ports: ["9092:9092"]
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
  
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

**Testing:**
- [ ] `turbo run build` completes without errors across all packages
- [ ] `turbo run test` runs Jest test suites in each package
- [ ] `turbo run lint` and `turbo run typecheck` pass
- [ ] `docker compose up` starts PostgreSQL, Kafka, and Redis
- [ ] NestJS application starts and responds to `GET /health` with `{ status: "ok" }`

---

### Task 1.2: Core Database Schema (Hybrid Relational + JSONB)

**What:** Implement the database migration scripts for all core tables from the hybrid data model. This includes: tenant, organisation, patient, provider, care_team, condition, encounter, observation, care_plan, task, sdoh_screening, referral, quality_measure, care_gap, claim, risk_score, audit_log, app_user. Configure PostgreSQL JSONB columns with GIN indexes on all extension/JSONB columns.

**Design:**

```typescript
// apps/api/src/database/migrations/001_core_schema.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  // Tenant table
  await knex.schema.createTable('tenant', (t) => {
    t.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    t.string('name', 255).notNullable();
    t.string('type', 50).notNullable(); // health_system, health_plan, aco, physician_group, cbo
    t.jsonb('config').notNullable().defaultTo('{}');
    t.boolean('is_active').notNullable().defaultTo(true);
    t.timestamps(true, true);
  });

  // Patient table with JSONB extensions
  await knex.schema.createTable('patient', (t) => {
    t.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    t.uuid('tenant_id').notNullable().references('id').inTable('tenant');
    t.string('fhir_id', 64);
    t.string('mrn', 50);
    t.string('first_name', 100).notNullable();
    t.string('last_name', 100).notNullable();
    t.date('date_of_birth').notNullable();
    t.string('gender', 20);
    t.string('sex_at_birth', 20);       // USCDI v3
    t.string('sexual_orientation', 50);  // USCDI v3
    t.string('gender_identity', 50);     // USCDI v3
    t.string('race', 50);
    t.string('ethnicity', 50);
    t.string('preferred_language', 10);
    t.jsonb('address');
    t.jsonb('phones');
    t.string('email', 255);
    t.jsonb('identifiers').notNullable().defaultTo('[]');
    t.string('risk_tier', 20);
    t.decimal('composite_risk_score', 10, 4);
    t.specificType('active_sdoh_needs', 'TEXT[]');
    t.jsonb('extensions').notNullable().defaultTo('{}');
    t.boolean('is_active').notNullable().defaultTo(true);
    t.timestamps(true, true);
  });
  
  // GIN indexes for JSONB columns
  await knex.raw(`
    CREATE INDEX idx_patient_identifiers ON patient USING GIN (identifiers jsonb_path_ops);
    CREATE INDEX idx_patient_extensions ON patient USING GIN (extensions jsonb_path_ops);
  `);
  
  // ... remaining tables follow the hybrid model schema
}
```

**Testing:**
- [ ] Migration runs successfully: `knex migrate:latest` completes without error
- [ ] Rollback works cleanly: `knex migrate:rollback` drops all tables
- [ ] All JSONB GIN indexes are created (verified via `\di` in psql)
- [ ] Insert sample data into each table with JSONB extensions; verify JSONB containment queries (`@>`) return correct results
- [ ] Verify all foreign key constraints enforce referential integrity (attempt invalid FK insert, confirm rejection)
- [ ] Verify CHECK constraints on enum columns (e.g., `care_plan.status` rejects invalid values)

---

### Task 1.3: Row-Level Security (Multi-Tenancy)

**What:** Implement PostgreSQL Row-Level Security (RLS) policies on all clinical tables. Every query must be scoped to the tenant identified in the connection context. The NestJS application sets `app.tenant_id` via `SET LOCAL` at the start of each request within a transaction.

**Design:**

```sql
-- apps/api/src/database/rls-policies/tenant_isolation.sql

-- Enable RLS on patient table
ALTER TABLE patient ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see patients in their tenant
CREATE POLICY tenant_isolation_patient ON patient
  USING (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);

-- Repeat for all clinical tables:
-- organisation, provider, care_team, condition, encounter, observation,
-- care_plan, task, sdoh_screening, referral, quality_measure, care_gap,
-- claim, risk_score, outreach, audit_log
```

```typescript
// apps/api/src/common/interceptors/tenant-context.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { DataSource } from 'typeorm';

@Injectable()
export class TenantContextInterceptor implements NestInterceptor {
  constructor(private dataSource: DataSource) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const tenantId = request.user?.tenantId;
    
    if (tenantId) {
      const queryRunner = this.dataSource.createQueryRunner();
      await queryRunner.query(`SET LOCAL app.tenant_id = '${tenantId}'`);
    }
    
    return next.handle();
  }
}
```

**Testing:**
- [ ] Create two tenants (Tenant A, Tenant B) with patients in each
- [ ] Query as Tenant A: only Tenant A patients returned
- [ ] Query as Tenant B: only Tenant B patients returned
- [ ] Attempt cross-tenant insert (Tenant A user inserts with Tenant B tenant_id): verify rejection by WITH CHECK policy
- [ ] Verify RLS is enabled on every clinical table: `SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public' AND rowsecurity = true`
- [ ] Performance test: verify RLS does not add more than 5% overhead to indexed queries on 100K patient dataset

---

### Task 1.4: Authentication and Authorisation (RBAC)

**What:** Implement OAuth 2.0 authentication with SMART on FHIR support. Build a role-based access control (RBAC) system with roles (admin, care_manager, provider, analyst, patient) and resource-level permissions. Integrate with Auth0 or Keycloak for identity management. Enforce MFA for all care manager and provider accounts (HIPAA Security Rule NPRM compliance).

**Design:**

```typescript
// apps/api/src/modules/auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;
    
    const request = context.switchToHttp().getRequest();
    const userRoles: Array<{ role: string; org_id: string }> = request.user?.roles || [];
    
    return requiredRoles.some(role => 
      userRoles.some(ur => ur.role === role)
    );
  }
}

// apps/api/src/modules/auth/strategies/smart-on-fhir.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-oauth2';

export class SmartOnFhirStrategy extends PassportStrategy(Strategy, 'smart-on-fhir') {
  constructor() {
    super({
      authorizationURL: '', // Dynamic per EHR
      tokenURL: '',         // Dynamic per EHR
      clientID: process.env.SMART_CLIENT_ID,
      scope: 'launch/patient openid fhirUser patient/*.read',
    });
  }
}
```

**Testing:**
- [ ] Login flow: user authenticates via OAuth 2.0, receives JWT with tenant_id and roles
- [ ] RBAC enforcement: care_manager role can access care plans; analyst role cannot create care plans
- [ ] Admin role can manage users and roles within their tenant; cannot access other tenants
- [ ] SMART on FHIR launch: simulated EHR launch sets patient context correctly
- [ ] MFA enforcement: login without MFA for care_manager role is rejected
- [ ] Token refresh: expired access token triggers refresh without user re-authentication
- [ ] Invalid token: request with tampered JWT returns 401

---

### Task 1.5: HIPAA Audit Logging

**What:** Implement an immutable audit log that captures every data access and modification. Every API endpoint that touches PHI must generate an audit event. The audit log table is partitioned by month for retention management and query performance. Audit records include: tenant_id, user_id, action, resource_type, resource_id, patient_id (for PHI tracking), IP address, user agent, and a JSONB details field.

**Design:**

```typescript
// apps/api/src/common/interceptors/audit-log.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { tap } from 'rxjs/operators';

@Injectable()
export class AuditLogInterceptor implements NestInterceptor {
  constructor(private auditService: AuditService) {}

  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();

    return next.handle().pipe(
      tap({
        next: (data) => {
          this.auditService.log({
            tenantId: request.user?.tenantId,
            userId: request.user?.id,
            action: this.mapHttpMethodToAction(request.method),
            resourceType: context.getClass().name.replace('Controller', ''),
            resourceId: request.params?.id,
            patientId: data?.patientId || request.params?.patientId,
            ipAddress: request.ip,
            userAgent: request.headers['user-agent'],
            details: {
              method: request.method,
              path: request.path,
              statusCode: context.switchToHttp().getResponse().statusCode,
              durationMs: Date.now() - startTime,
            },
          });
        },
      }),
    );
  }

  private mapHttpMethodToAction(method: string): string {
    const mapping: Record<string, string> = {
      GET: 'read', POST: 'create', PUT: 'update',
      PATCH: 'update', DELETE: 'delete',
    };
    return mapping[method] || 'unknown';
  }
}
```

```sql
-- Partitioned audit log
CREATE TABLE audit_log (
    id              UUID DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    patient_id      UUID,
    ip_address      INET,
    user_agent      VARCHAR(500),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE audit_log_2026_06 PARTITION OF audit_log
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
```

**Testing:**
- [ ] Every API call to a patient endpoint generates an audit record with correct patient_id
- [ ] Audit records are immutable: UPDATE and DELETE on audit_log are rejected (via trigger or permissions)
- [ ] Audit log query by patient_id returns all access events for that patient (HIPAA accounting of disclosures)
- [ ] Audit log query by user_id returns all actions by that user
- [ ] Monthly partition exists for current month; verify partition pruning via EXPLAIN ANALYZE
- [ ] Audit log write does not add more than 10ms latency to API responses (async write via Kafka or queue)
- [ ] Audit records include IP address and user agent for forensic investigation

---

### Task 1.6: API Error Handling and Validation Framework

**What:** Implement global exception filters, request validation (Zod schemas in the shared `validation` package), and standardised error response format aligned with FHIR OperationOutcome for FHIR-facing APIs and RFC 7807 Problem Details for non-FHIR APIs.

**Design:**

```typescript
// packages/validation/src/patient.schema.ts
import { z } from 'zod';

export const CreatePatientSchema = z.object({
  firstName: z.string().min(1).max(100),
  lastName: z.string().min(1).max(100),
  dateOfBirth: z.string().date(),
  gender: z.enum(['male', 'female', 'other', 'unknown']).optional(),
  mrn: z.string().max(50).optional(),
  identifiers: z.array(z.object({
    system: z.string().url(),
    value: z.string().min(1),
    type: z.string().optional(),
  })).default([]),
  extensions: z.record(z.unknown()).default({}),
});

// apps/api/src/common/filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    
    if (exception instanceof ZodError) {
      return response.status(400).json({
        type: 'https://care-coord.dev/errors/validation',
        title: 'Validation Error',
        status: 400,
        detail: 'Request body failed validation',
        errors: exception.issues.map(i => ({
          field: i.path.join('.'),
          message: i.message,
        })),
      });
    }
    // ... other exception types
  }
}
```

**Testing:**
- [ ] Invalid patient creation (missing `lastName`) returns 400 with field-specific error
- [ ] Invalid date format returns 400 with descriptive message
- [ ] FHIR endpoints return FHIR OperationOutcome on error
- [ ] Non-FHIR endpoints return RFC 7807 Problem Details
- [ ] Unhandled exceptions return 500 with generic message (no stack traces or internal details in production)
- [ ] Error responses never leak PHI in error messages

---

### Phase 1 Definition of Done

- [ ] Monorepo builds, lints, and passes all tests in CI
- [ ] PostgreSQL schema deployed with all core tables and JSONB GIN indexes
- [ ] RLS policies active on all clinical tables; cross-tenant access verified as impossible
- [ ] OAuth 2.0 + SMART on FHIR authentication working end-to-end
- [ ] RBAC guards enforce role-based access on all endpoints
- [ ] Audit log captures every PHI access with patient_id, user_id, IP address
- [ ] Zod validation on all request bodies; standardised error responses
- [ ] Docker Compose local development environment starts in under 60 seconds
- [ ] CI pipeline runs in under 10 minutes

---

## Phase 2: FHIR Interoperability Layer

**Goal:** Deploy a HAPI FHIR R4 server and build the bidirectional sync layer between the operational database and the FHIR server. Implement SMART on FHIR authorisation for the FHIR API. Establish FHIR resource profiles for US Core and Gravity SDOH IG compliance.

**Duration estimate:** 4-5 weeks

### Task 2.1: HAPI FHIR Server Deployment

**What:** Deploy HAPI FHIR JPA Server Starter as a Docker container with PostgreSQL backend. Configure US Core STU8 profiles, Gravity SDOH Clinical Care IG profiles, and Da Vinci CDex profiles. Enable SMART on FHIR authorisation interceptor. Configure FHIR Bulk Data Export (for population-level data exchange).

**Design:**

```yaml
# apps/fhir-server/Dockerfile
FROM hapiproject/hapi:v7.4.0
COPY application.yaml /app/config/application.yaml

# apps/fhir-server/application.yaml
hapi:
  fhir:
    server_address: https://fhir.carecoord.example.com/fhir
    fhir_version: R4
    default_encoding: JSON
    allow_external_references: true
    smart:
      enabled: true
      openid_configuration_url: ${AUTH_ISSUER}/.well-known/openid-configuration
    bulk_export:
      enabled: true
    implementationguides:
      us_core:
        name: hl7.fhir.us.core
        version: 8.0.0
      sdoh:
        name: hl7.fhir.us.sdoh-clinicalcare
        version: 2.2.0
      davinci_cdex:
        name: hl7.fhir.us.davinci-cdex
        version: 2.1.0
```

**Testing:**
- [ ] HAPI FHIR server starts and responds to `GET /fhir/metadata` with CapabilityStatement
- [ ] CapabilityStatement includes US Core, Gravity SDOH, and Da Vinci CDex profiles
- [ ] FHIR CRUD: create, read, update, delete a Patient resource via REST API
- [ ] SMART on FHIR: unauthenticated FHIR request returns 401
- [ ] SMART on FHIR: authenticated request with valid token returns FHIR resources
- [ ] Bulk Data Export: `GET /fhir/$export` initiates export; poll returns ndjson files
- [ ] FHIR validation: resource missing a US Core required field is rejected with OperationOutcome

---

### Task 2.2: FHIR Resource Mapper (Operational DB <-> FHIR)

**What:** Build a bidirectional mapper service that converts between operational database records and FHIR R4 resources. Each major entity (patient, condition, observation, care_plan, care_team, goal, task, encounter) has a mapper class. The mapper handles JSONB extensions by mapping them to FHIR extension elements. The `fhir_id` column on each table links the operational record to its FHIR resource counterpart.

**Design:**

```typescript
// apps/api/src/modules/fhir-sync/mappers/patient.mapper.ts
import { Patient as FhirPatient } from '@care-coord/fhir-models';

export class PatientFhirMapper {
  toFhir(dbPatient: PatientEntity): FhirPatient {
    return {
      resourceType: 'Patient',
      id: dbPatient.fhirId,
      identifier: dbPatient.identifiers.map(id => ({
        system: id.system,
        value: id.value,
        type: { coding: [{ code: id.type }] },
      })),
      name: [{
        family: dbPatient.lastName,
        given: [dbPatient.firstName],
      }],
      birthDate: dbPatient.dateOfBirth,
      gender: dbPatient.gender as 'male' | 'female' | 'other' | 'unknown',
      address: dbPatient.address ? [{
        line: [dbPatient.address.line1, dbPatient.address.line2].filter(Boolean),
        city: dbPatient.address.city,
        state: dbPatient.address.state,
        postalCode: dbPatient.address.postal_code,
        country: dbPatient.address.country,
      }] : [],
      extension: this.mapExtensions(dbPatient.extensions),
    };
  }

  fromFhir(fhirPatient: FhirPatient): Partial<PatientEntity> {
    return {
      fhirId: fhirPatient.id,
      firstName: fhirPatient.name?.[0]?.given?.[0] || '',
      lastName: fhirPatient.name?.[0]?.family || '',
      dateOfBirth: fhirPatient.birthDate,
      gender: fhirPatient.gender,
      identifiers: (fhirPatient.identifier || []).map(id => ({
        system: id.system || '',
        value: id.value || '',
        type: id.type?.coding?.[0]?.code,
      })),
    };
  }

  private mapExtensions(extensions: Record<string, unknown>): FhirExtension[] {
    return Object.entries(extensions).map(([key, value]) => ({
      url: `https://carecoord.dev/fhir/StructureDefinition/${key}`,
      valueString: String(value),
    }));
  }
}
```

**Testing:**
- [ ] Round-trip: DB record -> FHIR resource -> DB record preserves all core fields
- [ ] FHIR Patient includes all US Core mandatory elements (name, gender, birthDate, identifier)
- [ ] FHIR Patient extensions include USCDI v3 fields (sex_at_birth, sexual_orientation, gender_identity)
- [ ] Care plan mapper produces valid FHIR CarePlan with goals and activities
- [ ] SDOH condition mapper sets Gravity IG category codes correctly
- [ ] Mapper handles null/empty JSONB extensions gracefully
- [ ] FHIR validation: mapped resources pass HAPI FHIR validation against US Core profiles

---

### Task 2.3: FHIR Sync Service (Bidirectional)

**What:** Build a sync service that maintains consistency between the operational database and the HAPI FHIR server. When a record is created or updated in the operational database, the corresponding FHIR resource is created/updated on the FHIR server. When a FHIR resource is received from an external system (via API or subscription), the operational database is updated. Sync events flow through Kafka for reliability.

**Design:**

```typescript
// apps/api/src/modules/fhir-sync/fhir-sync.service.ts
@Injectable()
export class FhirSyncService {
  constructor(
    private fhirClient: FhirClient,
    private kafkaProducer: KafkaProducer,
    private patientMapper: PatientFhirMapper,
  ) {}

  async syncPatientToFhir(patient: PatientEntity): Promise<void> {
    const fhirPatient = this.patientMapper.toFhir(patient);
    
    if (patient.fhirId) {
      await this.fhirClient.update('Patient', patient.fhirId, fhirPatient);
    } else {
      const created = await this.fhirClient.create('Patient', fhirPatient);
      // Update operational DB with the FHIR-assigned ID
      await this.kafkaProducer.send('fhir.sync.id-update', {
        entityType: 'patient',
        entityId: patient.id,
        fhirId: created.id,
      });
    }
  }

  async syncFhirToOperational(fhirResource: FhirResource): Promise<void> {
    const mapper = this.getMapper(fhirResource.resourceType);
    const dbEntity = mapper.fromFhir(fhirResource);
    
    await this.kafkaProducer.send('fhir.sync.inbound', {
      resourceType: fhirResource.resourceType,
      fhirId: fhirResource.id,
      entity: dbEntity,
    });
  }
}
```

**Testing:**
- [ ] Create patient in operational DB -> FHIR Patient created on HAPI server with matching data
- [ ] Update patient in operational DB -> FHIR Patient updated on HAPI server
- [ ] Receive FHIR Patient from external system -> operational DB patient record created
- [ ] `fhir_id` column is populated after initial sync
- [ ] Sync failure (HAPI server down) -> event queued in Kafka; retried on recovery
- [ ] Duplicate sync protection: same record synced twice does not create duplicate FHIR resources
- [ ] Performance: sync of 1000 patients completes within 30 seconds (batched)

---

### Phase 2 Definition of Done

- [ ] HAPI FHIR R4 server deployed, accessible, and returning valid CapabilityStatement
- [ ] US Core STU8, Gravity SDOH IG, and Da Vinci CDex profiles loaded and active
- [ ] SMART on FHIR authorisation enforced on all FHIR endpoints
- [ ] Bidirectional sync operational for Patient, Condition, Observation, CarePlan, CareTeam, Encounter resources
- [ ] All mapped FHIR resources pass US Core profile validation
- [ ] Bulk Data Export functional for population-level queries
- [ ] FHIR sync events flow through Kafka with at-least-once delivery

---

## Phase 3: Patient Data Aggregation and Clinical Data Pipeline

**Goal:** Build the data ingestion pipelines for HL7 v2 ADT messages, C-CDA documents, ANSI X12 claims data, and FHIR-based clinical data exchange. Establish the multi-source patient data aggregation that is the foundation of all downstream care coordination workflows.

**Duration estimate:** 5-7 weeks

### Task 3.1: HL7 v2 ADT Message Consumer

**What:** Build a Kafka consumer that receives HL7 v2 ADT (Admit/Discharge/Transfer) messages from hospital interfaces, parses them into structured data, and upserts patient demographics, encounters, and discharge events into the operational database. ADT messages are the primary real-time feed for knowing when patients are admitted, discharged, or transferred.

**Design:**

```typescript
// services/data-pipeline/src/consumers/hl7v2-adt.consumer.ts
import { Hl7Message } from 'hl7-standard';

export class Hl7v2AdtConsumer {
  async processMessage(rawHl7: string): Promise<void> {
    const msg = new Hl7Message(rawHl7);
    const messageType = msg.get('MSH.9.2'); // ADT event type: A01, A03, A04, etc.
    
    const patientData = {
      mrn: msg.get('PID.3.1'),
      lastName: msg.get('PID.5.1'),
      firstName: msg.get('PID.5.2'),
      dateOfBirth: this.parseHl7Date(msg.get('PID.7')),
      gender: this.mapGender(msg.get('PID.8')),
      address: {
        line1: msg.get('PID.11.1'),
        city: msg.get('PID.11.3'),
        state: msg.get('PID.11.4'),
        postal_code: msg.get('PID.11.5'),
      },
      phones: [{ type: 'home', number: msg.get('PID.13.1') }],
    };

    const encounterData = {
      classCode: this.mapEncounterClass(messageType),
      status: this.mapEncounterStatus(messageType),
      periodStart: this.parseHl7Date(msg.get('PV1.44')),
      dischargeDisposition: msg.get('PV1.36.1'),
    };

    await this.patientService.upsertByMrn(patientData);
    await this.encounterService.processAdtEvent(messageType, encounterData);
  }
}
```

**Testing:**
- [ ] A01 (Admit) message creates a new encounter with status 'in-progress' and class 'IMP'
- [ ] A03 (Discharge) message updates encounter status to 'finished' and sets discharge disposition
- [ ] A04 (Register Outpatient) creates encounter with class 'AMB'
- [ ] Patient demographics updated from PID segment on each ADT message
- [ ] Duplicate MRN handling: second message for same MRN updates existing patient, not duplicate
- [ ] Malformed HL7 message: logged to dead-letter queue, not crash the consumer
- [ ] Processing throughput: 500 ADT messages/second sustained for 5 minutes

---

### Task 3.2: C-CDA Document Ingestion

**What:** Build a parser for C-CDA R2.1/R3.0 documents (Continuity of Care Documents, Referral Notes, Discharge Summaries). Extract structured clinical data (conditions, medications, allergies, labs, vitals, care teams) and store in the operational database. C-CDA documents are the primary format for care transition summaries from referring providers.

**Design:**

```typescript
// services/data-pipeline/src/parsers/ccda-parser.ts
import { DOMParser } from 'xmldom';
import * as xpath from 'xpath';

export class CcdaParser {
  private namespaces = {
    cda: 'urn:hl7-org:v3',
    sdtc: 'urn:hl7-org:sdtc',
  };

  parse(xmlContent: string): CcdaParsedDocument {
    const doc = new DOMParser().parseFromString(xmlContent);
    
    return {
      patient: this.extractPatient(doc),
      conditions: this.extractConditions(doc),
      medications: this.extractMedications(doc),
      allergies: this.extractAllergies(doc),
      results: this.extractLabResults(doc),
      vitalSigns: this.extractVitalSigns(doc),
      encounters: this.extractEncounters(doc),
      careTeamMembers: this.extractCareTeam(doc),
      dischargeSummary: this.extractDischargeSummary(doc),
    };
  }

  private extractConditions(doc: Document): ParsedCondition[] {
    const nodes = xpath.select(
      '//cda:section[cda:templateId[@root="2.16.840.1.113883.10.20.22.2.5.1"]]/cda:entry/cda:act/cda:entryRelationship/cda:observation',
      doc, this.namespaces
    );
    
    return nodes.map(node => ({
      code: xpath.select1('cda:value/@code', node)?.value,
      codeSystem: xpath.select1('cda:value/@codeSystem', node)?.value,
      displayName: xpath.select1('cda:value/@displayName', node)?.value,
      onsetDate: xpath.select1('cda:effectiveTime/cda:low/@value', node)?.value,
      status: this.mapConditionStatus(xpath.select1('cda:statusCode/@code', node)?.value),
    }));
  }
}
```

**Testing:**
- [ ] Parse a valid CCD: all conditions, medications, allergies, labs extracted correctly
- [ ] Parse a Referral Note: referring provider, reason for referral, care team extracted
- [ ] Parse a Discharge Summary: discharge diagnosis, disposition, follow-up instructions extracted
- [ ] ICD-10-CM codes from condition entries mapped correctly to the condition table
- [ ] LOINC codes from lab results mapped correctly to the observation table
- [ ] Invalid/missing XML elements handled gracefully (logged, not crashed)
- [ ] SNOMED CT to ICD-10-CM cross-mapping applied where possible

---

### Task 3.3: ANSI X12 Claims Data Pipeline

**What:** Build a batch pipeline to ingest ANSI X12 837 (claims) and 835 (remittance) files. Parse claims into the claim table with claim lines stored as JSONB arrays. Claims data provides utilisation history, diagnosis codes, procedure codes, and financial data for risk stratification and care gap identification.

**Design:**

```typescript
// services/data-pipeline/src/parsers/x12-parser.ts
export class X12ClaimParser {
  parse837(content: string): ParsedClaim[] {
    const segments = content.split('~').map(s => s.trim()).filter(Boolean);
    const claims: ParsedClaim[] = [];
    let currentClaim: Partial<ParsedClaim> = {};
    let currentLines: ParsedClaimLine[] = [];

    for (const segment of segments) {
      const elements = segment.split('*');
      const segmentId = elements[0];

      switch (segmentId) {
        case 'CLM': // Claim header
          if (currentClaim.claimNumber) {
            claims.push({ ...currentClaim, lines: currentLines } as ParsedClaim);
          }
          currentClaim = {
            claimNumber: elements[1],
            billedAmount: parseFloat(elements[2]),
            claimType: this.determineClaimType(elements),
          };
          currentLines = [];
          break;
        case 'SV1': // Professional service line
        case 'SV2': // Institutional service line
          currentLines.push(this.parseServiceLine(elements));
          break;
        case 'DTP': // Date/time period
          this.applyDateToContext(elements, currentClaim, currentLines);
          break;
        case 'HI': // Health care diagnosis
          currentClaim.primaryDxCode = elements[1]?.split(':')[1];
          break;
      }
    }
    
    return claims;
  }
}
```

**Testing:**
- [ ] Parse an 837P (Professional) file: claims and service lines extracted with correct CPT codes
- [ ] Parse an 837I (Institutional) file: claims with DRG codes, admission/discharge dates extracted
- [ ] Parse an 835 file: payment amounts mapped to existing claims
- [ ] Claim lines stored as JSONB array in the claim table with correct structure
- [ ] ICD-10 diagnosis codes extracted from HI segments
- [ ] Patient matching: claims matched to existing patients by member ID in identifiers JSONB
- [ ] Batch performance: 100K claims processed within 10 minutes

---

### Task 3.4: Patient Master Data Management (EMPI-Lite)

**What:** Build a lightweight enterprise master patient index (EMPI) that deduplicates patient records received from multiple sources (EHR ADT feeds, C-CDA documents, claims files, FHIR APIs). Matching uses deterministic rules (MRN + source system, Medicare ID, SSN last-4 + DOB + last name) and probabilistic scoring (Jaro-Winkler on name, address similarity). Matched records are linked via the `identifiers` JSONB array.

**Design:**

```typescript
// apps/api/src/modules/patient/services/patient-matching.service.ts
export class PatientMatchingService {
  async findOrCreatePatient(
    tenantId: string,
    incomingData: IncomingPatientData,
  ): Promise<{ patient: PatientEntity; matchType: 'exact' | 'probable' | 'new' }> {
    
    // Stage 1: Deterministic match on identifiers
    for (const id of incomingData.identifiers) {
      const match = await this.patientRepo.findByIdentifier(tenantId, id.system, id.value);
      if (match) return { patient: match, matchType: 'exact' };
    }

    // Stage 2: Probabilistic match
    const candidates = await this.patientRepo.findCandidates(tenantId, {
      lastName: incomingData.lastName,
      dateOfBirth: incomingData.dateOfBirth,
    });

    for (const candidate of candidates) {
      const score = this.calculateMatchScore(candidate, incomingData);
      if (score >= 0.95) return { patient: candidate, matchType: 'exact' };
      if (score >= 0.80) return { patient: candidate, matchType: 'probable' };
    }

    // Stage 3: Create new patient
    const newPatient = await this.patientRepo.create(tenantId, incomingData);
    return { patient: newPatient, matchType: 'new' };
  }

  private calculateMatchScore(existing: PatientEntity, incoming: IncomingPatientData): number {
    let score = 0;
    const weights = { lastName: 0.25, firstName: 0.20, dob: 0.30, gender: 0.05, address: 0.10, phone: 0.10 };

    score += weights.lastName * jaroWinkler(existing.lastName, incoming.lastName);
    score += weights.firstName * jaroWinkler(existing.firstName, incoming.firstName);
    score += weights.dob * (existing.dateOfBirth === incoming.dateOfBirth ? 1 : 0);
    score += weights.gender * (existing.gender === incoming.gender ? 1 : 0);
    // ... address and phone scoring
    
    return score;
  }
}
```

**Testing:**
- [ ] Exact match: patient with same MRN+source returns existing record
- [ ] Exact match: patient with same Medicare ID returns existing record
- [ ] Probabilistic match: same last name, DOB, and similar address returns probable match (score >= 0.80)
- [ ] No match: entirely new patient creates a new record
- [ ] Merge identifiers: matched patient's identifiers JSONB array updated with new identifier
- [ ] Case insensitivity: "SMITH" matches "Smith"
- [ ] Edge case: hyphenated last names matched correctly (Jaro-Winkler handles)
- [ ] Performance: matching against 500K patients completes within 200ms per lookup

---

### Phase 3 Definition of Done

- [ ] HL7 v2 ADT messages processed in real-time; patient demographics and encounters updated
- [ ] C-CDA documents parsed; conditions, medications, allergies, labs, vitals extracted and stored
- [ ] ANSI X12 837/835 claims ingested in batch; claims linked to patients via identifier matching
- [ ] Patient matching service deduplicates across data sources with >= 95% accuracy
- [ ] All ingested data synced to HAPI FHIR server via Phase 2 sync layer
- [ ] Data pipeline observability: Kafka consumer lag, parse error counts, match rates monitored
- [ ] Dead-letter queues for unparseable messages with alerting

---

## Phase 4: Care Management Workflow

**Goal:** Build the core care management experience: care plan creation and management, task management, care manager worklist with prioritised patient queue, and care team collaboration.

**Duration estimate:** 5-6 weeks

### Task 4.1: Care Plan CRUD and Lifecycle

**What:** Implement care plan creation, update, and lifecycle management. A care plan addresses one or more conditions, contains goals (with targets and achievement status), and contains activities (tasks, referrals, medication requests). Care plans follow the FHIR CarePlan lifecycle (draft -> active -> on-hold -> completed/revoked). Goals and activities are stored as JSONB arrays within the care plan record (per the hybrid model). Care plan versioning tracks changes.

**Design:**

```typescript
// apps/api/src/modules/care-plan/care-plan.service.ts
@Injectable()
export class CarePlanService {
  async create(tenantId: string, dto: CreateCarePlanDto): Promise<CarePlanEntity> {
    const carePlan = await this.carePlanRepo.create({
      tenantId,
      patientId: dto.patientId,
      careTeamId: dto.careTeamId,
      status: 'draft',
      intent: 'plan',
      title: dto.title,
      description: dto.description,
      periodStart: dto.periodStart,
      periodEnd: dto.periodEnd,
      authorId: dto.authorId,
      addresses: dto.conditionIds.map(id => ({ condition_id: id })),
      goals: dto.goals.map(g => ({
        id: randomUUID(),
        status: 'proposed',
        description: g.description,
        targetDate: g.targetDate,
        targetValue: g.targetValue,
        targetUnit: g.targetUnit,
        achievement: 'in-progress',
        isSdoh: g.isSdoh || false,
        sdohCategory: g.sdohCategory,
      })),
      activities: [],
      version: 1,
    });

    await this.fhirSyncService.syncCarePlanToFhir(carePlan);
    await this.auditService.log('create', 'care_plan', carePlan.id, dto.patientId);
    
    return carePlan;
  }

  async activate(carePlanId: string): Promise<CarePlanEntity> {
    const carePlan = await this.carePlanRepo.findById(carePlanId);
    if (carePlan.status !== 'draft') {
      throw new BadRequestException('Only draft care plans can be activated');
    }
    
    return this.carePlanRepo.update(carePlanId, {
      status: 'active',
      version: carePlan.version + 1,
    });
  }

  async updateGoalAchievement(
    carePlanId: string,
    goalId: string,
    achievement: string,
  ): Promise<CarePlanEntity> {
    const carePlan = await this.carePlanRepo.findById(carePlanId);
    const goals = carePlan.goals.map(g =>
      g.id === goalId ? { ...g, achievement } : g
    );
    
    return this.carePlanRepo.update(carePlanId, {
      goals,
      version: carePlan.version + 1,
    });
  }
}
```

**Testing:**
- [ ] Create care plan with conditions, goals, and activities; verify all stored correctly
- [ ] Lifecycle: draft -> active -> on-hold -> active -> completed transitions work
- [ ] Invalid transition (completed -> active) rejected with 400
- [ ] Goal update: achievement status change increments care plan version
- [ ] Add activity to active care plan: activity appears in activities JSONB array
- [ ] FHIR sync: care plan creation triggers FHIR CarePlan resource creation on HAPI server
- [ ] Care plan list by patient: returns all care plans with status, goal counts, activity counts
- [ ] Concurrent update: optimistic locking prevents lost updates (version check)

---

### Task 4.2: Task Management and Assignment

**What:** Implement the task lifecycle (draft -> requested -> accepted -> in-progress -> completed). Tasks are linked to care plans and patients. Care managers are assigned as task owners. Tasks support time tracking for CCM/TCM billing documentation. Task types include: outreach_call, care_gap_closure, sdoh_referral, med_reconciliation, follow_up, documentation, review.

**Design:**

```typescript
// apps/api/src/modules/task/task.service.ts
@Injectable()
export class TaskService {
  async createTask(tenantId: string, dto: CreateTaskDto): Promise<TaskEntity> {
    return this.taskRepo.create({
      tenantId,
      patientId: dto.patientId,
      carePlanId: dto.carePlanId,
      status: 'requested',
      priority: dto.priority || 'routine',
      taskType: dto.taskType,
      description: dto.description,
      requesterId: dto.requesterId,
      ownerId: dto.assignedToId,
      dueDate: dto.dueDate,
      taskData: dto.taskData || {},
    });
  }

  async logTime(taskId: string, minutes: number, notes: string): Promise<TaskEntity> {
    const task = await this.taskRepo.findById(taskId);
    return this.taskRepo.update(taskId, {
      timeSpentMin: task.timeSpentMin + minutes,
      taskData: {
        ...task.taskData,
        timeEntries: [
          ...(task.taskData.timeEntries || []),
          { minutes, notes, loggedAt: new Date().toISOString(), loggedBy: 'current-user-id' },
        ],
      },
    });
  }
}
```

**Testing:**
- [ ] Create task assigned to care manager: appears in their worklist
- [ ] Task lifecycle: requested -> accepted -> in-progress -> completed
- [ ] Time logging: 15 minutes logged; task.timeSpentMin updated to 15; time entry in taskData
- [ ] Task priority ordering: stat > asap > urgent > routine
- [ ] Overdue task detection: tasks past due_date flagged in worklist query
- [ ] Task reassignment: ownerId changed; task moves to new care manager's worklist
- [ ] Task linked to care plan: appears in care plan activities

---

### Task 4.3: Care Manager Worklist

**What:** Build the care manager worklist -- the primary UI for care managers. It shows a prioritised list of patients assigned to the care manager, with risk tier, open care gaps, open tasks, next task due date, and SDOH flags. Patients are ranked by composite risk score and urgency (overdue tasks, stat priority). The worklist supports filtering by risk tier, SDOH needs, programme, and care gap status.

**Design:**

```typescript
// apps/api/src/modules/task/worklist.service.ts
@Injectable()
export class WorklistService {
  async getWorklist(tenantId: string, careManagerId: string, filters: WorklistFilters): Promise<WorklistItem[]> {
    const query = this.db('patient')
      .select(
        'patient.id', 'patient.first_name', 'patient.last_name',
        'patient.date_of_birth', 'patient.risk_tier', 'patient.composite_risk_score',
        'patient.active_sdoh_needs',
        this.db.raw(`(SELECT COUNT(*) FROM care_gap WHERE care_gap.patient_id = patient.id AND care_gap.gap_status = 'open') AS open_care_gaps`),
        this.db.raw(`(SELECT COUNT(*) FROM task WHERE task.patient_id = patient.id AND task.owner_id = ? AND task.status NOT IN ('completed', 'cancelled')) AS open_tasks`, [careManagerId]),
        this.db.raw(`(SELECT MIN(due_date) FROM task WHERE task.patient_id = patient.id AND task.owner_id = ? AND task.status NOT IN ('completed', 'cancelled')) AS next_due_date`, [careManagerId]),
      )
      .where('patient.tenant_id', tenantId)
      .whereExists(function() {
        this.select('*').from('task')
          .whereRaw('task.patient_id = patient.id')
          .where('task.owner_id', careManagerId)
          .whereNotIn('task.status', ['completed', 'cancelled']);
      });

    if (filters.riskTier) query.where('patient.risk_tier', filters.riskTier);
    if (filters.hasSDOHNeeds) query.whereRaw("array_length(patient.active_sdoh_needs, 1) > 0");
    
    return query.orderByRaw(`
      CASE patient.risk_tier
        WHEN 'very_high' THEN 1 WHEN 'high' THEN 2
        WHEN 'moderate' THEN 3 WHEN 'rising' THEN 4
        WHEN 'low' THEN 5
      END ASC,
      patient.composite_risk_score DESC
    `);
  }
}
```

```tsx
// apps/web/src/app/(dashboard)/worklist/page.tsx
export default async function WorklistPage() {
  return (
    <div className="space-y-4">
      <WorklistHeader />
      <WorklistFilters />
      <WorklistTable>
        {/* Columns: Patient Name | Risk Tier | Risk Score | Open Gaps | Open Tasks | Next Due | SDOH Needs | Actions */}
      </WorklistTable>
    </div>
  );
}
```

**Testing:**
- [ ] Worklist shows only patients with tasks assigned to the logged-in care manager
- [ ] Patients ordered by risk tier (very_high first) then composite_risk_score descending
- [ ] Open care gap count is accurate for each patient
- [ ] Open task count reflects only non-completed/cancelled tasks
- [ ] Next due date shows the earliest upcoming task due date
- [ ] Filter by risk tier: only patients with selected tier shown
- [ ] Filter by SDOH needs: only patients with active_sdoh_needs shown
- [ ] Empty worklist: care manager with no assigned patients sees an empty state message
- [ ] Performance: worklist for care manager with 200 patients loads within 500ms

---

### Task 4.4: Care Team Management

**What:** Build care team CRUD. A care team is a group of providers (and the patient) associated with a patient. Members are stored as a JSONB array with role, provider_id, and validity period. Care team changes are logged for audit. Care teams are synced to FHIR CareTeam resources.

**Design:**

```typescript
// apps/api/src/modules/care-team/care-team.service.ts
@Injectable()
export class CareTeamService {
  async addMember(careTeamId: string, dto: AddCareTeamMemberDto): Promise<CareTeamEntity> {
    const careTeam = await this.careTeamRepo.findById(careTeamId);
    
    const updatedMembers = [
      ...careTeam.members,
      {
        provider_id: dto.providerId,
        role: dto.role,
        display: dto.displayName,
        period_start: dto.startDate || new Date().toISOString().split('T')[0],
        period_end: null,
      },
    ];

    const updated = await this.careTeamRepo.update(careTeamId, { members: updatedMembers });
    await this.fhirSyncService.syncCareTeamToFhir(updated);
    
    return updated;
  }

  async removeMember(careTeamId: string, providerId: string): Promise<CareTeamEntity> {
    const careTeam = await this.careTeamRepo.findById(careTeamId);
    const updatedMembers = careTeam.members.map(m =>
      m.provider_id === providerId
        ? { ...m, period_end: new Date().toISOString().split('T')[0] }
        : m
    );

    return this.careTeamRepo.update(careTeamId, { members: updatedMembers });
  }
}
```

**Testing:**
- [ ] Create care team for a patient with PCP and care manager
- [ ] Add specialist to care team: member appears in JSONB array with correct role
- [ ] Remove member: period_end set to today (soft-delete pattern)
- [ ] Care team synced to FHIR CareTeam resource with correct participant roles
- [ ] Query: "find all care teams where Dr. Smith is a member" via GIN index on members JSONB
- [ ] Care team cannot have duplicate active members (same provider_id with no period_end)

---

### Phase 4 Definition of Done

- [ ] Care plans: full CRUD with lifecycle management, goals, and activities
- [ ] Tasks: creation, assignment, lifecycle, time tracking, priority ordering
- [ ] Worklist: care manager sees prioritised patient list with risk, gaps, tasks, SDOH flags
- [ ] Care teams: CRUD with member management, FHIR sync, audit trail
- [ ] All entities synced to FHIR server
- [ ] Frontend: worklist page, patient detail page, care plan editor functional
- [ ] API documentation: OpenAPI/Swagger generated for all endpoints

---

## Phase 5: Risk Stratification and Care Gap Engine

**Goal:** Build the ML-based risk stratification engine and the care gap identification system. These are the analytical engines that drive care manager prioritisation and quality measure performance.

**Duration estimate:** 5-7 weeks

### Task 5.1: Risk Stratification Model

**What:** Build a Python ML service that computes composite risk scores for each patient based on chronic condition burden (HCC count, CCI score), recent utilisation (ED visits, inpatient admissions in 12 months), SDOH factors, medication adherence signals, and care plan adherence. Deploy as a microservice that runs on a schedule (nightly batch) and on-demand (when new clinical data arrives). Risk scores are stored in the risk_score table with contributing factors in JSONB details.

**Design:**

```python
# services/risk-engine/src/models/composite_risk.py
import xgboost as xgb
import pandas as pd
from dataclasses import dataclass

@dataclass
class RiskFeatures:
    hcc_count: int
    cci_score: float           # Charlson Comorbidity Index
    ed_visits_12m: int
    inpatient_admits_12m: int
    inpatient_days_12m: int
    sdoh_need_count: int
    open_care_gaps: int
    days_since_last_pcp_visit: int
    medication_count: int
    age: int
    has_behavioral_health_dx: bool

class CompositeRiskModel:
    def __init__(self, model_path: str):
        self.model = xgb.Booster()
        self.model.load_model(model_path)
    
    def predict(self, features: RiskFeatures) -> dict:
        df = pd.DataFrame([features.__dict__])
        dmatrix = xgb.DMatrix(df)
        raw_score = self.model.predict(dmatrix)[0]
        
        return {
            'score_value': float(raw_score),
            'risk_tier': self._score_to_tier(raw_score),
            'contributing_factors': self._explain(df, raw_score),
        }
    
    def _score_to_tier(self, score: float) -> str:
        if score >= 0.85: return 'very_high'
        if score >= 0.70: return 'high'
        if score >= 0.50: return 'moderate'
        if score >= 0.30: return 'rising'
        return 'low'
    
    def _explain(self, df, score) -> list:
        # SHAP-based feature importance for this prediction
        import shap
        explainer = shap.TreeExplainer(self.model)
        shap_values = explainer.shap_values(df)
        
        features = df.columns.tolist()
        contributions = sorted(
            zip(features, shap_values[0]),
            key=lambda x: abs(x[1]), reverse=True
        )
        return [{'factor': f, 'weight': round(float(w), 4)} for f, w in contributions[:5]]
```

**Testing:**
- [ ] Model training: train on synthetic dataset of 10K patients; AUC-ROC >= 0.75
- [ ] Score range: all scores between 0.0 and 1.0
- [ ] Tier assignment: score 0.90 maps to 'very_high'; score 0.20 maps to 'low'
- [ ] Contributing factors: top-5 SHAP values returned for each prediction
- [ ] Batch scoring: 100K patients scored within 15 minutes
- [ ] On-demand scoring: single patient scored within 500ms
- [ ] Score stored in risk_score table with model_version, contributing factors in details JSONB
- [ ] Patient record's denormalised risk_tier and composite_risk_score fields updated after scoring

---

### Task 5.2: Readmission Risk Model

**What:** Build a 30-day readmission prediction model. Features include: discharge diagnosis, length of stay, number of prior admissions in 12 months, discharge disposition, SDOH flags, medication count, age, and whether a follow-up PCP visit was scheduled. Model output is stored as a separate risk_score record with score_type = 'readmission_30d'.

**Design:**

```python
# services/risk-engine/src/models/readmission_risk.py
class ReadmissionRiskModel:
    FEATURES = [
        'discharge_drg', 'length_of_stay_days', 'prior_admits_12m',
        'prior_ed_visits_12m', 'discharge_to_snf', 'sdoh_need_count',
        'medication_count', 'age', 'has_pcp_followup_scheduled',
        'has_behavioral_health_dx', 'hcc_raf_score',
    ]
    
    def predict_at_discharge(self, encounter_id: str) -> dict:
        features = self.feature_extractor.extract(encounter_id)
        score = self.model.predict_proba([features])[0][1]
        
        return {
            'score_type': 'readmission_30d',
            'score_value': float(score),
            'risk_tier': 'high' if score >= 0.30 else ('moderate' if score >= 0.15 else 'low'),
            'details': {
                'encounter_id': encounter_id,
                'model_version': self.model_version,
                'contributing_factors': self._explain(features),
            },
        }
```

**Testing:**
- [ ] Model predicts at discharge event (A03 ADT message triggers scoring)
- [ ] High-risk patients (score >= 0.30) flagged with 'high' readmission risk tier
- [ ] Score incorporates SDOH factors (food insecurity increases risk)
- [ ] Patients discharged to SNF scored differently from those discharged home
- [ ] Performance: scoring triggered within 60 seconds of discharge ADT message
- [ ] Readmission risk alerts created as tasks for transitional care team

---

### Task 5.3: Care Gap Identification Engine

**What:** Build the care gap engine that identifies open care gaps for each patient based on quality measure definitions. The engine runs nightly and also on-demand when new clinical data arrives. Care gaps are identified by evaluating each patient against the denominator criteria, then checking whether the numerator criteria are met. Open gaps are inserted into the care_gap table and synced to care manager worklists.

**Design:**

```typescript
// apps/api/src/modules/care-gap/care-gap-engine.service.ts
@Injectable()
export class CareGapEngineService {
  async evaluatePatient(tenantId: string, patientId: string): Promise<CareGapResult[]> {
    const measures = await this.measureRepo.findActive(tenantId);
    const patientData = await this.patientDataService.getFullProfile(tenantId, patientId);
    const results: CareGapResult[] = [];

    for (const measure of measures) {
      // Check if patient is in denominator
      if (!this.isInDenominator(patientData, measure.criteria.denominator)) continue;
      
      // Check if patient is excluded
      if (this.isExcluded(patientData, measure.criteria.exclusions)) {
        results.push({ measureId: measure.id, status: 'excluded' });
        continue;
      }

      // Check if numerator criteria met
      if (this.isNumeratorMet(patientData, measure.criteria.numerator)) {
        results.push({ measureId: measure.id, status: 'closed' });
      } else {
        results.push({ measureId: measure.id, status: 'open', dueDate: this.calculateDueDate(measure) });
      }
    }

    return results;
  }

  private isInDenominator(patient: PatientProfile, criteria: DenominatorCriteria): boolean {
    // Age check
    if (criteria.age_range) {
      const age = this.calculateAge(patient.dateOfBirth);
      if (age < criteria.age_range[0] || age > criteria.age_range[1]) return false;
    }
    // Condition check (ICD-10 pattern matching)
    if (criteria.conditions) {
      const hasDx = criteria.conditions.some(pattern =>
        patient.conditions.some(c => c.code_value.startsWith(pattern.replace('*', '')))
      );
      if (!hasDx) return false;
    }
    return true;
  }

  private isNumeratorMet(patient: PatientProfile, criteria: NumeratorCriteria): boolean {
    if (criteria.observation_code) {
      const obs = patient.observations
        .filter(o => o.code_value === criteria.observation_code)
        .sort((a, b) => new Date(b.effective_date).getTime() - new Date(a.effective_date).getTime());
      
      if (obs.length === 0) return false;
      const latestValue = obs[0].value_quantity;
      
      if (criteria.comparison === 'less_than') return latestValue < criteria.value_threshold;
      if (criteria.comparison === 'greater_than') return latestValue > criteria.value_threshold;
    }
    return false;
  }
}
```

**Testing:**
- [ ] Diabetes care gap (HbA1c): patient with E11.9 and no HbA1c in 12 months -> gap open
- [ ] Diabetes care gap: patient with E11.9 and HbA1c = 6.5% (< 9.0 threshold) -> gap closed
- [ ] Age exclusion: patient age 80 excluded from measure with age_range [18, 75]
- [ ] Hospice exclusion: patient with hospice encounter excluded from all gaps
- [ ] Nightly batch: 50K patients evaluated against 20 measures within 30 minutes
- [ ] On-demand: new lab result triggers re-evaluation of affected measures for that patient
- [ ] Care gap closure: lab result meeting numerator criteria automatically closes the gap
- [ ] Care gap inserted with correct measure_id, identified_date, and due_date

---

### Phase 5 Definition of Done

- [ ] Composite risk scores computed for all patients; risk_tier and composite_risk_score on patient record
- [ ] Readmission risk scored at every discharge event
- [ ] Care gaps identified for all patients against active quality measures
- [ ] Care gap closure triggered automatically by new clinical evidence (lab results, encounters)
- [ ] Risk scores and care gaps visible in care manager worklist
- [ ] Contributing factors (SHAP values) available for each risk score
- [ ] ML model versioning: new model versions deployed without disrupting existing scores

---

## Phase 6: Patient Outreach and Engagement

**Goal:** Build automated member outreach across SMS, email, and phone channels with response capture and care plan update. Implement the patient engagement portal for care plan visibility and secure messaging.

**Duration estimate:** 4-5 weeks

### Task 6.1: Outreach Campaign Engine

**What:** Build a rules-based outreach engine that generates and executes outreach campaigns for care gap closure, appointment reminders, medication adherence, and SDOH follow-up. Outreach rules are configured per tenant without custom code. Channels include SMS (Twilio), email (SendGrid), and phone (for manual call tasks). Outreach attempts are tracked in the outreach table.

**Design:**

```typescript
// services/outreach-engine/src/outreach.service.ts
@Injectable()
export class OutreachService {
  async executeOutreach(outreachRequest: OutreachRequest): Promise<OutreachResult> {
    const patient = await this.patientService.findById(outreachRequest.patientId);
    const template = await this.templateService.getTemplate(outreachRequest.templateId);
    
    const personalised = this.personalise(template, patient, outreachRequest.context);
    
    let result: OutreachResult;
    switch (outreachRequest.channel) {
      case 'sms':
        result = await this.twilioService.sendSms(patient.phones?.[0]?.number, personalised.body);
        break;
      case 'email':
        result = await this.sendgridService.sendEmail(patient.email, personalised.subject, personalised.body);
        break;
      case 'phone':
        // Create a manual call task for the care manager
        result = await this.taskService.createCallTask(outreachRequest);
        break;
    }

    await this.outreachRepo.create({
      tenantId: outreachRequest.tenantId,
      patientId: outreachRequest.patientId,
      taskId: outreachRequest.taskId,
      careGapId: outreachRequest.careGapId,
      channel: outreachRequest.channel,
      direction: 'outbound',
      status: result.success ? 'completed' : 'failed',
      attemptNumber: outreachRequest.attemptNumber,
      scheduledAt: outreachRequest.scheduledAt,
      attemptedAt: new Date(),
    });

    return result;
  }
}
```

**Testing:**
- [ ] SMS outreach: Twilio sandbox receives SMS with personalised care gap message
- [ ] Email outreach: SendGrid sends email with care gap details and call-to-action
- [ ] Phone outreach: task created for care manager with call script
- [ ] Outreach recorded in outreach table with correct channel, status, attempt number
- [ ] Retry logic: failed SMS retried via email after 48 hours (configurable per tenant)
- [ ] Opt-out handling: patient who opted out of SMS does not receive SMS outreach
- [ ] Rate limiting: no more than 3 outreach attempts per patient per care gap per week
- [ ] Template personalisation: patient name, care gap description, provider name inserted correctly

---

### Task 6.2: Patient Engagement Portal

**What:** Build a patient-facing web portal where patients can view their care plan, see upcoming appointments and tasks, complete SDOH screening questionnaires, message their care team, and access self-management resources. Implemented as a separate Next.js application (apps/portal) with patient-level OAuth authentication.

**Design:**

```tsx
// apps/portal/src/app/(patient)/care-plan/page.tsx
export default async function PatientCarePlanPage() {
  const carePlan = await getActiveCarePlan();
  
  return (
    <div className="max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-semibold">{carePlan.title}</h1>
      <p className="text-muted-foreground">{carePlan.description}</p>
      
      <section className="mt-6">
        <h2 className="text-xl font-medium">Your Goals</h2>
        {carePlan.goals.map(goal => (
          <GoalCard key={goal.id} goal={goal} />
        ))}
      </section>
      
      <section className="mt-6">
        <h2 className="text-xl font-medium">Upcoming Tasks</h2>
        {carePlan.activities
          .filter(a => a.status !== 'completed')
          .map(activity => (
            <ActivityCard key={activity.type + activity.scheduled} activity={activity} />
          ))}
      </section>
      
      <section className="mt-6">
        <h2 className="text-xl font-medium">Messages</h2>
        <SecureMessaging careTeamId={carePlan.careTeamId} />
      </section>
    </div>
  );
}
```

**Testing:**
- [ ] Patient logs in with their credentials; sees only their own care plan
- [ ] Care plan goals displayed with achievement status (in-progress, improving, achieved)
- [ ] Upcoming tasks shown with due dates
- [ ] SDOH screening questionnaire: patient completes AHC-HRSN; responses saved to sdoh_screening table
- [ ] Secure messaging: patient sends message to care manager; care manager receives notification
- [ ] Accessibility: portal passes WCAG 2.1 AA audit (colour contrast, keyboard navigation, screen reader)
- [ ] Mobile responsive: portal usable on mobile devices (375px width and up)

---

### Phase 6 Definition of Done

- [ ] Outreach engine sends SMS, email, and creates phone call tasks
- [ ] Outreach rules configurable per tenant without code changes
- [ ] All outreach attempts tracked with channel, status, and attempt number
- [ ] Patient portal: care plan view, goal tracking, SDOH screening, secure messaging
- [ ] Patient authentication separate from care manager authentication
- [ ] WCAG 2.1 AA compliance on patient portal
- [ ] Outreach opt-out/consent management functional

---

## Phase 7: Referral Management and Transitions of Care

**Goal:** Build referral creation, tracking, and transitions-of-care workflows. Implement AI-generated clinical summaries for referral packets. Build the provider network directory with quality metrics.

**Duration estimate:** 4-6 weeks

### Task 7.1: Referral Lifecycle Management

**What:** Implement referral creation (specialty, post-acute, behavioural health, SDOH/CBO), tracking through the lifecycle (draft -> active -> on-hold -> completed/revoked), and provider matching. Referrals link to care plans and patients. The receiving provider/organisation accepts or declines the referral. Include abandonment risk scoring (ML-predicted risk that the referral will not be completed).

**Design:**

```typescript
// apps/api/src/modules/referral/referral.service.ts
@Injectable()
export class ReferralService {
  async createReferral(tenantId: string, dto: CreateReferralDto): Promise<ReferralEntity> {
    const abandmentRisk = await this.riskService.predictAbandonmentRisk({
      referralType: dto.referralType,
      patientRiskTier: dto.patientRiskTier,
      distanceToProvider: dto.distanceMiles,
      historicalCompletionRate: dto.receivingOrgCompletionRate,
    });

    return this.referralRepo.create({
      tenantId,
      patientId: dto.patientId,
      carePlanId: dto.carePlanId,
      referralType: dto.referralType,
      status: 'active',
      priority: dto.priority,
      reasonCode: dto.reasonCode,
      reasonDisplay: dto.reasonDisplay,
      referringProviderId: dto.referringProviderId,
      receivingOrgId: dto.receivingOrgId,
      receivingProviderId: dto.receivingProviderId,
      requestedDate: new Date(),
      abandonmentRiskScore: abandmentRisk.score,
      sdohCategory: dto.sdohCategory,
    });
  }

  async trackScheduling(referralId: string, scheduledDate: Date): Promise<ReferralEntity> {
    return this.referralRepo.update(referralId, {
      scheduledDate,
      status: 'active',
      abandonmentRiskScore: 0.05, // Scheduled referrals have low abandonment risk
    });
  }
}
```

**Testing:**
- [ ] Create specialty referral with referring and receiving providers
- [ ] Referral lifecycle: active -> scheduled -> completed
- [ ] Abandonment risk score computed at creation; high-risk referrals flagged
- [ ] Referral linked to care plan: appears in care plan activities
- [ ] Referral decline: receiving provider declines; status updated; care manager notified
- [ ] Overdue referral: not scheduled within 14 days triggers escalation task
- [ ] FHIR sync: referral creates ServiceRequest on HAPI FHIR server

---

### Task 7.2: AI-Generated Clinical Summaries for Referral Packets

**What:** Build an AI service that generates structured clinical summaries for referral packets and transitions-of-care documents. The summary synthesises the patient's active conditions, recent encounters, medications, labs, and care plan into a concise narrative suitable for the receiving provider. Output is a C-CDA Referral Note or a FHIR DocumentReference.

**Design:**

```python
# services/ai-copilot/src/clinical_summary.py
class ClinicalSummaryGenerator:
    def generate_referral_summary(self, patient_data: dict, referral_context: dict) -> str:
        prompt = f"""Generate a structured clinical referral summary for the following patient.
        
Patient: {patient_data['name']}, {patient_data['age']}yo {patient_data['gender']}
Reason for Referral: {referral_context['reason']}
Referring Provider: {referral_context['referring_provider']}

Active Conditions:
{self._format_conditions(patient_data['conditions'])}

Current Medications:
{self._format_medications(patient_data['medications'])}

Recent Lab Results:
{self._format_labs(patient_data['recent_labs'])}

Recent Encounters:
{self._format_encounters(patient_data['recent_encounters'])}

Active Care Plan Goals:
{self._format_goals(patient_data['care_plan_goals'])}

Generate a concise, structured clinical summary suitable for the receiving specialist.
Include: chief concern, relevant history, current medications, pertinent labs, 
and specific questions for the specialist."""
        
        response = self.llm_client.complete(prompt, max_tokens=2000)
        return response.text
```

**Testing:**
- [ ] Summary includes all active conditions with ICD-10 codes
- [ ] Summary includes current medications with dosages
- [ ] Summary includes pertinent recent lab results (HbA1c, lipids, renal function)
- [ ] Summary includes reason for referral and specific clinical questions
- [ ] PHI handling: summary generated via API with BAA-covered LLM provider
- [ ] Summary length: between 500 and 2000 words (configurable)
- [ ] C-CDA output: generated summary wrapped in valid C-CDA Referral Note XML
- [ ] FHIR output: summary stored as DocumentReference on HAPI FHIR server

---

### Task 7.3: Provider Network Directory

**What:** Build a searchable provider network directory with quality metrics (referral completion rate, readmission rate, average wait time) and availability information. Used by care managers when creating referrals to find the best matching in-network provider. Supports filtering by specialty, location, quality score, and network membership.

**Design:**

```typescript
// apps/api/src/modules/provider/provider-directory.service.ts
@Injectable()
export class ProviderDirectoryService {
  async searchProviders(tenantId: string, criteria: ProviderSearchCriteria): Promise<ProviderDirectoryResult[]> {
    const query = this.db('provider')
      .join('organisation', 'provider.organisation_id', 'organisation.id')
      .select(
        'provider.*',
        'organisation.name as org_name',
        'organisation.city', 'organisation.state_code',
      )
      .where('provider.tenant_id', tenantId)
      .where('provider.is_active', true);

    if (criteria.specialty) query.where('provider.specialty_code', criteria.specialty);
    if (criteria.city) query.where('organisation.city', criteria.city);
    if (criteria.state) query.where('organisation.state_code', criteria.state);

    const providers = await query;

    // Enrich with quality metrics
    return Promise.all(providers.map(async p => ({
      ...p,
      metrics: await this.getProviderMetrics(tenantId, p.id),
    })));
  }

  private async getProviderMetrics(tenantId: string, providerId: string) {
    const referrals = await this.db('referral')
      .where({ tenant_id: tenantId, receiving_provider_id: providerId })
      .select(
        this.db.raw("COUNT(*) as total_referrals"),
        this.db.raw("COUNT(*) FILTER (WHERE status = 'completed') as completed_referrals"),
        this.db.raw("AVG(EXTRACT(DAY FROM scheduled_date - requested_date)) as avg_days_to_schedule"),
      )
      .first();

    return {
      referralCompletionRate: referrals.total_referrals > 0
        ? referrals.completed_referrals / referrals.total_referrals
        : null,
      avgDaysToSchedule: referrals.avg_days_to_schedule,
      totalReferrals: referrals.total_referrals,
    };
  }
}
```

**Testing:**
- [ ] Search by specialty returns only providers with matching specialty_code
- [ ] Search by city/state returns providers at organisations in that location
- [ ] Quality metrics: referral completion rate calculated correctly from referral history
- [ ] Average days to schedule calculated correctly
- [ ] Providers with no referral history show null for metrics (not zero)
- [ ] Results ordered by referral completion rate descending by default
- [ ] Performance: search across 5K providers returns within 500ms

---

### Phase 7 Definition of Done

- [ ] Referral CRUD with lifecycle management and abandonment risk scoring
- [ ] AI-generated clinical summaries for referral packets (C-CDA and FHIR format)
- [ ] Provider network directory with searchable quality metrics
- [ ] High-risk referrals (abandonment score > 0.5) generate automated follow-up tasks
- [ ] Referral dashboard showing pipeline by status, type, and risk level
- [ ] Transitions of care documentation supports CMS TCM billing requirements

---

## Phase 8: SDOH Screening and Closed-Loop Referral

**Goal:** Implement SDOH screening (PRAPARE, AHC-HRSN), connection to community-based organisations (CBOs), and closed-loop tracking that confirms whether social needs were actually addressed. This addresses the industry gap where platforms screen for SDOH but do not close the loop.

**Duration estimate:** 3-4 weeks

### Task 8.1: SDOH Screening Workflow

**What:** Implement SDOH screening using standardised instruments (PRAPARE, AHC-HRSN). Screening can be initiated by care managers during encounters or by patients via the portal. Responses are stored in the sdoh_screening table with responses as JSONB array. Risk flags are computed per SDOH domain (food insecurity, housing instability, transportation, utilities, interpersonal safety). Identified needs update the patient's `active_sdoh_needs` array.

**Design:**

```typescript
// apps/api/src/modules/sdoh/sdoh-screening.service.ts
@Injectable()
export class SdohScreeningService {
  async submitScreening(tenantId: string, dto: SubmitScreeningDto): Promise<SdohScreeningEntity> {
    const identifiedNeeds = dto.responses
      .filter(r => r.riskFlag)
      .map(r => r.sdohCategory)
      .filter((v, i, a) => a.indexOf(v) === i); // unique categories

    const screening = await this.screeningRepo.create({
      tenantId,
      patientId: dto.patientId,
      encounterId: dto.encounterId,
      instrument: dto.instrument,
      screeningDate: new Date(),
      screenerId: dto.screenerId,
      status: 'completed',
      responses: dto.responses,
      identifiedNeeds,
    });

    // Update patient's active SDOH needs
    await this.patientRepo.updateSdohNeeds(dto.patientId, identifiedNeeds);
    
    // Create SDOH conditions for identified needs (Gravity IG)
    for (const need of identifiedNeeds) {
      await this.conditionService.createSdohCondition(tenantId, dto.patientId, need);
    }

    // Auto-generate SDOH referral tasks for identified needs
    for (const need of identifiedNeeds) {
      await this.taskService.createTask(tenantId, {
        patientId: dto.patientId,
        taskType: 'sdoh_referral',
        description: `SDOH referral needed: ${need}`,
        priority: 'urgent',
      });
    }

    // Sync to FHIR (Gravity SDOH IG Observation resources)
    await this.fhirSyncService.syncSdohScreening(screening);

    return screening;
  }
}
```

**Testing:**
- [ ] AHC-HRSN screening with food insecurity risk flag: identifiedNeeds includes 'food_insecurity'
- [ ] Patient's active_sdoh_needs array updated with new needs
- [ ] SDOH condition created with is_sdoh=true and correct sdoh_category
- [ ] Referral task auto-generated for each identified need
- [ ] FHIR sync: Observation resources created per Gravity SDOH IG profiles
- [ ] Duplicate screening: second screening in same day updates, not duplicates
- [ ] Portal submission: patient self-completes screening; same processing pipeline

---

### Task 8.2: CBO Directory and Closed-Loop Referral

**What:** Build a community-based organisation (CBO) directory and implement closed-loop SDOH referral tracking. When a social need is identified, the care manager selects a CBO from the directory and creates a referral. The CBO confirms or denies service delivery via a CBO portal or API. The confirmation closes the loop, updating the referral record and the patient's SDOH status.

**Design:**

```typescript
// apps/api/src/modules/sdoh/cbo-referral.service.ts
@Injectable()
export class CboReferralService {
  async createSdohReferral(tenantId: string, dto: CreateSdohReferralDto): Promise<ReferralEntity> {
    const referral = await this.referralService.createReferral(tenantId, {
      patientId: dto.patientId,
      referralType: 'sdoh_cbo',
      reasonDisplay: `SDOH: ${dto.sdohCategory}`,
      referringProviderId: dto.careManagerId,
      receivingOrgId: dto.cboOrganisationId,
      sdohCategory: dto.sdohCategory,
    });

    // Sync to FHIR (Gravity IG ServiceRequest + Task for CBO)
    await this.fhirSyncService.syncSdohReferral(referral);
    
    return referral;
  }

  async confirmCboService(referralId: string, dto: CboConfirmationDto): Promise<ReferralEntity> {
    const referral = await this.referralRepo.update(referralId, {
      status: 'completed',
      completedDate: new Date(),
      cboTracking: {
        cbo_name: dto.cboName,
        cbo_contact: dto.contactPerson,
        service_type: dto.serviceType,
        confirmation_date: new Date().toISOString(),
        confirmation_method: dto.method, // 'portal', 'phone', 'fax'
        services_provided: dto.servicesProvided,
        follow_up_date: dto.followUpDate,
      },
    });

    // Resolve the SDOH condition if need is met
    if (dto.needResolved) {
      await this.conditionService.resolveCondition(referral.patientId, referral.sdohCategory);
      await this.patientRepo.removeSdohNeed(referral.patientId, referral.sdohCategory);
    }

    return referral;
  }
}
```

**Testing:**
- [ ] CBO directory search by SDOH category (food_insecurity) returns relevant CBOs
- [ ] SDOH referral created with sdoh_category and linked to CBO organisation
- [ ] CBO confirmation: cboTracking JSONB populated with service details
- [ ] Closed loop: CBO confirms service -> SDOH condition resolved -> patient active_sdoh_needs updated
- [ ] Unconfirmed referral: no CBO response after 14 days -> escalation task created
- [ ] FHIR sync: ServiceRequest and Task resources created per Gravity SDOH IG
- [ ] Reporting: count of open vs closed SDOH referrals by category

---

### Phase 8 Definition of Done

- [ ] SDOH screening (PRAPARE/AHC-HRSN) functional from care manager UI and patient portal
- [ ] Identified social needs create conditions, tasks, and update patient record
- [ ] CBO directory searchable by SDOH category and location
- [ ] Closed-loop referral: CBO confirmation closes the SDOH referral and resolves the condition
- [ ] Gravity SDOH IG compliance: all SDOH data exchanged as conformant FHIR resources
- [ ] End-to-end flow: screening -> need identified -> CBO referral -> CBO confirms -> need resolved

---

## Phase 9: Quality Measurement and Reporting

**Goal:** Build HEDIS measure calculation and CMS quality programme reporting. Implement quality measure dashboards with drill-down to individual patient care gaps.

**Duration estimate:** 4-5 weeks

### Task 9.1: HEDIS Measure Calculation Engine

**What:** Build a configurable quality measure calculation engine that evaluates patient populations against HEDIS measure specifications. Measure definitions are stored in the quality_measure table with criteria in JSONB. The engine computes denominators, numerators, exclusions, and rates for each measure at the organisation, provider, and programme level. Results are stored in a measure_result table (to be added as a migration).

**Design:**

```typescript
// apps/api/src/modules/quality-measure/measure-engine.service.ts
@Injectable()
export class MeasureEngineService {
  async calculateMeasure(
    tenantId: string,
    measureId: string,
    reportingPeriod: { start: Date; end: Date },
    orgId?: string,
  ): Promise<MeasureResult> {
    const measure = await this.measureRepo.findById(measureId);
    const patients = await this.patientRepo.findByTenant(tenantId, orgId);
    
    let denominator = 0, numerator = 0, exclusions = 0;

    for (const patient of patients) {
      const profile = await this.patientDataService.getProfile(tenantId, patient.id, reportingPeriod);
      
      if (!this.isInDenominator(profile, measure.criteria.denominator)) continue;
      
      if (this.isExcluded(profile, measure.criteria.exclusions)) {
        exclusions++;
        continue;
      }
      
      denominator++;
      if (this.isNumeratorMet(profile, measure.criteria.numerator)) {
        numerator++;
      }
    }

    const rate = denominator > 0 ? (numerator / denominator) * 100 : 0;

    return this.measureResultRepo.create({
      tenantId,
      measureId,
      reportingPeriodStart: reportingPeriod.start,
      reportingPeriodEnd: reportingPeriod.end,
      organisationId: orgId,
      denominatorCount: denominator,
      numeratorCount: numerator,
      exclusionCount: exclusions,
      rate,
    });
  }
}
```

**Testing:**
- [ ] Comprehensive Diabetes Care (HbA1c < 9%): correct denominator (diabetics 18-75), numerator (HbA1c < 9)
- [ ] Transitions of Care (TRC): patients with inpatient discharge have follow-up visit within 30 days
- [ ] Exclusions: hospice patients excluded from all applicable measures
- [ ] Rate calculation: 80 numerator / 100 denominator = 80.00% rate
- [ ] Measure results stored with reporting period, org, and measure ID
- [ ] Batch calculation: 20 measures across 50K patients within 60 minutes
- [ ] Measure results match manually calculated expected values for a test population

---

### Task 9.2: Quality Measure Dashboard

**What:** Build a quality measure dashboard showing performance across all active measures with rates, benchmarks (50th and 90th percentile), and trend over time. Drill-down to individual measure shows denominator/numerator patients with open care gaps. Provider-level scorecards show performance by attributed provider.

**Design:**

```tsx
// apps/web/src/app/(dashboard)/quality/page.tsx
export default async function QualityDashboard() {
  const measures = await getMeasureResults();
  
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-semibold">Quality Measure Performance</h1>
      
      <MeasureSummaryCards measures={measures} />
      
      <MeasurePerformanceTable>
        {/* Columns: Measure | Domain | Rate | 50th %ile | 90th %ile | Trend | Open Gaps | Actions */}
      </MeasurePerformanceTable>
      
      <ProviderScorecard />
    </div>
  );
}
```

**Testing:**
- [ ] Dashboard shows all active measures with current rate, benchmark comparisons
- [ ] Trend chart: rate over last 6 reporting periods displayed correctly
- [ ] Drill-down: clicking a measure shows patients in denominator with gap status
- [ ] Provider scorecard: per-provider rates displayed for attributed panels
- [ ] Export: measure results exportable as CSV for CMS reporting
- [ ] Filter by measure set (HEDIS, CMS Stars, MIPS)

---

### Phase 9 Definition of Done

- [ ] HEDIS measure calculation engine evaluating populations against measure criteria
- [ ] Measure results stored with rates, denominators, numerators, exclusions
- [ ] Quality dashboard with measure performance, benchmarks, and trends
- [ ] Provider scorecards showing per-provider quality performance
- [ ] Drill-down from measure to individual patient care gaps
- [ ] CSV export for CMS quality programme submissions
- [ ] NCQA licence compliance documented (required for commercial HEDIS calculation)

---

## Phase 10: AI Copilot and Autonomous Agents

**Goal:** Build AI-native features that differentiate this platform: the AI Copilot for care managers, autonomous care gap closure agents, and NLP extraction from clinical notes.

**Duration estimate:** 6-8 weeks

### Task 10.1: AI Copilot for Care Managers

**What:** Build an AI assistant embedded in the care manager workflow that auto-populates care plan fields, suggests goals based on patient conditions, drafts documentation, and recommends next actions. The Copilot uses LLM APIs (Claude/GPT-4) with patient context to generate suggestions. All Copilot outputs are presented as suggestions requiring care manager approval.

**Design:**

```typescript
// services/ai-copilot/src/copilot.service.ts
@Injectable()
export class CopilotService {
  async suggestCarePlan(patientId: string): Promise<CarePlanSuggestion> {
    const context = await this.patientDataService.getFullProfile(patientId);
    
    const prompt = this.buildCarePlanPrompt(context);
    const response = await this.llmClient.complete(prompt, {
      maxTokens: 3000,
      temperature: 0.3,
    });
    
    return this.parseCarePlanSuggestion(response.text);
  }

  async suggestNextActions(patientId: string, carePlanId: string): Promise<ActionSuggestion[]> {
    const context = await this.buildActionContext(patientId, carePlanId);
    
    const prompt = `Given this patient's care plan status and open items, suggest the top 3 next actions:
    
Patient: ${context.patientSummary}
Active Conditions: ${context.conditions}
Open Care Gaps: ${context.openGaps}
Open Tasks: ${context.openTasks}
Last Outreach: ${context.lastOutreach}
SDOH Needs: ${context.sdohNeeds}

For each action, provide: action description, priority (stat/urgent/routine), and rationale.`;

    const response = await this.llmClient.complete(prompt, { maxTokens: 1000 });
    return this.parseActionSuggestions(response.text);
  }

  async draftDocumentation(taskId: string): Promise<string> {
    const task = await this.taskService.findById(taskId);
    const context = await this.buildDocContext(task);
    
    const prompt = `Draft a clinical note documenting the following care coordination activity:
    
Activity Type: ${task.taskType}
Patient: ${context.patientName}
Duration: ${task.timeSpentMin} minutes
Notes: ${task.notes || 'No notes provided'}
Care Plan: ${context.carePlanTitle}

Draft a concise clinical note suitable for the medical record. Include time spent (for CCM/TCM documentation). Use professional clinical language.`;

    const response = await this.llmClient.complete(prompt, { maxTokens: 500 });
    return response.text;
  }
}
```

**Testing:**
- [ ] Care plan suggestion: Copilot suggests appropriate goals for a diabetic patient (HbA1c target, foot exam, eye exam)
- [ ] Next action suggestion: identifies the highest-priority open care gap and suggests outreach
- [ ] Documentation draft: generates a clinical note with time spent for CCM billing
- [ ] All Copilot outputs are suggestions; no auto-execution without care manager approval
- [ ] Copilot response time: under 5 seconds for each suggestion type
- [ ] PHI handling: Copilot uses BAA-covered LLM provider; no PHI in logs
- [ ] Hallucination guard: Copilot cannot invent conditions or medications not in the patient's record

---

### Task 10.2: Autonomous Care Gap Closure Agents

**What:** Build AI agents that autonomously close low-complexity care gaps without care manager intervention. The agent identifies the gap, generates personalised outreach (SMS/email), captures the patient's response, and updates the care plan. Agents only operate on gaps classified as "low-complexity" (e.g., preventive care reminders, medication refill reminders). All agent actions are logged for audit.

**Design:**

```typescript
// services/ai-copilot/src/agents/care-gap-agent.ts
export class CareGapClosureAgent {
  async processGap(careGap: CareGapEntity): Promise<AgentResult> {
    // Only process low-complexity gaps
    if (!this.isLowComplexity(careGap)) {
      return { action: 'skipped', reason: 'High-complexity gap requires care manager' };
    }

    // Generate personalised outreach
    const patient = await this.patientService.findById(careGap.patientId);
    const message = await this.generateOutreachMessage(patient, careGap);
    
    // Execute outreach
    const outreachResult = await this.outreachService.executeOutreach({
      patientId: patient.id,
      careGapId: careGap.id,
      channel: this.selectChannel(patient),
      message,
      isAutonomous: true,
    });

    // Log agent action for audit
    await this.auditService.logAgentAction({
      agentId: 'care-gap-closure-agent-v1',
      action: 'outreach_sent',
      careGapId: careGap.id,
      patientId: patient.id,
      details: { channel: outreachResult.channel, messagePreview: message.substring(0, 100) },
    });

    // Update care gap status
    await this.careGapRepo.update(careGap.id, {
      gapStatus: 'pending_closure',
      closureMethod: 'ai_agent',
      evidence: {
        ...careGap.evidence,
        outreach_history: [
          ...(careGap.evidence?.outreach_history || []),
          { date: new Date().toISOString(), channel: outreachResult.channel, status: 'sent', agent: true },
        ],
      },
    });

    return { action: 'outreach_sent', channel: outreachResult.channel };
  }

  private isLowComplexity(gap: CareGapEntity): boolean {
    const lowComplexityMeasures = [
      'preventive_care_screening',
      'annual_wellness_visit',
      'flu_vaccination',
      'mammography_screening',
      'colorectal_cancer_screening',
    ];
    return lowComplexityMeasures.includes(gap.measureSet);
  }
}
```

**Testing:**
- [ ] Agent processes low-complexity gap (flu vaccination): outreach sent via SMS
- [ ] Agent skips high-complexity gap (diabetes management): returns 'skipped'
- [ ] Audit log records agent action with agent ID, care gap ID, and action details
- [ ] Care gap status updated to 'pending_closure' with closure_method = 'ai_agent'
- [ ] Patient opt-out: agent does not contact patients who opted out
- [ ] Rate limit: agent processes no more than 500 gaps per hour per tenant
- [ ] Agent does not re-process gaps that already have pending outreach within 7 days

---

### Task 10.3: Clinical NLP Extraction

**What:** Build an NLP service that extracts structured diagnoses, procedures, and HCC risk-adjustment codes from unstructured clinical notes. Uses a fine-tuned NLP model or LLM prompting to identify ICD-10 codes, SNOMED CT concepts, and medication mentions in free-text notes.

**Design:**

```python
# services/nlp-service/src/clinical_nlp.py
class ClinicalNLPExtractor:
    def extract_from_note(self, clinical_note: str) -> ExtractionResult:
        prompt = f"""Extract structured clinical information from the following note.
        
For each finding, provide:
- ICD-10-CM code (if applicable)
- SNOMED CT code (if applicable)  
- Confidence level (high/medium/low)
- Relevant text span

Clinical Note:
{clinical_note}

Return as JSON array of findings."""

        response = self.llm_client.complete(prompt, response_format="json")
        findings = json.loads(response.text)
        
        return ExtractionResult(
            conditions=[f for f in findings if f['type'] == 'condition'],
            medications=[f for f in findings if f['type'] == 'medication'],
            procedures=[f for f in findings if f['type'] == 'procedure'],
            hcc_codes=self._map_to_hcc(findings),
        )

    def _map_to_hcc(self, findings: list) -> list:
        """Map ICD-10 codes to HCC categories for risk adjustment."""
        hcc_mappings = self.hcc_crosswalk.lookup_batch(
            [f['icd10_code'] for f in findings if f.get('icd10_code')]
        )
        return [m for m in hcc_mappings if m is not None]
```

**Testing:**
- [ ] Note mentioning "Type 2 diabetes with nephropathy" extracts E11.65 and maps to HCC 18
- [ ] Note mentioning "metformin 1000mg BID" extracts medication with dose
- [ ] Confidence levels: explicit diagnoses get 'high'; mentioned-in-passing get 'medium'
- [ ] No hallucinated codes: extracted ICD-10 codes validate against ICD-10-CM codeset
- [ ] Batch processing: 100 clinical notes processed within 5 minutes
- [ ] Extracted conditions presented to provider for confirmation before adding to record

---

### Phase 10 Definition of Done

- [ ] AI Copilot: care plan suggestions, next action recommendations, documentation drafting
- [ ] All Copilot outputs require human approval before execution
- [ ] Autonomous care gap closure agents operational for low-complexity gaps
- [ ] Agent actions fully audited with agent ID and correlation to patient/gap
- [ ] Clinical NLP extracting ICD-10, SNOMED, and HCC codes from unstructured notes
- [ ] PHI handled securely: BAA-covered LLM providers, no PHI in logs
- [ ] Agent processing rate limits and opt-out controls functional

---

## Phase 11: Billing, CCM/TCM, and Value-Based Care Programmes

**Goal:** Build CCM (99490-99491) and TCM (99495-99496) billing documentation with time tracking. Implement value-based care programme management with patient attribution.

**Duration estimate:** 3-4 weeks

### Task 11.1: CCM/TCM Time Tracking and Billing

**What:** Build dedicated CCM and TCM billing documentation. CCM requires tracking cumulative non-face-to-face time per patient per month (>= 20 min for 99490, >= 40 min for 99491). TCM requires documenting discharge contact within 2 business days and a face-to-face visit within 7 or 14 days of discharge. Time is aggregated from task time entries. Billing eligibility is calculated and billing records are generated.

**Design:**

```typescript
// apps/api/src/modules/billing/ccm-billing.service.ts
@Injectable()
export class CcmBillingService {
  async calculateMonthlyBilling(
    tenantId: string,
    patientId: string,
    serviceMonth: Date,
  ): Promise<CcmBillingEligibility> {
    const monthStart = startOfMonth(serviceMonth);
    const monthEnd = endOfMonth(serviceMonth);

    // Aggregate time from all tasks for this patient this month
    const tasks = await this.taskRepo.findByPatientAndDateRange(
      tenantId, patientId, monthStart, monthEnd
    );

    const totalMinutes = tasks.reduce((sum, t) => sum + (t.timeSpentMin || 0), 0);

    let cptCode: string | null = null;
    if (totalMinutes >= 40) {
      cptCode = '99491'; // Complex CCM
    } else if (totalMinutes >= 20) {
      cptCode = '99490'; // Standard CCM
    }

    // Verify patient has 2+ chronic conditions (CCM requirement)
    const chronicConditions = await this.conditionService.countChronic(tenantId, patientId);
    if (chronicConditions < 2) cptCode = null;

    return {
      patientId,
      serviceMonth: monthStart,
      totalMinutes,
      cptCode,
      eligible: cptCode !== null,
      chronicConditionCount: chronicConditions,
      tasks: tasks.map(t => ({
        taskId: t.id,
        type: t.taskType,
        minutes: t.timeSpentMin,
        date: t.completedDate,
      })),
    };
  }
}
```

**Testing:**
- [ ] Patient with 25 minutes logged in May: eligible for 99490 (standard CCM)
- [ ] Patient with 45 minutes logged in May: eligible for 99491 (complex CCM)
- [ ] Patient with 15 minutes logged: not eligible (below 20-minute threshold)
- [ ] Patient with 30 minutes but only 1 chronic condition: not eligible
- [ ] TCM: discharge contact within 2 business days documented; follow-up visit within 14 days
- [ ] TCM 99495 (high complexity): face-to-face within 7 days of discharge
- [ ] Billing log created with cptCode, totalMinutes, and task references

---

### Task 11.2: Value-Based Care Programme Management

**What:** Build VBC programme management with patient attribution. Programmes (MSSP ACO, REACH ACO, Medicare Advantage, Medicaid managed care) have defined populations, payers, and performance targets. Patients are attributed to programmes via claims-based or prospective attribution.

**Design:**

```typescript
// apps/api/src/modules/billing/vbc-programme.service.ts
@Injectable()
export class VbcProgrammeService {
  async createProgramme(tenantId: string, dto: CreateProgrammeDto): Promise<VbcProgrammeEntity> {
    return this.programmeRepo.create({
      tenantId,
      name: dto.name,
      programmeType: dto.programmeType,
      payerId: dto.payerId,
      startDate: dto.startDate,
      endDate: dto.endDate,
    });
  }

  async attributePatient(dto: AttributePatientDto): Promise<PatientAttributionEntity> {
    return this.attributionRepo.create({
      patientId: dto.patientId,
      tenantId: dto.tenantId,
      programmeId: dto.programmeId,
      payerId: dto.payerId,
      attributedPcpId: dto.pcpId,
      attributionStart: dto.startDate,
      attributionMethod: dto.method,
    });
  }

  async getProgrammeDashboard(tenantId: string, programmeId: string) {
    const attribution = await this.attributionRepo.countByProgramme(programmeId);
    const qualityMetrics = await this.measureService.getProgrammeMetrics(tenantId, programmeId);
    const financials = await this.claimService.getProgrammeFinancials(tenantId, programmeId);

    return { attribution, qualityMetrics, financials };
  }
}
```

**Testing:**
- [ ] Create MSSP ACO programme with payer, start/end dates
- [ ] Attribute 500 patients to the programme with claims-based method
- [ ] Programme dashboard: attributed patient count, quality metrics, financial summary
- [ ] Patient profile shows programme attribution(s)
- [ ] De-attribution: patient attribution end date set; patient no longer counted in programme

---

### Phase 11 Definition of Done

- [ ] CCM billing: monthly time aggregation, eligibility calculation, billing log generation
- [ ] TCM billing: discharge contact and follow-up visit documentation
- [ ] VBC programme CRUD with patient attribution
- [ ] Programme dashboards with attribution counts, quality metrics, and financials
- [ ] Billing codes correctly assigned based on time thresholds and eligibility criteria

---

## Phase 12: Analytics Dashboards and Population Health Views

**Goal:** Build executive and operational analytics dashboards providing population health views, programme performance, and actionable insights. This is the reporting layer that ties together all previous phases.

**Duration estimate:** 4-5 weeks

### Task 12.1: Population Health Dashboard

**What:** Build a population health dashboard showing risk tier distribution, care gap rates, SDOH prevalence, utilisation patterns, and outreach effectiveness across the attributed population. Support drill-down by organisation, provider panel, programme, and payer.

**Design:**

```tsx
// apps/web/src/app/(dashboard)/analytics/population/page.tsx
export default async function PopulationHealthDashboard() {
  return (
    <div className="grid grid-cols-2 gap-6">
      <RiskTierDistribution />   {/* Pie/donut chart: % in each risk tier */}
      <CareGapRates />           {/* Bar chart: open gap rate by measure */}
      <SdohPrevalence />         {/* Horizontal bar: % with each SDOH need */}
      <UtilisationTrends />      {/* Line chart: ED visits, admits over 12 months */}
      <OutreachEffectiveness />  {/* Funnel: sent -> reached -> completed -> gap closed */}
      <ReferralLeakage />       {/* Gauge: referral completion rate, abandonment rate */}
      <ProviderPerformance />   {/* Table: providers ranked by care gap closure rate */}
      <ProgrammeSummary />      {/* Cards: per-programme attributed lives, quality score */}
    </div>
  );
}
```

**Testing:**
- [ ] Risk tier distribution: chart shows correct percentages matching database counts
- [ ] Care gap rates: open gap rate = open gaps / denominator for each measure
- [ ] SDOH prevalence: percentage of population with each SDOH category
- [ ] Drill-down by organisation: filter changes all charts to selected org's population
- [ ] Drill-down by provider: charts show only patients attributed to selected provider
- [ ] Date range filter: analytics computed for selected time period
- [ ] Performance: dashboard loads within 3 seconds for 100K patient population
- [ ] Export: all charts exportable as PNG; data tables exportable as CSV

---

### Task 12.2: Executive Summary and Outcomes Tracking

**What:** Build an executive summary view showing financial impact (projected savings from readmission prevention, care gap closure), quality improvement trends, and programme ROI. This is the view used by CMOs, VP Care Management, and ACO administrators.

**Design:**

```typescript
// apps/api/src/modules/analytics/executive-summary.service.ts
@Injectable()
export class ExecutiveSummaryService {
  async getSummary(tenantId: string, period: DateRange): Promise<ExecutiveSummary> {
    const readmissionReduction = await this.calculateReadmissionImpact(tenantId, period);
    const careGapImpact = await this.calculateCareGapImpact(tenantId, period);
    const sdohImpact = await this.calculateSdohImpact(tenantId, period);
    
    return {
      financialImpact: {
        projectedSavingsFromReadmissionPrevention: readmissionReduction.savings,
        readmissionRateReduction: readmissionReduction.rateChange,
        qualityBonusProjection: careGapImpact.bonusProjection,
      },
      operationalMetrics: {
        patientsManaged: await this.patientRepo.countActive(tenantId),
        careGapsClosed: careGapImpact.closedCount,
        careGapClosureRate: careGapImpact.closureRate,
        avgTimeToGapClosure: careGapImpact.avgDays,
        sdohNeedsResolved: sdohImpact.resolvedCount,
        referralCompletionRate: await this.referralService.getCompletionRate(tenantId, period),
      },
      aiMetrics: {
        autonomousGapsClosed: careGapImpact.aiClosedCount,
        documentationTimeSaved: careGapImpact.docTimeSavedHours,
        copilotActionsAccepted: await this.copilotService.getAcceptanceRate(tenantId, period),
      },
    };
  }
}
```

**Testing:**
- [ ] Financial impact: readmission savings calculated from actual vs predicted readmission counts
- [ ] Quality bonus projection: based on current measure rates vs benchmark thresholds
- [ ] Care gap closure rate: closed / (closed + open) for the period
- [ ] AI metrics: autonomous gaps closed counted separately from human-closed gaps
- [ ] Documentation time saved: hours saved by AI Copilot documentation drafting
- [ ] All metrics verified against manual calculations on a test dataset

---

### Phase 12 Definition of Done

- [ ] Population health dashboard with risk, gaps, SDOH, utilisation, referral analytics
- [ ] Executive summary with financial impact, operational metrics, and AI effectiveness
- [ ] All charts support drill-down by organisation, provider, programme, payer
- [ ] Date range filtering across all analytics views
- [ ] Export capabilities (CSV, PNG) for all data and charts
- [ ] Dashboard performance: under 3 seconds for populations up to 100K patients
- [ ] Analytics data refreshed nightly from operational database

---

## Definition of Done (Global)

Every phase must meet these criteria before it is considered complete:

### Code Quality
- [ ] All new code has >= 80% unit test coverage
- [ ] Integration tests cover all API endpoints
- [ ] TypeScript strict mode with no `any` types in production code
- [ ] ESLint and Prettier pass with zero warnings
- [ ] No known security vulnerabilities in dependencies (npm audit clean)

### Security and Compliance
- [ ] All PHI access logged to audit_log with patient_id
- [ ] RLS active on all clinical data tables; cross-tenant access impossible
- [ ] Encryption at rest (AES-256) and in transit (TLS 1.3) for all PHI
- [ ] MFA enforced for all clinical user accounts
- [ ] No PHI in application logs, error messages, or stack traces
- [ ] All external API calls use BAA-covered services

### Interoperability
- [ ] All clinical data available via FHIR R4 API from the HAPI FHIR server
- [ ] FHIR resources conform to US Core STU8 profiles
- [ ] SDOH data conforms to Gravity SDOH Clinical Care IG
- [ ] Clinical data exchange conforms to Da Vinci CDex

### Performance
- [ ] API response time P95 < 500ms for operational queries
- [ ] Worklist page load < 1 second for care manager with 200 patients
- [ ] Batch processing (risk scoring, care gaps, measure calculation) completes within SLA
- [ ] Database queries use indexes; no full table scans on clinical data tables

### Documentation
- [ ] OpenAPI/Swagger documentation generated and published for all REST endpoints
- [ ] FHIR CapabilityStatement accurate and up to date
- [ ] Architecture Decision Records (ADRs) documented for all technology choices
- [ ] Deployment runbook for Kubernetes deployment

### Observability
- [ ] Structured logging (JSON) for all application components
- [ ] Health check endpoints for all services
- [ ] Kafka consumer lag monitoring with alerts
- [ ] Database connection pool monitoring
- [ ] Error rate alerting (> 1% error rate triggers alert)
