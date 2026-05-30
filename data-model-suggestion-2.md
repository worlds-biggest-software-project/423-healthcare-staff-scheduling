# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Healthcare Staff Scheduling (423)
> Approach: Event Sourcing with Command Query Responsibility Segregation
> Generated: 2026-05-25

---

## Summary

An event-sourced architecture where every state change in the scheduling system is captured as an immutable event in an append-only log. The system separates write operations (commands that produce events) from read operations (queries served by purpose-built projections). This approach provides a complete audit trail by construction, enables temporal queries ("what was the schedule at 3 PM last Tuesday?"), and supports complex workflows like shift swaps and credential verifications where the history of decisions matters as much as the current state.

Healthcare scheduling is an especially strong fit for event sourcing because regulatory compliance demands full traceability of who changed what, when, and why; because schedules go through multi-stage workflows (draft, review, publish, modify); and because credential status changes have retroactive implications that require reconstructing past states.

---

## Core Architecture

```
                    ┌─────────────┐
                    │  Commands   │
                    │  (API/UI)   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Command    │
                    │  Handlers   │
                    │  (Domain)   │
                    └──────┬──────┘
                           │ emit events
                    ┌──────▼──────┐
                    │  Event      │
                    │  Store      │
                    │  (append)   │
                    └──────┬──────┘
                           │ subscribe
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐ ┌──▼────┐ ┌─────▼─────┐
       │ Schedule     │ │Staff  │ │ Analytics │
       │ Read Model   │ │Read   │ │ Read      │
       │ (PostgreSQL) │ │Model  │ │ Model     │
       └─────────────┘ └───────┘ └───────────┘
              │            │            │
       ┌──────▼──────┐ ┌──▼────┐ ┌─────▼─────┐
       │  Query API  │ │Query  │ │ Dashboard │
       │  (shifts,   │ │API    │ │ API       │
       │  calendar)  │ │       │ │           │
       └─────────────┘ └───────┘ └───────────┘
```

---

## Key Entities as Event Streams

### Event Store Schema

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  TEXT NOT NULL,       -- 'Schedule', 'StaffMember', 'Credential', 'ShiftAssignment', 'OpenShift'
    aggregate_id    UUID NOT NULL,
    event_type      TEXT NOT NULL,       -- 'ShiftAssigned', 'CredentialVerified', 'SwapRequested', etc.
    event_data      JSONB NOT NULL,      -- full event payload
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, user_id, ip_address
    version         BIGINT NOT NULL,     -- optimistic concurrency per aggregate
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_type, aggregate_id, version)
);

-- Index for stream replay
CREATE INDEX idx_events_aggregate ON events (aggregate_type, aggregate_id, version);
-- Index for global ordering
CREATE INDEX idx_events_recorded ON events (recorded_at);
-- Index for event type filtering (projections)
CREATE INDEX idx_events_type ON events (event_type);
```

### Event Types by Domain

#### Schedule Lifecycle Events

```json
// ScheduleCreated
{
  "event_type": "ScheduleCreated",
  "aggregate_type": "Schedule",
  "aggregate_id": "sched-001",
  "event_data": {
    "department_id": "dept-icu-01",
    "start_date": "2026-06-01",
    "end_date": "2026-06-14",
    "created_by": "user-mgr-001"
  }
}

// ShiftAssigned
{
  "event_type": "ShiftAssigned",
  "aggregate_type": "Schedule",
  "aggregate_id": "sched-001",
  "event_data": {
    "assignment_id": "asgn-001",
    "staff_id": "staff-rn-042",
    "shift_template_id": "tmpl-day-12",
    "shift_date": "2026-06-03",
    "assignment_type": "regular",
    "credential_check_passed": true,
    "credentials_verified": ["rn_license_ca", "bls_cert", "acls_cert"]
  }
}

// SchedulePublished
{
  "event_type": "SchedulePublished",
  "aggregate_type": "Schedule",
  "aggregate_id": "sched-001",
  "event_data": {
    "published_by": "user-mgr-001",
    "staff_notified_count": 47,
    "total_shifts": 312,
    "unfilled_shifts": 4
  }
}

// ShiftSwapRequested, ShiftSwapApproved, ShiftSwapDenied
// ShiftCancelled, ShiftReassigned
// ScheduleLocked, ScheduleArchived
```

#### Credential Lifecycle Events

```json
// CredentialSubmitted
{
  "event_type": "CredentialSubmitted",
  "aggregate_type": "Credential",
  "aggregate_id": "cred-001",
  "event_data": {
    "staff_id": "staff-rn-042",
    "credential_type": "rn_license",
    "issuing_state": "CA",
    "credential_number": "RN-123456",
    "expiration_date": "2027-03-31"
  }
}

// CredentialPSVCompleted  (Primary Source Verification)
// CredentialApproved
// CredentialExpired
// CredentialSuspended
// CredentialRenewed
// PrivilegeGranted
// PrivilegeRevoked
```

#### Staffing Operations Events

```json
// OpenShiftPosted
{
  "event_type": "OpenShiftPosted",
  "aggregate_type": "OpenShift",
  "aggregate_id": "open-001",
  "event_data": {
    "department_id": "dept-icu-01",
    "shift_template_id": "tmpl-night-12",
    "shift_date": "2026-06-05",
    "reason": "callout",
    "original_staff_id": "staff-rn-019",
    "required_credentials": ["rn_license_ca", "acls_cert"],
    "cost_tier_max": "float_pool"
  }
}

// ShiftBidPlaced, ShiftBidAccepted, ShiftBidRejected
// CalloutReported
// CensusUpdated
// OvertimeThresholdReached
// FatigueRiskFlagged
```

---

## Read Model Projections

### Current Schedule Projection

```sql
-- Materialized from ShiftAssigned/ShiftCancelled/ShiftReassigned events
CREATE TABLE schedule_view (
    assignment_id       UUID PRIMARY KEY,
    schedule_id         UUID NOT NULL,
    department_id       UUID NOT NULL,
    department_name     TEXT NOT NULL,
    staff_id            UUID NOT NULL,
    staff_name          TEXT NOT NULL,
    role_type           TEXT NOT NULL,
    shift_date          DATE NOT NULL,
    start_time          TIME NOT NULL,
    end_time            TIME NOT NULL,
    assignment_type     TEXT NOT NULL,
    status              TEXT NOT NULL,
    cost_tier           TEXT,
    last_modified_at    TIMESTAMPTZ NOT NULL,
    last_modified_by    UUID
);

CREATE INDEX idx_schedule_view_dept_date ON schedule_view (department_id, shift_date);
CREATE INDEX idx_schedule_view_staff ON schedule_view (staff_id, shift_date);
```

### Staff Credential Status Projection

```sql
-- Materialized from credential lifecycle events
CREATE TABLE credential_status_view (
    staff_id            UUID NOT NULL,
    staff_name          TEXT NOT NULL,
    credential_type     TEXT NOT NULL,
    credential_number   TEXT,
    status              TEXT NOT NULL,
    expiration_date     DATE,
    days_until_expiry   INTEGER,
    psv_status          TEXT,
    last_verified_at    TIMESTAMPTZ,
    PRIMARY KEY (staff_id, credential_type)
);
```

### Overtime and Compliance Projection

```sql
-- Materialized from ShiftAssigned/TimeEntryRecorded events
CREATE TABLE compliance_view (
    staff_id            UUID NOT NULL,
    week_start          DATE NOT NULL,
    total_scheduled_hours NUMERIC(5,2),
    total_worked_hours  NUMERIC(5,2),
    overtime_hours      NUMERIC(5,2),
    consecutive_shifts  INTEGER,
    hours_since_last_rest NUMERIC(5,2),
    fatigue_risk_score  NUMERIC(3,2),
    compliance_violations TEXT[],
    PRIMARY KEY (staff_id, week_start)
);
```

### Staffing Analytics Projection

```sql
-- Materialized from various events for dashboard consumption
CREATE TABLE staffing_analytics_view (
    department_id       UUID NOT NULL,
    shift_date          DATE NOT NULL,
    shift_type          TEXT NOT NULL,
    required_staff      INTEGER,
    scheduled_staff     INTEGER,
    actual_staff        INTEGER,
    callout_count       INTEGER,
    agency_fills        INTEGER,
    overtime_fills      INTEGER,
    census_at_shift     INTEGER,
    acuity_at_shift     NUMERIC(3,1),
    cost_total          NUMERIC(10,2),
    PRIMARY KEY (department_id, shift_date, shift_type)
);
```

---

## Command Handlers (Domain Logic)

```python
# Pseudocode for key command handlers

class AssignShiftHandler:
    def handle(self, cmd: AssignShiftCommand) -> list[Event]:
        # 1. Load schedule aggregate from event stream
        schedule = self.repo.load(cmd.schedule_id)

        # 2. Validate business rules
        staff = self.staff_repo.load(cmd.staff_id)
        self.credential_checker.verify(staff, cmd.shift_template, cmd.shift_date)
        self.constraint_engine.check_fatigue(staff, cmd.shift_date)
        self.constraint_engine.check_overtime(staff, cmd.shift_date)
        self.constraint_engine.check_union_rules(staff, cmd.shift_date, cmd.shift_template)

        # 3. Emit event (or raise error)
        return [ShiftAssignedEvent(
            schedule_id=cmd.schedule_id,
            staff_id=cmd.staff_id,
            shift_template_id=cmd.shift_template_id,
            shift_date=cmd.shift_date,
            credential_check_passed=True,
            credentials_verified=staff.active_credential_codes()
        )]


class RequestShiftSwapHandler:
    def handle(self, cmd: RequestSwapCommand) -> list[Event]:
        # Multi-step workflow: request -> peer accept -> manager approve
        # Each step emits a separate event
        assignment = self.repo.load(cmd.assignment_id)
        self.constraint_engine.validate_swap_eligibility(
            cmd.requester_id, cmd.target_staff_id, assignment
        )
        return [ShiftSwapRequestedEvent(...)]
```

---

## Pros

- **Complete audit trail by construction.** Every change to every schedule, credential, and assignment is permanently recorded. Regulatory compliance (HIPAA, Joint Commission, state labor board) is built into the architecture rather than bolted on.
- **Temporal query support.** "What was the ICU schedule at 2 AM on March 15?" is answered by replaying events up to that timestamp. Critical for incident investigations and malpractice defense.
- **Workflow modeling.** Multi-step processes (credential verification, shift swap approval, schedule draft-review-publish) map naturally to event sequences. Each step is traceable.
- **Independent read model scaling.** The schedule calendar view, credential dashboard, and analytics dashboard each have their own optimized projection, scaled independently. Mobile schedule views don't compete with analytics queries.
- **Retroactive correction.** When a credential is discovered to have been invalid, compensating events can be emitted and all affected schedule assignments can be identified and flagged.
- **Event-driven integration.** EHR census updates, credentialing service responses, and payroll exports naturally integrate as events, matching the append-only model.
- **Debugging and support.** When a scheduling dispute arises, the complete event history shows exactly what happened, who did it, and what the system state was at every point.

## Cons

- **Increased complexity.** Event sourcing requires maintaining event schemas, building and maintaining projections, handling eventual consistency between write and read sides, and managing event versioning over time.
- **Eventual consistency.** After a command is processed, read models update asynchronously. A manager publishing a schedule may not see it reflected in the calendar view for milliseconds to seconds. This can confuse users if not handled with UI patterns (optimistic updates, "publishing..." indicators).
- **Event schema evolution.** As the domain evolves (new constraint types, new credential categories), event schemas must be versioned. Old events must remain readable. This requires careful schema evolution strategies (upcasting, versioned deserializers).
- **Projection rebuild cost.** If a read model projection has a bug, it must be rebuilt from the event log. For a large health system with years of events, this can take significant time.
- **Team learning curve.** Event sourcing and CQRS are less familiar to most developers than CRUD. Hiring and onboarding takes longer.
- **Testing complexity.** Testing requires asserting on emitted events rather than database state. Integration tests must verify that projections correctly consume events.
- **Operational overhead.** Running an event store, projection services, and multiple read databases requires more infrastructure than a single PostgreSQL instance.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event Store** | EventStoreDB (purpose-built), or PostgreSQL with append-only events table + outbox pattern |
| **Command/Query Framework** | Axon Framework (Java/Kotlin), Marten (C#/.NET), or custom with Node.js/Python |
| **Read Model Databases** | PostgreSQL for relational projections; Redis for real-time shift availability; Elasticsearch for staff/credential search |
| **Event Bus** | Apache Kafka or Amazon EventBridge for event distribution to projections and external systems |
| **Projection Hosting** | Stateless workers subscribing to event streams, one per read model |
| **API Layer** | Separate command API (validates and emits events) and query API (reads from projections) |
| **Monitoring** | Track projection lag (time between event occurrence and projection update) as a key SLA metric |

---

## Migration and Scaling Considerations

- **Start hybrid.** Begin with PostgreSQL as both event store and read model database. The events table is the source of truth; read models are materialized views or separate tables updated by triggers or application-level subscribers. Move to dedicated infrastructure (EventStoreDB, Kafka) as scale demands.
- **Event partitioning.** Partition the events table by aggregate_type or by month. Schedule events dominate volume; credential events are lower volume but higher criticality.
- **Snapshotting.** For aggregates with long event histories (a department's schedule over years), store periodic snapshots to avoid replaying thousands of events on every command.
- **Event versioning strategy.** Use a schema registry (e.g., Confluent Schema Registry with Avro/JSON Schema) to manage event schema evolution. Support at least two versions simultaneously during migration windows.
- **Multi-tenancy.** Add `tenant_id` to every event. Read model projections can be partitioned or filtered by tenant. Each health system's event stream is logically isolated.
- **Disaster recovery.** The event log is the system of record. Read models are disposable and rebuildable. Backup strategy focuses on protecting the event store with point-in-time recovery.
- **Gradual adoption.** Event sourcing can be adopted incrementally: start with the scheduling domain (highest audit value), then extend to credentialing, then staffing operations. Other domains can remain CRUD until the team gains confidence.
