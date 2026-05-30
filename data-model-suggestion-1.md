# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Healthcare Staff Scheduling (423)
> Approach: Fully normalized relational schema in PostgreSQL
> Generated: 2026-05-25

---

## Summary

A traditional normalized relational database design (3NF/BCNF) in PostgreSQL, with clearly defined entity-relationship mappings across all core domains: organizations, staff, credentials, shifts, schedules, and compliance. This approach maximizes data integrity through foreign keys, check constraints, and database-level enforcement of business rules. It is the most conventional and well-understood approach, with the deepest ecosystem of tooling, ORM support, and developer familiarity.

---

## Key Entities and Relationships

### Organizational Structure

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL CHECK (type IN ('health_system', 'hospital', 'clinic', 'staffing_agency')),
    parent_org_id   UUID REFERENCES organizations(id),
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE facilities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state           CHAR(2),
    zip             VARCHAR(10),
    timezone        TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID NOT NULL REFERENCES facilities(id),
    name            TEXT NOT NULL,
    code            VARCHAR(20),
    unit_type       TEXT CHECK (unit_type IN ('med_surg', 'icu', 'er', 'or', 'l_and_d', 'nicu', 'psych', 'outpatient', 'other')),
    target_nurse_ratio NUMERIC(3,1),  -- e.g., 4.0 patients per nurse
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Staff and Roles

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
    employment_type TEXT NOT NULL CHECK (employment_type IN ('full_time', 'part_time', 'per_diem', 'agency', 'travel', 'resident')),
    fte_target      NUMERIC(3,2) DEFAULT 1.00,  -- 0.00 to 1.00
    home_facility_id UUID REFERENCES facilities(id),
    home_department_id UUID REFERENCES departments(id),
    hourly_rate     NUMERIC(10,2),
    overtime_rate   NUMERIC(10,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    role_type       TEXT NOT NULL CHECK (role_type IN (
        'rn', 'lpn', 'cna', 'np', 'pa', 'md', 'do',
        'crna', 'rt', 'pharmacist', 'tech', 'clerk', 'other'
    )),
    specialty       TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    effective_date  DATE NOT NULL,
    end_date        DATE,
    UNIQUE (staff_id, role_type, effective_date)
);

CREATE TABLE staff_skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    skill_code      VARCHAR(50) NOT NULL,
    skill_name      TEXT NOT NULL,
    proficiency     TEXT CHECK (proficiency IN ('basic', 'competent', 'proficient', 'expert')),
    verified_date   DATE,
    expiration_date DATE,
    verified_by     UUID REFERENCES staff_members(id)
);
```

### Credentialing and Privileges

```sql
CREATE TABLE credential_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) UNIQUE NOT NULL,
    name            TEXT NOT NULL,
    category        TEXT NOT NULL CHECK (category IN ('license', 'certification', 'dea', 'privilege', 'background_check', 'immunization', 'cme')),
    issuing_body    TEXT,
    renewal_period_months INTEGER,
    is_required_for_practice BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE staff_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    credential_type_id UUID NOT NULL REFERENCES credential_types(id),
    credential_number VARCHAR(100),
    issuing_state   CHAR(2),
    issue_date      DATE NOT NULL,
    expiration_date DATE,
    status          TEXT NOT NULL CHECK (status IN ('active', 'pending', 'expired', 'suspended', 'revoked', 'renewal_pending')),
    primary_source_verified BOOLEAN NOT NULL DEFAULT false,
    psv_date        TIMESTAMPTZ,
    psv_method      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE facility_privileges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    facility_id     UUID NOT NULL REFERENCES facilities(id),
    privilege_type  TEXT NOT NULL,
    service_line    TEXT,
    status          TEXT NOT NULL CHECK (status IN ('active', 'provisional', 'suspended', 'revoked', 'expired')),
    granted_date    DATE NOT NULL,
    expiration_date DATE,
    approved_by     UUID REFERENCES staff_members(id),
    UNIQUE (staff_id, facility_id, privilege_type)
);

CREATE TABLE credential_requirements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_type       TEXT NOT NULL,
    facility_id     UUID REFERENCES facilities(id),  -- NULL = system-wide
    department_id   UUID REFERENCES departments(id),  -- NULL = facility-wide
    credential_type_id UUID NOT NULL REFERENCES credential_types(id),
    is_mandatory    BOOLEAN NOT NULL DEFAULT true
);
```

### Shift Definitions and Schedules

```sql
CREATE TABLE shift_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id   UUID NOT NULL REFERENCES departments(id),
    name            TEXT NOT NULL,               -- e.g., "Day 7a-7p", "Night 7p-7a"
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    duration_hours  NUMERIC(4,2) NOT NULL,
    shift_type      TEXT NOT NULL CHECK (shift_type IN ('regular', 'on_call', 'standby', 'callback')),
    required_roles  TEXT[] NOT NULL,              -- e.g., {'rn', 'cna'}
    min_staff       INTEGER NOT NULL DEFAULT 1,
    max_staff       INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE schedule_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id   UUID NOT NULL REFERENCES departments(id),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('draft', 'published', 'locked', 'archived')),
    created_by      UUID NOT NULL REFERENCES staff_members(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (end_date > start_date)
);

CREATE TABLE shift_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_period_id UUID NOT NULL REFERENCES schedule_periods(id),
    shift_template_id UUID NOT NULL REFERENCES shift_templates(id),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    shift_date      DATE NOT NULL,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    status          TEXT NOT NULL CHECK (status IN ('scheduled', 'confirmed', 'in_progress', 'completed', 'no_show', 'cancelled')),
    assignment_type TEXT NOT NULL CHECK (assignment_type IN ('regular', 'overtime', 'float', 'agency', 'swap', 'callback')),
    cost_tier       TEXT CHECK (cost_tier IN ('regular', 'overtime', 'float_pool', 'agency', 'premium')),
    assigned_by     UUID REFERENCES staff_members(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT,
    UNIQUE (staff_id, shift_date, shift_template_id)
);
```

### Open Shifts and Shift Marketplace

```sql
CREATE TABLE open_shifts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shift_template_id UUID NOT NULL REFERENCES shift_templates(id),
    shift_date      DATE NOT NULL,
    reason          TEXT CHECK (reason IN ('understaffed', 'callout', 'census_increase', 'new_shift', 'vacancy')),
    priority        TEXT CHECK (priority IN ('critical', 'high', 'normal', 'low')),
    cost_tier_max   TEXT,  -- maximum cost tier willing to fill at
    posted_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    filled_by       UUID REFERENCES staff_members(id),
    filled_at       TIMESTAMPTZ,
    status          TEXT NOT NULL CHECK (status IN ('open', 'claimed', 'filled', 'cancelled', 'expired'))
);

CREATE TABLE shift_bids (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    open_shift_id   UUID NOT NULL REFERENCES open_shifts(id),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    bid_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL CHECK (status IN ('pending', 'approved', 'rejected', 'withdrawn')),
    reviewed_by     UUID REFERENCES staff_members(id),
    reviewed_at     TIMESTAMPTZ,
    rejection_reason TEXT,
    UNIQUE (open_shift_id, staff_id)
);
```

### Scheduling Constraints and Rules

```sql
CREATE TABLE scheduling_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id     UUID REFERENCES facilities(id),
    department_id   UUID REFERENCES departments(id),
    rule_name       TEXT NOT NULL,
    rule_category   TEXT NOT NULL CHECK (rule_category IN ('labor_law', 'union_contract', 'fatigue', 'organizational', 'credential')),
    description     TEXT,
    is_hard_constraint BOOLEAN NOT NULL DEFAULT true,  -- hard = must enforce; soft = prefer
    priority        INTEGER DEFAULT 100,               -- for soft constraints
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- Examples of parameterized constraint values
CREATE TABLE scheduling_rule_parameters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES scheduling_rules(id),
    parameter_name  TEXT NOT NULL,      -- e.g., 'max_consecutive_hours', 'min_rest_between_shifts'
    parameter_value TEXT NOT NULL,      -- stored as text, parsed by application
    parameter_unit  TEXT               -- e.g., 'hours', 'days', 'shifts'
);
```

### Time and Attendance

```sql
CREATE TABLE time_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    shift_assignment_id UUID REFERENCES shift_assignments(id),
    clock_in        TIMESTAMPTZ NOT NULL,
    clock_out       TIMESTAMPTZ,
    break_minutes   INTEGER DEFAULT 0,
    total_hours     NUMERIC(5,2) GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (clock_out - clock_in)) / 3600.0 - break_minutes / 60.0
    ) STORED,
    status          TEXT NOT NULL CHECK (status IN ('clocked_in', 'clocked_out', 'approved', 'disputed', 'adjusted')),
    approved_by     UUID REFERENCES staff_members(id),
    approved_at     TIMESTAMPTZ
);

CREATE TABLE overtime_tracking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    pay_period_start DATE NOT NULL,
    pay_period_end  DATE NOT NULL,
    regular_hours   NUMERIC(6,2) NOT NULL DEFAULT 0,
    overtime_hours  NUMERIC(6,2) NOT NULL DEFAULT 0,
    double_time_hours NUMERIC(6,2) NOT NULL DEFAULT 0,
    status          TEXT CHECK (status IN ('tracking', 'closed', 'exported')),
    UNIQUE (staff_id, pay_period_start)
);
```

### Staff Availability and Requests

```sql
CREATE TABLE staff_availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    day_of_week     SMALLINT CHECK (day_of_week BETWEEN 0 AND 6),  -- 0=Sunday
    available_from  TIME,
    available_to    TIME,
    preference      TEXT CHECK (preference IN ('preferred', 'available', 'unavailable')),
    effective_date  DATE NOT NULL,
    end_date        DATE
);

CREATE TABLE time_off_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_id        UUID NOT NULL REFERENCES staff_members(id),
    request_type    TEXT NOT NULL CHECK (request_type IN ('pto', 'sick', 'fmla', 'bereavement', 'jury_duty', 'military', 'unpaid', 'other')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('pending', 'approved', 'denied', 'cancelled')),
    reason          TEXT,
    reviewed_by     UUID REFERENCES staff_members(id),
    reviewed_at     TIMESTAMPTZ,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE shift_swap_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requester_assignment_id UUID NOT NULL REFERENCES shift_assignments(id),
    target_assignment_id UUID REFERENCES shift_assignments(id),  -- NULL if seeking any taker
    target_staff_id UUID REFERENCES staff_members(id),
    status          TEXT NOT NULL CHECK (status IN ('pending_peer', 'pending_manager', 'approved', 'denied', 'cancelled')),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    approved_by     UUID REFERENCES staff_members(id),
    approved_at     TIMESTAMPTZ
);
```

### Census and Staffing Ratios

```sql
CREATE TABLE census_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id   UUID NOT NULL REFERENCES departments(id),
    snapshot_time   TIMESTAMPTZ NOT NULL,
    patient_count   INTEGER NOT NULL,
    acuity_average  NUMERIC(3,1),
    required_rn     INTEGER,
    required_cna    INTEGER,
    required_other  INTEGER,
    source          TEXT CHECK (source IN ('ehr_integration', 'manual_entry', 'predicted'))
);
```

---

## Entity-Relationship Overview

```
organizations 1──* facilities 1──* departments
                                       │
                                  1──* shift_templates
                                  1──* schedule_periods 1──* shift_assignments *──1 staff_members
                                  1──* census_snapshots                             │
                                                                              1──* staff_credentials
                                                                              1──* facility_privileges
                                                                              1──* staff_roles
                                                                              1──* staff_skills
                                                                              1──* staff_availability
                                                                              1──* time_entries
                                                                              1──* time_off_requests

shift_assignments *──1 shift_templates
open_shifts *──1 shift_templates
open_shifts 1──* shift_bids *──1 staff_members

scheduling_rules 1──* scheduling_rule_parameters
credential_types 1──* staff_credentials
credential_types 1──* credential_requirements
```

---

## Pros

- **Data integrity at the database level.** Foreign keys, check constraints, and unique constraints prevent invalid states such as double-booking, assigning staff to non-existent departments, or credential status mismatches.
- **Mature ecosystem.** PostgreSQL has decades of tooling for ORMs, migrations, backups, monitoring, and replication. Every major web framework has first-class PostgreSQL support.
- **Query flexibility.** Complex reporting queries (overtime analysis across departments, credential expiration reports, staffing ratio compliance) are straightforward with SQL joins and window functions.
- **ACID compliance.** Schedule publishing, shift swaps, and credential updates involve multi-table transactions that must be atomic. PostgreSQL handles this natively.
- **Well-understood scaling patterns.** Read replicas, connection pooling (PgBouncer), table partitioning (by date on shift_assignments), and materialized views for dashboards.
- **Regulatory audit support.** Structured, typed columns make it easy to build audit triggers and demonstrate compliance to regulators and accreditors.

## Cons

- **Schema rigidity for rules and constraints.** Union contracts, state-specific labor laws, and organizational policies vary enormously. Storing scheduling rules in normalized tables requires either a very generic parameter model (hard to validate) or frequent schema migrations.
- **Constraint solver impedance mismatch.** The NP-hard scheduling optimization engine needs to load constraints into memory for solving. Normalised tables require many joins to hydrate the solver's working set, adding latency and complexity.
- **Credential rule variability.** Different states, facilities, and specialties have different credentialing requirements. A fully normalized approach leads to a complex web of junction tables that can be difficult to maintain.
- **Performance at scale.** Queries joining staff, credentials, assignments, templates, and departments across large health systems (thousands of staff, millions of historical assignments) require careful indexing and potential denormalization.
- **Limited flexibility for AI/ML features.** Storing ML model outputs (burnout scores, callout probabilities, fatigue indices) requires adding columns or tables for each new feature.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with partitioning on shift_assignments by month |
| **ORM** | Prisma, Drizzle, or SQLAlchemy for type-safe schema management |
| **Migrations** | Flyway or Alembic for versioned schema migrations |
| **Connection pooling** | PgBouncer or Supavisor for high-concurrency mobile access |
| **Search** | PostgreSQL full-text search for staff/credential lookup; upgrade to Elasticsearch if needed |
| **Caching** | Redis for schedule snapshots and real-time shift availability |
| **Audit logging** | PostgreSQL audit triggers with a separate audit_log table |

---

## Migration and Scaling Considerations

- **Partitioning strategy.** Partition `shift_assignments` and `time_entries` by month using PostgreSQL declarative partitioning. Historical data older than 2 years can be moved to cold storage.
- **Multi-tenancy.** Use a shared-schema approach with `organization_id` tenant columns and Row-Level Security (RLS) policies for data isolation across health systems.
- **Read replicas.** Route analytics and reporting queries to read replicas to keep the primary database responsive for scheduling operations.
- **Materialized views.** Pre-compute common dashboard queries (overtime by department, credential expiration counts, staffing ratio compliance) as materialized views refreshed on a schedule.
- **Incremental migration.** Start with core tables (staff, credentials, shifts, assignments) and add analytics, marketplace, and AI-related tables as features are built. Each feature area is a separate migration.
- **Data import.** Build ETL pipelines for importing staff records from HRIS, credential data from verification services, and census data from EHR integrations. Use staging tables for validation before loading into production.
