# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Healthcare Staff Scheduling (423)
> Approach: PostgreSQL with strategic JSONB columns for variable/complex data
> Generated: 2026-05-25

---

## Summary

A pragmatic hybrid approach that uses normalized relational tables for core entities with stable schemas (staff, facilities, credentials, shift assignments) while leveraging PostgreSQL's native JSONB type for data that is inherently variable, configuration-heavy, or deeply nested: scheduling constraint definitions, union contract rules, credential requirement matrices, AI/ML model outputs, and facility-specific policy configurations. This avoids the "EAV anti-pattern" trap that fully normalized constraint models fall into, while retaining the referential integrity and query power of relational design where it matters most.

PostgreSQL JSONB provides binary-stored, indexable JSON with GIN indexes, containment operators, and path-based queries -- giving document-database flexibility inside a fully ACID-compliant relational engine. This eliminates the need for a separate document store while keeping the operational simplicity of a single database.

---

## Design Principles

1. **Relational for identity, relationships, and high-query columns.** Staff, facilities, credentials, and shift assignments use typed columns with foreign keys.
2. **JSONB for variability, configuration, and extensibility.** Scheduling rules, constraint parameters, shift requirements, and ML outputs use JSONB columns within relational tables.
3. **GIN indexes on JSONB.** Enable fast lookups on frequently queried JSON paths without sacrificing write performance.
4. **Application-level validation for JSONB.** Use JSON Schema validation at the API layer since PostgreSQL cannot enforce complex JSONB structure via native constraints.

---

## Key Entities and Schema

### Organizational Structure (Relational)

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL CHECK (type IN ('health_system', 'hospital', 'clinic', 'staffing_agency')),
    parent_org_id   UUID REFERENCES organizations(id),
    settings        JSONB NOT NULL DEFAULT '{}',  -- org-wide configuration
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- settings JSONB example:
-- {
--   "default_timezone": "America/Chicago",
--   "pay_periods": "biweekly",
--   "overtime_threshold_weekly": 40,
--   "overtime_threshold_daily": null,
--   "agency_approval_required": true,
--   "credential_renewal_alert_days": [90, 60, 30, 14, 7]
-- }

CREATE TABLE facilities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    state           CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    address         JSONB,          -- structured address without separate columns
    labor_rules     JSONB NOT NULL DEFAULT '{}',  -- state/facility-specific labor regulations
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- labor_rules JSONB example:
-- {
--   "state_regulations": {
--     "max_consecutive_hours": 16,
--     "min_rest_between_shifts_hours": 8,
--     "mandatory_meal_break_after_hours": 5,
--     "meal_break_duration_minutes": 30,
--     "max_days_without_day_off": 6,
--     "nurse_patient_ratio": {"icu": 2, "med_surg": 5, "er": 4}
--   },
--   "facility_policies": {
--     "overtime_pre_approval_required": true,
--     "float_pool_orientation_required": true,
--     "agency_credential_lead_time_days": 3
--   }
-- }

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(id),
    name            TEXT NOT NULL,
    code            VARCHAR(20),
    unit_type       TEXT NOT NULL,
    staffing_config JSONB NOT NULL DEFAULT '{}',  -- department-specific staffing rules
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- staffing_config JSONB example:
-- {
--   "default_ratios": {
--     "rn": {"patients_per_nurse": 4, "min_per_shift": 3},
--     "cna": {"patients_per_cna": 8, "min_per_shift": 2},
--     "charge_rn": {"min_per_shift": 1}
--   },
--   "acuity_adjustments": {
--     "high_acuity_threshold": 3.5,
--     "high_acuity_ratio_reduction": 0.75
--   },
--   "shift_types": ["day_12", "night_12", "day_8", "evening_8", "night_8"],
--   "skill_requirements": {
--     "required": ["bls"],
--     "preferred": ["acls", "pals"]
--   }
-- }
```

### Staff Members (Relational + JSONB for preferences)

```sql
CREATE TABLE staff_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_number VARCHAR(50) UNIQUE,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT UNIQUE,
    phone           VARCHAR(20),
    hire_date       DATE NOT NULL,
    termination_date DATE,
    employment_type TEXT NOT NULL CHECK (employment_type IN (
        'full_time', 'part_time', 'per_diem', 'agency', 'travel', 'resident'
    )),
    fte_target      NUMERIC(3,2) DEFAULT 1.00,
    home_facility_id UUID REFERENCES facilities(id),
    home_department_id UUID REFERENCES departments(id),
    hourly_rate     NUMERIC(10,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB for variable/extensible data
    roles           JSONB NOT NULL DEFAULT '[]',       -- array of role objects
    skills          JSONB NOT NULL DEFAULT '[]',       -- array of skill objects
    preferences     JSONB NOT NULL DEFAULT '{}',       -- scheduling preferences
    contact_info    JSONB NOT NULL DEFAULT '{}',       -- flexible contact fields
    metadata        JSONB NOT NULL DEFAULT '{}',       -- extensible attributes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- roles JSONB example:
-- [
--   {"role": "rn", "specialty": "critical_care", "is_primary": true, "effective_date": "2020-01-15"},
--   {"role": "charge_rn", "is_primary": false, "effective_date": "2023-06-01"}
-- ]

-- preferences JSONB example:
-- {
--   "preferred_shifts": ["day_12"],
--   "unavailable_days": ["saturday"],
--   "max_nights_per_schedule": 4,
--   "float_willing": true,
--   "float_facilities": ["facility-002", "facility-003"],
--   "notification_preferences": {
--     "open_shifts": true,
--     "schedule_changes": true,
--     "channels": ["push", "sms"]
--   }
-- }

-- GIN index for querying roles and skills
CREATE INDEX idx_staff_roles ON staff_members USING GIN (roles);
CREATE INDEX idx_staff_skills ON staff_members USING GIN (skills);
CREATE INDEX idx_staff_preferences ON staff_members USING GIN (preferences);
```

### Credentials (Relational core + JSONB details)

```sql
CREATE TABLE staff_credentials (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id            UUID NOT NULL REFERENCES staff_members(id),
    credential_category TEXT NOT NULL CHECK (credential_category IN (
        'license', 'certification', 'dea', 'privilege',
        'background_check', 'immunization', 'cme'
    )),
    credential_type     TEXT NOT NULL,     -- e.g., 'rn_license', 'acls', 'bls'
    credential_number   VARCHAR(100),
    issuing_authority   TEXT,
    issuing_state       CHAR(2),
    issue_date          DATE NOT NULL,
    expiration_date     DATE,
    status              TEXT NOT NULL CHECK (status IN (
        'active', 'pending', 'expired', 'suspended', 'revoked', 'renewal_pending'
    )),
    -- JSONB for verification details and variable fields
    verification        JSONB NOT NULL DEFAULT '{}',
    restrictions        JSONB NOT NULL DEFAULT '[]',
    renewal_history     JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- verification JSONB example:
-- {
--   "primary_source_verified": true,
--   "psv_date": "2026-01-15T14:30:00Z",
--   "psv_method": "online_board_lookup",
--   "psv_source": "ca_brn_website",
--   "verified_by": "user-cred-001",
--   "npdb_queried": true,
--   "npdb_query_date": "2026-01-15",
--   "npdb_findings": "none",
--   "sanctions_checked": true,
--   "oig_exclusion_clear": true
-- }

-- restrictions JSONB example:
-- [
--   {"type": "supervision_required", "detail": "Must work under MD supervision for procedures", "effective_date": "2025-06-01"},
--   {"type": "facility_limited", "detail": "Credentialed at Main Campus only", "facilities": ["facility-001"]}
-- ]

CREATE INDEX idx_credentials_staff ON staff_credentials (staff_id);
CREATE INDEX idx_credentials_expiry ON staff_credentials (expiration_date) WHERE status = 'active';
CREATE INDEX idx_credentials_status ON staff_credentials (status);

CREATE TABLE facility_privileges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    facility_id     UUID NOT NULL REFERENCES facilities(id),
    privilege_scope JSONB NOT NULL,  -- what they're privileged to do, varies by role
    status          TEXT NOT NULL CHECK (status IN ('active', 'provisional', 'suspended', 'revoked', 'expired')),
    granted_date    DATE NOT NULL,
    expiration_date DATE,
    conditions      JSONB NOT NULL DEFAULT '[]',  -- facility-specific conditions
    approved_by     UUID REFERENCES staff_members(id),
    UNIQUE (staff_id, facility_id)
);

-- privilege_scope JSONB example:
-- {
--   "service_lines": ["critical_care", "emergency", "med_surg"],
--   "procedures": ["central_line", "intubation", "chest_tube"],
--   "supervision_level": "independent",
--   "telehealth_eligible": true
-- }
```

### Scheduling Constraints (JSONB-first)

```sql
CREATE TABLE scheduling_rule_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    scope_type      TEXT NOT NULL CHECK (scope_type IN ('system', 'organization', 'facility', 'department', 'role')),
    scope_id        UUID,             -- references the applicable entity
    rule_source     TEXT NOT NULL CHECK (rule_source IN (
        'federal_law', 'state_law', 'union_contract', 'facility_policy', 'department_policy'
    )),
    priority        INTEGER NOT NULL DEFAULT 100,  -- higher = overrides lower
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_date  DATE NOT NULL,
    end_date        DATE,
    -- The rules themselves, as structured JSONB
    rules           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- rules JSONB example (union contract):
-- {
--   "contract_name": "SEIU Local 1021 - 2025-2028",
--   "constraints": [
--     {
--       "id": "uc-001",
--       "type": "max_consecutive_days",
--       "value": 5,
--       "is_hard": true,
--       "applies_to": ["rn", "lpn", "cna"],
--       "description": "No more than 5 consecutive work days"
--     },
--     {
--       "id": "uc-002",
--       "type": "min_rest_between_shifts",
--       "value_hours": 11,
--       "is_hard": true,
--       "applies_to": ["rn", "lpn", "cna"],
--       "description": "Minimum 11 hours between end of shift and start of next"
--     },
--     {
--       "id": "uc-003",
--       "type": "weekend_distribution",
--       "max_weekends_per_schedule": 2,
--       "schedule_length_weeks": 4,
--       "is_hard": false,
--       "weight": 80,
--       "applies_to": ["rn", "lpn"],
--       "description": "Target max 2 weekends per 4-week schedule"
--     },
--     {
--       "id": "uc-004",
--       "type": "holiday_rotation",
--       "method": "round_robin",
--       "holiday_list": ["new_years", "memorial_day", "july_4th", "labor_day", "thanksgiving", "christmas"],
--       "is_hard": true,
--       "description": "Equitable holiday rotation"
--     },
--     {
--       "id": "uc-005",
--       "type": "shift_differential",
--       "night_premium_pct": 15,
--       "weekend_premium_pct": 10,
--       "holiday_premium_pct": 50,
--       "description": "Pay differentials for non-standard shifts"
--     }
--   ]
-- }

CREATE INDEX idx_rule_sets_scope ON scheduling_rule_sets (scope_type, scope_id) WHERE is_active = true;
```

### Shift Definitions and Assignments (Relational + JSONB metadata)

```sql
CREATE TABLE shift_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id   UUID NOT NULL REFERENCES departments(id),
    name            TEXT NOT NULL,
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    duration_hours  NUMERIC(4,2) NOT NULL,
    shift_type      TEXT NOT NULL CHECK (shift_type IN ('regular', 'on_call', 'standby', 'callback')),
    staffing_requirements JSONB NOT NULL,  -- flexible per-role staffing needs
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- staffing_requirements JSONB example:
-- {
--   "roles": [
--     {"role": "rn", "min": 4, "max": 6, "credentials_required": ["bls", "acls"]},
--     {"role": "charge_rn", "min": 1, "max": 1, "credentials_required": ["bls", "acls"]},
--     {"role": "cna", "min": 2, "max": 3, "credentials_required": ["bls"]},
--     {"role": "clerk", "min": 1, "max": 1, "credentials_required": []}
--   ],
--   "skill_preferences": ["iv_therapy", "wound_care"],
--   "experience_min_months": 6
-- }

CREATE TABLE schedule_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id   UUID NOT NULL REFERENCES departments(id),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('draft', 'published', 'locked', 'archived')),
    generation_params JSONB,  -- parameters used by the scheduling solver
    solver_results  JSONB,    -- optimization metrics from the last solver run
    created_by      UUID NOT NULL REFERENCES staff_members(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- solver_results JSONB example:
-- {
--   "solver": "or_tools_cp_sat",
--   "solve_time_seconds": 12.4,
--   "objective_score": 0.94,
--   "hard_constraints_satisfied": true,
--   "soft_constraint_violations": [
--     {"rule_id": "uc-003", "staff_id": "staff-042", "detail": "3 weekends in schedule period"}
--   ],
--   "fairness_metrics": {
--     "weekend_gini_coefficient": 0.08,
--     "night_shift_gini_coefficient": 0.12,
--     "holiday_distribution_variance": 0.05
--   }
-- }

CREATE TABLE shift_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_period_id UUID NOT NULL REFERENCES schedule_periods(id),
    shift_template_id UUID NOT NULL REFERENCES shift_templates(id),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    shift_date      DATE NOT NULL,
    status          TEXT NOT NULL CHECK (status IN (
        'scheduled', 'confirmed', 'in_progress', 'completed', 'no_show', 'cancelled'
    )),
    assignment_type TEXT NOT NULL CHECK (assignment_type IN (
        'regular', 'overtime', 'float', 'agency', 'swap', 'callback'
    )),
    cost_tier       TEXT,
    -- JSONB for assignment-specific context
    assignment_context JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (staff_id, shift_date, shift_template_id)
);

-- assignment_context JSONB example:
-- {
--   "assigned_by": "user-mgr-001",
--   "assignment_method": "solver_generated",
--   "credentials_verified_at": "2026-05-20T14:00:00Z",
--   "credentials_checked": ["rn_license_ca", "bls", "acls"],
--   "float_from_department": "dept-med-surg-02",
--   "swap_from_staff": null,
--   "overtime_pre_approved": true,
--   "overtime_approved_by": "user-mgr-001"
-- }
```

### Open Shifts Marketplace (Relational + JSONB)

```sql
CREATE TABLE open_shifts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shift_template_id UUID NOT NULL REFERENCES shift_templates(id),
    shift_date      DATE NOT NULL,
    reason          TEXT NOT NULL,
    priority        TEXT NOT NULL DEFAULT 'normal',
    status          TEXT NOT NULL CHECK (status IN ('open', 'claimed', 'filled', 'cancelled', 'expired')),
    eligibility_criteria JSONB NOT NULL DEFAULT '{}',  -- dynamic eligibility rules
    posted_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    filled_by       UUID REFERENCES staff_members(id),
    filled_at       TIMESTAMPTZ
);

-- eligibility_criteria JSONB example:
-- {
--   "required_roles": ["rn"],
--   "required_credentials": ["bls", "acls"],
--   "required_skills": ["iv_therapy"],
--   "max_cost_tier": "float_pool",
--   "exclude_overtime": false,
--   "facility_privilege_required": "facility-001",
--   "min_experience_months": 12,
--   "exclude_staff": ["staff-022"]
-- }
```

### AI/ML Model Outputs (JSONB-heavy)

```sql
CREATE TABLE ml_predictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_type      TEXT NOT NULL CHECK (model_type IN (
        'demand_forecast', 'burnout_risk', 'callout_prediction',
        'turnover_risk', 'fatigue_score', 'optimal_assignment'
    )),
    subject_type    TEXT NOT NULL,     -- 'staff', 'department', 'shift'
    subject_id      UUID NOT NULL,
    prediction_date DATE NOT NULL,
    model_version   TEXT NOT NULL,
    -- JSONB for model-specific prediction structure
    prediction      JSONB NOT NULL,
    confidence      NUMERIC(4,3),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- prediction JSONB examples:
-- Burnout risk:
-- {
--   "risk_score": 0.78,
--   "risk_level": "high",
--   "contributing_factors": [
--     {"factor": "consecutive_night_shifts", "impact": 0.35},
--     {"factor": "overtime_hours_30d", "impact": 0.25},
--     {"factor": "weekend_frequency", "impact": 0.18}
--   ],
--   "recommended_interventions": [
--     "reduce_night_shifts_next_period",
--     "schedule_consecutive_days_off"
--   ]
-- }
--
-- Demand forecast:
-- {
--   "department_id": "dept-er-01",
--   "forecast_date": "2026-06-15",
--   "shifts": {
--     "day": {"predicted_census": 42, "recommended_rn": 8, "recommended_cna": 4},
--     "night": {"predicted_census": 35, "recommended_rn": 7, "recommended_cna": 3}
--   },
--   "confidence_interval_95": {"census_low": 35, "census_high": 49}
-- }

CREATE INDEX idx_ml_predictions_subject ON ml_predictions (subject_type, subject_id, prediction_date DESC);
CREATE INDEX idx_ml_predictions_type ON ml_predictions (model_type, prediction_date DESC);
```

### Audit Log (Relational + JSONB payload)

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL CHECK (action IN ('create', 'update', 'delete', 'access', 'approve', 'reject')),
    actor_id        UUID NOT NULL REFERENCES staff_members(id),
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB captures the full change context without rigid schema
    changes         JSONB NOT NULL,
    ip_address      INET,
    user_agent      TEXT
) PARTITION BY RANGE (occurred_at);

-- changes JSONB example:
-- {
--   "field_changes": {
--     "status": {"old": "draft", "new": "published"},
--     "published_at": {"old": null, "new": "2026-05-20T08:00:00Z"}
--   },
--   "context": {
--     "reason": "Bi-weekly schedule for ICU",
--     "staff_notified": 47
--   }
-- }

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log (actor_id);
```

---

## Pros

- **Best of both worlds.** Core relationships and frequently filtered columns get full relational integrity (foreign keys, unique constraints, check constraints). Variable, policy-driven, and extensible data uses JSONB without schema migrations.
- **Single database engine.** PostgreSQL handles both relational and document workloads. No need to operate a separate MongoDB or document store. One backup strategy, one connection pool, one monitoring stack.
- **Constraint modeling flexibility.** Union contracts, state labor laws, and facility policies have wildly different structures. JSONB lets each rule set define its own constraint schema without a table-per-rule-type or EAV pattern.
- **AI/ML extensibility.** New ML models (burnout prediction, demand forecasting, fatigue scoring) produce different output structures. JSONB predictions can evolve without schema migrations.
- **GIN-indexed queries.** PostgreSQL GIN indexes on JSONB support fast containment queries: "find all staff with role 'rn' and skill 'acls'" via `staff_members.roles @> '[{"role": "rn"}]'`.
- **Solver integration.** The constraint solver can load scheduling_rule_sets.rules as JSON directly, without joining dozens of normalized parameter tables. Rules are already in a structure the solver can parse.
- **Reduced migration friction.** Adding a new field to preferences, verification details, or staffing requirements is a JSON key addition, not a database migration.

## Cons

- **Weaker enforcement on JSONB data.** Foreign keys, unique constraints, and NOT NULL cannot be applied inside JSONB columns. Invalid JSON structures must be caught by application-level validation (JSON Schema) rather than the database.
- **Reporting complexity.** Querying deeply nested JSONB requires `jsonb_path_query`, `->>`, and `#>>` operators that are less familiar to analysts and BI tools than flat column queries.
- **Performance edge cases.** Complex JSONB path queries with multiple conditions can be slower than equivalent queries on indexed relational columns. Careful GIN index design and query planning are required.
- **Schema documentation burden.** JSONB columns need documented JSON Schema definitions maintained separately from the database DDL. Without this, the "flexible" JSONB fields become a dumping ground.
- **Testing difficulty.** JSONB structure validation must be tested at the application layer. Bugs in JSON schema changes can introduce subtle data quality issues.
- **ORM partial support.** Most ORMs handle JSONB but with reduced type safety compared to regular columns. Custom query builders or raw SQL may be needed for complex JSONB operations.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with GIN indexes on all JSONB columns |
| **Schema Validation** | JSON Schema (Draft 2020-12) enforced at API layer using Ajv (JS) or jsonschema (Python) |
| **ORM** | Prisma with custom JSONB type mappings, or Drizzle ORM with json() columns |
| **API** | REST with OpenAPI 3.1 documenting both relational and JSONB field structures |
| **Constraint Solver** | Google OR-Tools CP-SAT solver, consuming rules JSONB directly as solver input |
| **Search** | PostgreSQL GIN indexes for primary queries; consider pg_trgm for fuzzy staff name search |
| **Caching** | Redis for schedule snapshots, materialized views for analytics dashboards |

---

## Migration and Scaling Considerations

- **Partition strategy.** Partition `shift_assignments` and `audit_log` by date range. JSONB columns do not affect partitioning.
- **JSONB schema versioning.** Add a `schema_version` field inside JSONB objects (e.g., `{"_v": 2, "constraints": [...]}`) to support backward-compatible evolution. Application code handles version migration on read.
- **Gradual JSONB adoption.** Start with relational-only tables and promote fields to JSONB as variability becomes apparent. Fields that stabilize can be "graduated" back to typed columns via migration.
- **Read replicas for analytics.** JSONB extraction queries for dashboards can be expensive. Route analytics to read replicas with materialized views that flatten JSONB into columnar structures.
- **Multi-tenancy.** Same approach as fully relational: organization_id with RLS. JSONB contents are tenant-specific (different union contracts, different state regulations) without schema differences.
- **Data migration from legacy systems.** JSONB columns can initially hold raw imported data from legacy scheduling systems, then be gradually normalized as field mappings stabilize. This reduces the upfront migration burden.
