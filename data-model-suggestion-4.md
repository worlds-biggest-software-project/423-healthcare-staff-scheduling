# Data Model Suggestion 4: Constraint-Solver-Centric with Temporal Model

> Project: Healthcare Staff Scheduling (423)
> Approach: Optimization-first architecture with bitemporal data, graph-based eligibility, and solver-native data structures
> Generated: 2026-05-25

---

## Summary

This approach designs the data model around the central technical challenge of healthcare scheduling: solving an NP-hard combinatorial optimization problem under thousands of simultaneous constraints. Instead of treating the database as a passive record store that a solver queries, this architecture structures the entire data layer to minimize the impedance mismatch between persistent storage and the constraint solver's working memory.

The design combines three specialized patterns:

1. **Solver-native constraint representation** -- scheduling constraints, staff eligibility rules, and coverage requirements stored in structures that map directly to solver variables and constraints (Google OR-Tools CP-SAT or similar).
2. **Bitemporal data model** -- every entity tracks both "valid time" (when the fact is true in the real world) and "transaction time" (when the database recorded the fact). This is critical for healthcare credentialing, where a license discovered to have lapsed retroactively affects the validity of past assignments.
3. **Graph-based eligibility model** -- staff-to-shift eligibility is modeled as a bipartite graph (staff nodes to shift-slot nodes) with edges weighted by credential match, cost tier, preference score, and fatigue risk. The solver traverses this graph rather than running eligibility queries at solve time.

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────┐
│                    API / UI Layer                      │
└──────────┬───────────────────────────────┬────────────┘
           │                               │
┌──────────▼──────────┐     ┌──────────────▼────────────┐
│  Transactional DB   │     │  Solver Service            │
│  (PostgreSQL)       │     │  (OR-Tools / OptaPlanner)  │
│                     │     │                            │
│  • Bitemporal staff │◄────┤  • Eligibility graph       │
│  • Credentials      │     │  • Constraint matrix       │
│  • Assignments      │     │  • Objective function      │
│  • Audit history    │     │  • Solution output         │
└──────────┬──────────┘     └──────────────┬────────────┘
           │                               │
┌──────────▼──────────┐     ┌──────────────▼────────────┐
│  Eligibility Graph  │     │  Schedule Store            │
│  (Redis / in-memory)│     │  (PostgreSQL)              │
│                     │     │                            │
│  • Staff ↔ Slot     │     │  • Generated schedules     │
│  • Weighted edges   │     │  • Solver metadata         │
│  • Real-time update │     │  • Fairness metrics        │
└─────────────────────┘     └───────────────────────────┘
```

---

## Key Entities and Schema

### Bitemporal Staff and Credential Model

Every staff attribute and credential uses bitemporal columns to track both real-world validity and database recording time.

```sql
-- Bitemporal staff records
CREATE TABLE staff_members (
    id                  UUID NOT NULL,
    employee_number     VARCHAR(50),
    first_name          TEXT NOT NULL,
    last_name           TEXT NOT NULL,
    email               TEXT,
    employment_type     TEXT NOT NULL,
    fte_target          NUMERIC(3,2),
    home_facility_id    UUID,
    home_department_id  UUID,
    hourly_rate         NUMERIC(10,2),
    is_active           BOOLEAN NOT NULL DEFAULT true,

    -- Bitemporal columns
    valid_from          TIMESTAMPTZ NOT NULL,       -- when this fact became true in reality
    valid_to            TIMESTAMPTZ NOT NULL DEFAULT 'infinity',  -- when this fact ceased to be true
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),       -- when the DB recorded this version
    superseded_at       TIMESTAMPTZ NOT NULL DEFAULT 'infinity',  -- when this DB record was replaced
    recorded_by         UUID,

    PRIMARY KEY (id, valid_from, recorded_at)
);

-- Current staff view (convenience)
CREATE VIEW current_staff AS
SELECT * FROM staff_members
WHERE valid_to = 'infinity' AND superseded_at = 'infinity';

-- Staff as-of a specific point in time
-- Query: SELECT * FROM staff_members
--        WHERE valid_from <= '2026-03-15' AND valid_to > '2026-03-15'
--        AND recorded_at <= now() AND superseded_at > now();
```

```sql
-- Bitemporal credentials
CREATE TABLE staff_credentials (
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    staff_id            UUID NOT NULL,
    credential_type     TEXT NOT NULL,
    credential_number   VARCHAR(100),
    issuing_state       CHAR(2),
    issuing_authority   TEXT,

    -- Credential-specific valid time (license validity period)
    valid_from          TIMESTAMPTZ NOT NULL,
    valid_to            TIMESTAMPTZ NOT NULL DEFAULT 'infinity',

    -- Database recording time (when we learned about this credential)
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at       TIMESTAMPTZ NOT NULL DEFAULT 'infinity',

    status              TEXT NOT NULL,
    psv_completed       BOOLEAN NOT NULL DEFAULT false,
    psv_date            TIMESTAMPTZ,
    verification_data   JSONB NOT NULL DEFAULT '{}',
    recorded_by         UUID,

    PRIMARY KEY (id, recorded_at)
);

-- Retroactive credential invalidation example:
-- On June 1 we discover that nurse RN-042's license actually expired on May 15.
-- We insert a new record with:
--   valid_to = '2026-05-15', recorded_at = '2026-06-01'
-- And update the old record with:
--   superseded_at = '2026-06-01'
-- This lets us query: "which shifts between May 15-31 were staffed by
-- a nurse whose credential we now know was expired?"
```

```sql
-- Bitemporal facility privileges
CREATE TABLE facility_privileges (
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    staff_id            UUID NOT NULL,
    facility_id         UUID NOT NULL,
    privilege_type      TEXT NOT NULL,
    service_lines       TEXT[] NOT NULL DEFAULT '{}',
    supervision_level   TEXT,

    valid_from          TIMESTAMPTZ NOT NULL,
    valid_to            TIMESTAMPTZ NOT NULL DEFAULT 'infinity',
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at       TIMESTAMPTZ NOT NULL DEFAULT 'infinity',

    status              TEXT NOT NULL,
    conditions          JSONB NOT NULL DEFAULT '{}',
    recorded_by         UUID,

    PRIMARY KEY (id, recorded_at)
);
```

### Solver-Native Constraint Model

Constraints are stored in a format that maps directly to solver variables and constraint definitions, minimizing the transformation needed to load them into the solver.

```sql
-- Constraint definitions in solver-compatible format
CREATE TABLE solver_constraints (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    constraint_set_id   UUID NOT NULL,  -- group constraints by schedule generation run
    name                TEXT NOT NULL,
    constraint_class    TEXT NOT NULL CHECK (constraint_class IN (
        'coverage',           -- minimum/maximum staff per shift slot
        'assignment',         -- staff-to-slot assignment rules
        'sequence',           -- consecutive shift patterns
        'counting',           -- limits on totals (overtime, weekends)
        'fairness',           -- equitable distribution objectives
        'preference',         -- staff preference soft constraints
        'credential',         -- credential-based eligibility
        'cost'                -- cost optimization objectives
    )),
    is_hard             BOOLEAN NOT NULL DEFAULT true,
    weight              INTEGER DEFAULT 0,  -- for soft constraints: higher = more important
    scope_facility_id   UUID,
    scope_department_id UUID,
    scope_roles         TEXT[],             -- which roles this applies to
    -- Solver-native parameter representation
    parameters          JSONB NOT NULL
);

-- parameters JSONB examples by constraint_class:

-- coverage:
-- {
--   "type": "min_staff_per_slot",
--   "shift_template_id": "tmpl-day-12",
--   "role": "rn",
--   "min_count": 4,
--   "max_count": 6,
--   "acuity_adjusted": true,
--   "acuity_formula": "base * (1 + 0.25 * (acuity - 3.0))"
-- }

-- sequence:
-- {
--   "type": "max_consecutive_shifts",
--   "max_value": 3,
--   "shift_types": ["night_12"],
--   "applies_to_roles": ["rn", "lpn"],
--   "lookback_days": 7
-- }

-- sequence:
-- {
--   "type": "min_rest_between_shifts",
--   "min_hours": 11,
--   "penalty_per_hour_violation": 100
-- }

-- counting:
-- {
--   "type": "max_hours_per_period",
--   "period_days": 7,
--   "max_hours": 40,
--   "overtime_threshold": 40,
--   "double_time_threshold": 60
-- }

-- fairness:
-- {
--   "type": "equitable_distribution",
--   "distribute": "weekend_shifts",
--   "method": "minimize_variance",
--   "across": "all_staff_in_department",
--   "period_weeks": 4
-- }

-- preference:
-- {
--   "type": "staff_shift_preference",
--   "staff_id": "staff-042",
--   "preferred_shifts": ["day_12"],
--   "avoided_shifts": ["night_12"],
--   "weight": 50
-- }
```

```sql
-- Constraint sets group constraints for a specific solver run
CREATE TABLE constraint_sets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id       UUID NOT NULL,
    schedule_start      DATE NOT NULL,
    schedule_end        DATE NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Snapshot of applicable rules at creation time
    rule_sources        JSONB NOT NULL,  -- references to labor laws, union contracts, policies
    status              TEXT NOT NULL CHECK (status IN ('building', 'ready', 'solving', 'solved', 'failed'))
);
```

### Eligibility Graph Model

The eligibility relationship between staff and shift slots is pre-computed as a weighted graph, updated in real-time as credentials change.

```sql
-- Shift slots: the "demand side" of the graph
CREATE TABLE shift_slots (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    department_id       UUID NOT NULL,
    shift_template_id   UUID NOT NULL,
    shift_date          DATE NOT NULL,
    role_required       TEXT NOT NULL,
    credentials_required TEXT[] NOT NULL DEFAULT '{}',
    skills_required     TEXT[] NOT NULL DEFAULT '{}',
    skills_preferred    TEXT[] NOT NULL DEFAULT '{}',
    is_filled           BOOLEAN NOT NULL DEFAULT false,
    assigned_staff_id   UUID,
    UNIQUE (department_id, shift_template_id, shift_date, role_required)
);

-- Pre-computed eligibility edges (staff ↔ slot)
CREATE TABLE eligibility_edges (
    staff_id            UUID NOT NULL,
    slot_id             UUID NOT NULL REFERENCES shift_slots(id),
    -- Eligibility factors (each is a component of the edge weight)
    credential_match    BOOLEAN NOT NULL,       -- hard constraint: are credentials valid?
    privilege_match     BOOLEAN NOT NULL,       -- hard constraint: is staff privileged at facility?
    availability_match  BOOLEAN NOT NULL,       -- hard constraint: is staff available?
    overtime_ok         BOOLEAN NOT NULL,       -- hard constraint: would this cause overtime violation?
    rest_period_ok      BOOLEAN NOT NULL,       -- hard constraint: sufficient rest since last shift?
    -- Soft constraint scores (higher = better fit)
    cost_score          NUMERIC(5,2) NOT NULL DEFAULT 0,  -- lower cost tier = higher score
    preference_score    NUMERIC(5,2) NOT NULL DEFAULT 0,  -- staff preference alignment
    fairness_score      NUMERIC(5,2) NOT NULL DEFAULT 0,  -- how this assignment affects fairness
    fatigue_score       NUMERIC(5,2) NOT NULL DEFAULT 0,  -- fatigue risk (higher = less fatigued)
    experience_score    NUMERIC(5,2) NOT NULL DEFAULT 0,  -- experience level match
    -- Composite eligibility
    is_eligible         BOOLEAN GENERATED ALWAYS AS (
        credential_match AND privilege_match AND availability_match
        AND overtime_ok AND rest_period_ok
    ) STORED,
    composite_score     NUMERIC(6,2) GENERATED ALWAYS AS (
        cost_score + preference_score + fairness_score + fatigue_score + experience_score
    ) STORED,
    computed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (staff_id, slot_id)
);

CREATE INDEX idx_eligibility_eligible ON eligibility_edges (slot_id)
    WHERE is_eligible = true;
CREATE INDEX idx_eligibility_score ON eligibility_edges (slot_id, composite_score DESC)
    WHERE is_eligible = true;
```

### Solver Results and Schedule Output

```sql
CREATE TABLE solver_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    constraint_set_id   UUID NOT NULL REFERENCES constraint_sets(id),
    department_id       UUID NOT NULL,
    schedule_start      DATE NOT NULL,
    schedule_end        DATE NOT NULL,
    solver_engine       TEXT NOT NULL,  -- 'or_tools_cp_sat', 'gurobi', 'optaplanner'
    started_at          TIMESTAMPTZ NOT NULL,
    completed_at        TIMESTAMPTZ,
    status              TEXT NOT NULL CHECK (status IN ('running', 'optimal', 'feasible', 'infeasible', 'timeout')),
    -- Solver performance metrics
    solve_time_ms       BIGINT,
    variables_count     INTEGER,
    constraints_count   INTEGER,
    objective_value     NUMERIC(10,4),
    optimality_gap      NUMERIC(6,4),  -- gap from proven optimal (0.0 = optimal)
    -- Detailed results
    solution_quality    JSONB NOT NULL DEFAULT '{}',
    infeasibility_report JSONB,  -- if infeasible, which constraints conflict
    fairness_metrics    JSONB NOT NULL DEFAULT '{}'
);

-- solution_quality JSONB example:
-- {
--   "total_shifts_assigned": 308,
--   "total_shifts_required": 312,
--   "fill_rate": 0.987,
--   "overtime_shifts": 12,
--   "agency_shifts": 4,
--   "float_shifts": 8,
--   "soft_violations": [
--     {"constraint": "weekend_distribution", "staff_id": "staff-042", "severity": "minor"},
--   ],
--   "cost_estimate": {
--     "regular_labor": 145000.00,
--     "overtime_labor": 12500.00,
--     "agency_labor": 8200.00,
--     "total": 165700.00
--   }
-- }

-- fairness_metrics JSONB example:
-- {
--   "weekend_shifts": {
--     "gini_coefficient": 0.08,
--     "max_deviation_from_mean": 1,
--     "staff_with_max": ["staff-042"],
--     "staff_with_min": ["staff-017"]
--   },
--   "night_shifts": {
--     "gini_coefficient": 0.12,
--     "max_deviation_from_mean": 2
--   },
--   "holiday_balance": {
--     "all_staff_within_one_holiday": true
--   }
-- }

CREATE TABLE shift_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    solver_run_id       UUID REFERENCES solver_runs(id),
    slot_id             UUID NOT NULL REFERENCES shift_slots(id),
    staff_id            UUID NOT NULL,
    shift_date          DATE NOT NULL,
    assignment_type     TEXT NOT NULL,
    cost_tier           TEXT,
    status              TEXT NOT NULL DEFAULT 'scheduled',
    -- Solver decision metadata
    solver_score        NUMERIC(6,2),          -- why the solver chose this assignment
    alternative_count   INTEGER,               -- how many eligible staff existed
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Real-Time Staffing State

```sql
-- Live staffing state, updated by events (callouts, clock-ins, census changes)
CREATE TABLE staffing_state (
    department_id       UUID NOT NULL,
    shift_date          DATE NOT NULL,
    shift_template_id   UUID NOT NULL,
    role                TEXT NOT NULL,
    -- Current counts
    required_count      INTEGER NOT NULL,
    scheduled_count     INTEGER NOT NULL,
    confirmed_count     INTEGER NOT NULL DEFAULT 0,
    clocked_in_count    INTEGER NOT NULL DEFAULT 0,
    callout_count       INTEGER NOT NULL DEFAULT 0,
    -- Census-driven adjustments
    current_census      INTEGER,
    current_acuity      NUMERIC(3,1),
    acuity_adjusted_requirement INTEGER,
    -- Status
    coverage_status     TEXT GENERATED ALWAYS AS (
        CASE
            WHEN clocked_in_count >= required_count THEN 'adequate'
            WHEN scheduled_count >= required_count THEN 'scheduled_adequate'
            WHEN scheduled_count > 0 THEN 'understaffed'
            ELSE 'critical'
        END
    ) STORED,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (department_id, shift_date, shift_template_id, role)
);
```

---

## Pros

- **Solver-first design.** The data model is structured to minimize transformation between database storage and solver input. Constraint definitions map directly to CP-SAT variables and constraints. The eligibility graph is the solver's search space.
- **Bitemporal correctness.** Healthcare credentialing requires answering: "Was this nurse validly credentialed on the night of the incident?" Bitemporal modeling answers this definitively, supporting both regulatory audits and malpractice defense.
- **Pre-computed eligibility.** The eligibility graph pre-computes whether each staff member can work each slot, decomposing the N x M eligibility problem into a cached, incrementally updated structure. Solver startup time drops dramatically.
- **Retroactive analysis.** When a credential is discovered to be invalid, the bitemporal model identifies every shift that was worked under the invalid credential without modifying historical records.
- **Rich solver metadata.** Every generated schedule carries its optimization metrics, fairness scores, and constraint violation details. This enables "why was I scheduled this way?" explanations and scheduler decision support.
- **Real-time staffing state.** The staffing_state table provides an always-current view of coverage adequacy, enabling real-time alerting and dynamic open-shift generation.
- **Scalable constraint management.** New constraint types (union rules, state regulations, fatigue models) are added as JSON parameter objects within the established constraint_class taxonomy, without schema changes.

## Cons

- **High implementation complexity.** Bitemporal data models are notoriously difficult to implement correctly. Queries must always specify both time dimensions. ORMs have limited bitemporal support.
- **Eligibility graph maintenance.** The eligibility_edges table must be recomputed whenever credentials change, availability changes, or schedule state changes. For a health system with 5,000 staff and 2,000 daily slots, this is a 10M-row table that needs frequent partial updates.
- **Solver expertise required.** The constraint model assumes deep familiarity with CP-SAT or mixed-integer programming. The team must include operations research expertise to define, tune, and maintain solver constraints.
- **Bitemporal query complexity.** Every query that touches staff or credentials must include temporal predicates. Missing a temporal filter returns incorrect results silently.
- **Storage overhead.** Bitemporal tables store multiple versions of every record. Credential records that change status frequently generate significant row volume.
- **Operational complexity.** The architecture involves PostgreSQL, a solver service, an eligibility graph cache (Redis or in-memory), and real-time state management -- more moving parts than a monolithic database.
- **Debugging difficulty.** Solver infeasibility ("no valid schedule exists") requires understanding which constraints conflict. Infeasibility analysis is computationally expensive and produces technical output that requires OR expertise to interpret.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Primary Database** | PostgreSQL 16+ with temporal table extensions or application-managed bitemporality |
| **Constraint Solver** | Google OR-Tools CP-SAT (open source, Python/C++) for primary solving; Gurobi for large instances requiring commercial-grade MIP |
| **Eligibility Cache** | Redis with sorted sets for scored eligibility lookups; or in-memory graph in solver service |
| **Bitemporal Library** | Custom implementation with PostgreSQL range types (tstzrange), or sql_saga/temporal_tables extensions |
| **Real-time State** | PostgreSQL LISTEN/NOTIFY for staffing_state changes; Redis pub/sub for UI push |
| **Solver API** | gRPC service exposing SolveSchedule(ConstraintSet) -> SolverResult, with streaming progress updates |
| **Monitoring** | Solver performance metrics (solve time, optimality gap, constraint violation counts) as first-class SLIs |

---

## Migration and Scaling Considerations

- **Phased rollout.** Phase 1: relational schema with basic constraints (no bitemporality). Phase 2: add bitemporal tracking to credentials and privileges. Phase 3: build eligibility graph and solver integration. Phase 4: add real-time staffing state.
- **Bitemporal migration.** Existing credential records are imported with `valid_from = issue_date`, `valid_to = expiration_date`, `recorded_at = import_time`. Historical accuracy depends on source system quality.
- **Eligibility graph partitioning.** Partition eligibility_edges by department and date range. Only upcoming schedule periods need active eligibility computation. Historical eligibility can be archived.
- **Solver scaling.** Schedule generation is CPU-intensive but embarrassingly parallel by department. Each department's schedule can be solved independently, with cross-department constraints (float pool) resolved in a second coordination pass.
- **Multi-tenancy.** Each health system's solver constraints, eligibility graph, and bitemporal data are fully isolated. Solver service instances can be dedicated per tenant for large systems.
- **Fallback strategy.** If the solver times out or returns infeasible, the system falls back to: (1) relaxing soft constraints, (2) identifying the minimum constraint set that must be relaxed for feasibility, (3) presenting the infeasibility report to the scheduler for manual resolution.
- **Data retention.** Bitemporal data supports natural archival: old transaction-time versions (superseded_at < retention_date) can be moved to cold storage while preserving the ability to answer "as-of" queries for the retained valid-time range.
