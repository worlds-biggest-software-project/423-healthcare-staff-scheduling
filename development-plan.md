# Healthcare Staff Scheduling — Phased Development Plan

> Project: 423-healthcare-staff-scheduling · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The MVP feature set (constraint-based scheduling, credential-to-shift matching, multi-facility privilege management, compliance tracking, time & attendance, staff self-service, EHR/payroll integration) drives the early phases; AI-native differentiators (demand forecasting, burnout risk, fatigue-optimised scheduling, call-out prediction) drive the later phases.

The product is the only **open-source** entrant in a market of proprietary incumbents (QGenda, symplr, NurseGrid, Petal). It must deploy both **cloud-hosted (multi-tenant SaaS)** and **self-hosted (single-tenant, HIPAA data residency)**.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend + solver glue) | **Python 3.12** | The scheduling core is an NP-hard CP problem solved with Google OR-Tools, whose first-class binding is Python. The AI/ML phases (demand forecasting, burnout prediction) need the Python data/ML ecosystem (pandas, scikit-learn, prophet). Keeping solver, API, and ML in one language removes a serialization boundary on the hottest path. |
| API framework | **FastAPI** | Native Pydantic models give typed request/response validation and auto-generate the **OpenAPI 3.1** spec that `standards.md` requires for EHR/payroll/HRIS integration. ASGI supports the streaming and websocket needs of the real-time open-shift marketplace. |
| Database | **PostgreSQL 16** | Adopts **data-model-suggestion-3 (Hybrid Relational + JSONB)**. Relational integrity for staff/credentials/assignments; JSONB for the wildly variable union-contract / state-labor-law / facility-policy rule sets and ML outputs, avoiding the EAV anti-pattern and per-rule schema migrations called out in suggestion-1's cons. Single engine = one backup/RLS/monitoring story for self-hosted operators. |
| Constraint solver | **Google OR-Tools CP-SAT** | Industry-standard open-source CP solver. Consumes the `scheduling_rule_sets.rules` JSONB directly (suggestion-3 §Solver integration). Handles hard constraints (rest, max-hours, credential match) and weighted soft constraints (fairness, preferences) in one model. |
| ORM / query layer | **SQLAlchemy 2.0 (async) + Alembic** | Mature async ORM with explicit JSONB column support and versioned migrations. Each feature area ships as its own Alembic revision (additive, per the phase principle). |
| Schema validation for JSONB | **`jsonschema` (Draft 2020-12)** | `standards.md` mandates JSON Schema for shift/constraint/competency structures. PostgreSQL cannot enforce JSONB shape; we validate at the API boundary and store versioned schemas in `schemas/`. |
| Task queue / async workers | **Celery + Redis** | Long-running solver runs, EHR census polling, credential PSV lookups, and notification fan-out must run off the request thread. Redis doubles as the broker and the real-time open-shift cache. |
| Real-time push | **WebSocket (FastAPI) + Redis pub/sub** | Sub-minute notification of eligible staff on call-outs (research §Real-time responsiveness) needs push, not poll. |
| Frontend | **Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui** | Two surfaces: a manager dashboard (schedule grid, coverage heatmap, compliance alerts) and a mobile-first staff PWA (schedule view, availability, swaps, open-shift pickup). SSR for fast first paint on mobile; PWA installability for nurses. |
| Auth | **OAuth 2.0 / OIDC + SAML 2.0** | `standards.md`: enterprise health systems require SAML SSO (Entra ID, Okta). OAuth2 password/PKCE for the mobile PWA. RBAC + PostgreSQL Row-Level Security for multi-tenant isolation. |
| Integration adapters | **HL7 FHIR R4 (`fhir.resources`), X12 837/834 (`pyx12`), REST connectors** | FHIR `PractitionerRole` for credential/competency sync; X12 for payroll export; pluggable connector interface for Epic/Cerner/Athena census feeds and HRIS. |
| Containerisation | **Docker + docker-compose; Helm chart (later)** | Self-hosted operators get a single `docker-compose up`; SaaS runs the same images on Kubernetes via the Helm chart. |
| Testing | **pytest + pytest-asyncio + Testcontainers (Postgres/Redis); Playwright (frontend E2E)** | Real Postgres in CI via Testcontainers catches JSONB/constraint behaviour that SQLite mocks would hide. |
| Code quality | **ruff (lint+format), mypy (strict), pre-commit** | Strict typing protects the typed-Pydantic/JSONB boundary. |
| Package manager | **uv** (Python), **pnpm** (frontend) | Fast, reproducible lockfiles. |
| Observability | **OpenTelemetry → Prometheus/Grafana; structured JSON logs** | Solver latency, constraint-violation counts, and integration error rates are first-class operational metrics. |
| Audit logging | **Append-only `audit_log` table (partitioned) + DB triggers** | HIPAA §164.312(b) audit controls and Joint Commission traceability. |

### Project Structure

```
healthcare-staff-scheduling/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── README.md
├── .pre-commit-config.yaml
├── schemas/                          # JSON Schema (Draft 2020-12) for all JSONB structures
│   ├── scheduling_rules.schema.json
│   ├── staffing_requirements.schema.json
│   ├── eligibility_criteria.schema.json
│   ├── credential_verification.schema.json
│   └── ml_prediction.schema.json
├── backend/
│   ├── app/
│   │   ├── main.py                   # FastAPI app factory, router mount, middleware
│   │   ├── config.py                 # Pydantic Settings (env-driven)
│   │   ├── deps.py                   # DI: db session, current_user, tenant context
│   │   ├── db/
│   │   │   ├── base.py               # SQLAlchemy async engine/session
│   │   │   ├── models/               # ORM models grouped by domain
│   │   │   │   ├── org.py            # organizations, facilities, departments
│   │   │   │   ├── staff.py          # staff_members, availability, time_off
│   │   │   │   ├── credential.py     # staff_credentials, facility_privileges
│   │   │   │   ├── scheduling.py     # shift_templates, schedule_periods, shift_assignments
│   │   │   │   ├── marketplace.py    # open_shifts, shift_bids, swaps
│   │   │   │   ├── rules.py          # scheduling_rule_sets
│   │   │   │   ├── attendance.py     # time_entries, overtime_tracking
│   │   │   │   ├── census.py         # census_snapshots
│   │   │   │   ├── ml.py             # ml_predictions
│   │   │   │   └── audit.py          # audit_log
│   │   │   └── rls.py                # Row-Level Security policy helpers
│   │   ├── schemas/                  # Pydantic request/response DTOs (mirror by domain)
│   │   ├── api/v1/                   # FastAPI routers by domain
│   │   ├── services/                 # business logic
│   │   │   ├── credentialing.py
│   │   │   ├── eligibility.py        # credential-to-shift matching
│   │   │   ├── compliance.py         # labor-law / fatigue rule evaluation
│   │   │   ├── marketplace.py
│   │   │   └── attendance.py
│   │   ├── solver/
│   │   │   ├── model.py              # OR-Tools CP-SAT model builder
│   │   │   ├── constraints.py        # rule JSONB -> CP-SAT constraints
│   │   │   ├── objective.py          # soft-constraint weights, fairness terms
│   │   │   └── runner.py             # Celery task wrapping a solve
│   │   ├── rules/
│   │   │   ├── registry.py           # constraint-type registry
│   │   │   └── validators.py         # JSON Schema validation
│   │   ├── integrations/
│   │   │   ├── base.py               # Connector ABC
│   │   │   ├── fhir.py               # FHIR PractitionerRole, census
│   │   │   ├── ehr/                  # epic.py, cerner.py, athena.py
│   │   │   ├── hris.py
│   │   │   └── payroll.py            # X12 837/834 export
│   │   ├── ml/
│   │   │   ├── demand_forecast.py
│   │   │   ├── burnout.py
│   │   │   ├── callout.py
│   │   │   └── fatigue.py
│   │   ├── realtime/
│   │   │   └── ws.py                 # websocket hub + Redis pub/sub
│   │   ├── auth/                     # oidc.py, saml.py, rbac.py
│   │   └── workers/                  # celery_app.py, tasks
│   ├── alembic/versions/             # one revision per feature area
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/                 # sample staff, credentials, rule sets, census
├── frontend/
│   ├── package.json
│   ├── app/                          # Next.js App Router
│   │   ├── (manager)/                # dashboard, schedule grid, compliance
│   │   └── (staff)/                  # PWA: my-schedule, availability, swaps, open-shifts
│   ├── components/
│   ├── lib/api/                      # generated OpenAPI client
│   └── tests/                        # Playwright E2E
├── helm/                             # Kubernetes chart (SaaS)
└── docs/
    ├── api.md
    └── deployment.md
```

The structure is grouped by concern, not by phase; every phase adds files/migrations without restructuring.

---

## Phase 1: Foundation & Multi-Tenant Data Core

### Purpose
Stand up the project skeleton, the PostgreSQL hybrid schema for the organizational and staff domains, multi-tenant isolation, and the FastAPI app with auth and audit logging. After this phase the system can hold organizations, facilities, departments, and staff with role-based access and a complete audit trail — the substrate every later phase writes to.

### Tasks

#### 1.1 — Project scaffolding, config, and containerisation

**What**: A runnable FastAPI app with `docker-compose` bringing up the API, PostgreSQL 16, and Redis.

**Design**:
- `app/config.py` using `pydantic_settings.BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    deployment_mode: Literal["saas", "self_hosted"] = "self_hosted"
    jwt_secret: str
    oidc_issuer: str | None = None
    saml_metadata_url: str | None = None
    log_level: str = "INFO"
    solver_time_limit_seconds: int = 60
    model_config = SettingsConfigDict(env_file=".env")
```
- `app/main.py` exposes `create_app() -> FastAPI`; mounts `/api/v1`, `/health`, `/metrics`; adds CORS, request-ID, and OTel middleware.
- `docker-compose.yml` services: `api`, `worker` (Celery), `db` (postgres:16), `redis`, `frontend`.
- `GET /health` returns `{"status":"ok","db":"ok","redis":"ok"}` (checks connectivity).

**Testing**:
- `Unit: Settings loads from env vars → correct typed values, defaults applied.`
- `Unit: missing DATABASE_URL → ValidationError naming the field.`
- `Integration (Testcontainers): GET /health with live db+redis → 200, all "ok".`
- `Integration: GET /health with db down → 503, db="error".`

#### 1.2 — Hybrid schema: organizations, facilities, departments

**What**: Alembic revision and SQLAlchemy models for the organizational hierarchy with JSONB config columns.

**Design**: Adopt suggestion-3 DDL. `organizations(settings JSONB)`, `facilities(labor_rules JSONB, address JSONB)`, `departments(staffing_config JSONB)`. Example model:
```python
class Facility(Base):
    __tablename__ = "facilities"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    organization_id: Mapped[UUID] = mapped_column(ForeignKey("organizations.id"))
    name: Mapped[str]
    state: Mapped[str] = mapped_column(String(2))
    timezone: Mapped[str]
    is_active: Mapped[bool] = mapped_column(default=True)
    labor_rules: Mapped[dict] = mapped_column(JSONB, default=dict)
```
- `settings`, `labor_rules`, `staffing_config` validated against `schemas/*.schema.json` on write (see 1.4).
- All timestamps `TIMESTAMPTZ`; all dates/times encoded **ISO 8601** per `standards.md`.

**Testing**:
- `Unit: Facility model round-trips labor_rules JSONB unchanged.`
- `Integration: create org → facility → department, FK chain enforced; orphan facility insert → IntegrityError.`
- `Integration: facility with invalid labor_rules JSON (missing required key) → 422 before DB write.`

#### 1.3 — Auth: OIDC + SAML, RBAC, and Row-Level Security

**What**: Authentication via OIDC and SAML 2.0, role-based authorization, and tenant isolation via Postgres RLS.

**Design**:
- Roles: `system_admin`, `org_admin`, `scheduler`, `manager`, `credentialer`, `staff`. Permission matrix in `auth/rbac.py` (resource × action).
- JWT carries `sub`, `org_id`, `facility_ids`, `roles`.
- `deps.py::current_user()` decodes JWT; `deps.py::tenant_session()` sets `SET app.current_org = :org_id` so RLS policies (`app/db/rls.py`) filter every tenant-scoped table by `organization_id`.
- SAML via `python3-saml`; OIDC via `authlib`. Endpoints: `POST /auth/oidc/callback`, `POST /auth/saml/acs`, `POST /auth/token` (password/PKCE for PWA).

**Testing**:
- `Unit: RBAC denies "staff" the schedule.publish action; allows "scheduler".`
- `Integration: JWT with org A cannot read org B facilities (RLS) → empty result, not 403 leak.`
- `Integration (mocked IdP): valid SAML assertion → session issued; tampered assertion → 401.`

#### 1.4 — JSON Schema registry & validation layer

**What**: A validation utility loading `schemas/*.schema.json` and enforcing Draft 2020-12 on JSONB payloads, plus the constraint-type registry.

**Design**:
```python
class SchemaRegistry:
    def validate(self, schema_name: str, payload: dict) -> None: ...
        # raises SchemaValidationError(field_path, message) on failure
```
- Each JSONB object carries `"_v": <int>` for versioned evolution (suggestion-3 §JSONB schema versioning).
- `rules/registry.py` maps constraint `type` strings (`max_consecutive_days`, `min_rest_between_shifts`, `weekend_distribution`, `holiday_rotation`, …) to handler classes used later by the solver.

**Testing**:
- `Unit: valid scheduling_rules payload → no error.`
- `Unit: rule with unknown "type" → SchemaValidationError naming the offending path.`
- `Unit: payload missing "_v" → defaults to 1, logs deprecation warning.`

#### 1.5 — Audit log

**What**: Partitioned append-only `audit_log` with a SQLAlchemy event hook capturing create/update/delete on tenant tables.

**Design**: suggestion-3 `audit_log` DDL, `PARTITION BY RANGE (occurred_at)` monthly. `changes JSONB` stores `{field_changes:{f:{old,new}}, context:{}}`. `ip_address INET`, `user_agent`. No update/delete grants on the table (append-only).

**Testing**:
- `Integration: updating a facility name writes one audit row with old/new values and actor_id.`
- `Integration: attempt UPDATE on audit_log → permission denied.`
- `Unit: monthly partition routing selects correct child table for a given occurred_at.`

---

## Phase 2: Staff, Credentialing & Eligibility Engine

### Purpose
Model the workforce and the credentialing lifecycle, then build the eligibility engine that answers the system's most liability-critical question: *is this clinician credentialed and privileged to work this shift at this facility right now?* This is the core differentiator (symplr/TCP/Shift MedStaff feature) and a hard precondition for any schedule generation.

### Tasks

#### 2.1 — Staff members, roles, skills, availability

**What**: Models and CRUD for staff with JSONB `roles`/`skills`/`preferences`, plus availability and time-off.

**Design**: suggestion-3 `staff_members` (GIN indexes on `roles`, `skills`, `preferences`). Relational `staff_availability`, `time_off_requests` (suggestion-1). DTOs:
```python
class StaffRole(BaseModel):
    role: Literal["rn","lpn","cna","np","pa","md","do","crna","rt","pharmacist","tech","clerk","other"]
    specialty: str | None = None
    is_primary: bool = False
    effective_date: date
```
- Endpoints: `POST/GET/PATCH /api/v1/staff`, `POST /api/v1/staff/{id}/availability`, `POST /api/v1/staff/{id}/time-off`.
- Query helper: `find_staff(role, skills, facility_id)` using `roles @> '[{"role":"rn"}]'` GIN containment.

**Testing**:
- `Unit: containment query returns only staff with role rn AND skill acls.`
- `Integration: overlapping time-off requests for same staff → second flagged conflicting.`
- `Integration (RLS): manager sees only own-facility staff.`

#### 2.2 — Credentialing lifecycle & primary source verification

**What**: `staff_credentials` and `facility_privileges` with lifecycle state machine and PSV workflow.

**Design**: suggestion-3 credential tables. Status state machine:
```
pending → active → renewal_pending → active
                 → expired
active/renewal_pending → suspended → active|revoked
```
- `verification` JSONB validated against `credential_verification.schema.json` (PSV date/method/source, NPDB/OIG/DEA checks per research §Credential data currency).
- Expiration alerts at configurable lead times (`org.settings.credential_renewal_alert_days`, default `[90,60,30,14,7]`) — Celery beat job emits notifications.
- **Bitemporal note (adopted from suggestion-4)**: credentials keep `valid_from`/`valid_to` (real-world validity) distinct from `created_at`/`updated_at` (transaction time) so a retroactively-discovered lapse can flag past assignments without rewriting history.
- Endpoints: `POST /api/v1/staff/{id}/credentials`, `POST /api/v1/credentials/{id}/verify`, `POST /api/v1/staff/{id}/privileges`.

**Testing**:
- `Unit: state machine rejects active → pending transition.`
- `Unit: expiration alert job selects credentials at exactly 30 days out.`
- `Integration: PSV with valid verification JSON sets primary_source_verified, psv_date.`
- `Integration (bitemporal): setting valid_to in the past flags assignments dated after it.`

#### 2.3 — Eligibility engine (credential-to-shift matching)

**What**: A service that, given a (staff, shift_template, facility, date) tuple, returns eligible/ineligible with reasons.

**Design**:
```python
@dataclass
class EligibilityResult:
    eligible: bool
    reasons: list[str]          # e.g. ["ACLS expired 2026-04-01", "no privilege at facility-002"]
    cost_tier: Literal["regular","overtime","float_pool","agency","premium"]

class EligibilityEngine:
    def check(self, staff: Staff, shift: ShiftTemplate, facility_id: UUID, on_date: date) -> EligibilityResult: ...
```
Checks, short-circuiting on first hard failure: (1) role in `staffing_requirements.roles`; (2) all `credentials_required` active and unexpired on `on_date`; (3) `facility_privileges` active for facility/service-line; (4) restriction conditions; (5) cost-tier derivation from `employment_type` (employed → float → agency ordering per research §Float pool). Implements **JCAHO competency** gate and **FHIR PractitionerRole** mapping (`standards.md`).

**Testing**:
- `Unit: RN with current BLS+ACLS and active privilege → eligible, cost_tier=regular.`
- `Unit: credential expired one day before shift → ineligible, reason names the credential.`
- `Unit: agency nurse → eligible but cost_tier=agency.`
- `Unit: privileged at facility A, shift at facility B → ineligible, "no privilege at facility-B".`
- `Fixture: 50-staff dataset → batch eligibility matrix matches expected snapshot.`

---

## Phase 3: Shift Definitions, Schedules & Manual Assignment

### Purpose
Introduce shift templates, schedule periods, and shift assignments with the eligibility gate enforced at assignment time. This delivers a working *manual* scheduler — managers can build compliant schedules by hand — and establishes the data the solver will later populate automatically. Shipping manual scheduling early de-risks the UI and data model before the NP-hard engine lands.

### Tasks

#### 3.1 — Shift templates & schedule periods

**What**: Models/CRUD for `shift_templates` (JSONB `staffing_requirements`) and `schedule_periods` lifecycle.

**Design**: suggestion-3 DDL. `schedule_periods.status`: `draft → published → locked → archived`. `staffing_requirements` validated against `staffing_requirements.schema.json`. Publishing a period emits notifications (research §staff view weeks in advance).

**Testing**:
- `Unit: end_date <= start_date → 422.`
- `Integration: publish draft → status published, published_at set, notifications enqueued.`
- `Unit: staffing_requirements with min > max for a role → 422.`

#### 3.2 — Shift assignments with eligibility enforcement

**What**: Assignment endpoint that calls the eligibility engine and rejects scope-of-practice violations.

**Design**: suggestion-3 `shift_assignments` (`UNIQUE (staff_id, shift_date, shift_template_id)` prevents double-booking same template; an additional service check prevents overlapping shifts across templates). `assignment_context` JSONB records `credentials_checked`, `credentials_verified_at`, `assignment_method`. `POST /api/v1/schedules/{period_id}/assignments` returns `201` or `409` with `EligibilityResult.reasons`.

**Testing**:
- `Integration: assign eligible RN → 201, assignment_context records checked credentials.`
- `Integration: assign ineligible staff → 409, reasons returned, no row written.`
- `Integration: double-book same staff overlapping shift → 409.`

#### 3.3 — Coverage & gap calculation

**What**: A service computing, per shift slot, required vs. assigned headcount by role, surfacing gaps.

**Design**:
```python
@dataclass
class CoverageCell:
    shift_template_id: UUID; shift_date: date; role: str
    required_min: int; required_max: int; assigned: int
    status: Literal["gap","met","over"]
```
- `GET /api/v1/schedules/{period_id}/coverage` returns the grid; feeds the manager heatmap and the open-shift generator (Phase 5).

**Testing**:
- `Unit: 4 RNs required, 3 assigned → status=gap, deficit 1.`
- `Integration: coverage grid for fixture period matches snapshot.`

---

## Phase 4: Constraint-Based Schedule Generation (Solver)

### Purpose
The heart of the product. Build the OR-Tools CP-SAT engine that auto-generates schedules satisfying hard constraints (labor law, rest, credential match) and optimising soft constraints (fairness, preferences). This is where the open-source platform matches Petal's AI-scheduling claim and exceeds rules-only incumbents.

### Tasks

#### 4.1 — Constraint compiler: rule JSONB → CP-SAT

**What**: Translate `scheduling_rule_sets.rules` JSONB into CP-SAT variables and constraints.

**Design**:
- Decision variables: `x[s, slot] ∈ {0,1}` = staff `s` assigned to shift `slot`. Slots derived from templates × dates × required headcount.
- Hard constraints implemented from the registry (1.4): coverage (`sum_s x[s,slot] ≥ required_min`), one-shift-per-staff-per-day, min-rest-between-shifts (forbid assignment pairs violating gap), max-consecutive-days/hours, eligibility (force `x=0` where Phase 2 engine returns ineligible), approved time-off (`x=0`).
- Each constraint type is a `ConstraintHandler.apply(model, vars, ctx)`; unknown types raise at compile, not solve.
```python
class ConstraintHandler(Protocol):
    type: str
    is_hard: bool
    def apply(self, model: cp_model.CpModel, vars: VarIndex, ctx: SolveContext) -> None: ...
```

**Testing**:
- `Unit: min_rest=11h handler forbids 7p-7a then 7a-7p next day for same staff.`
- `Unit: max_consecutive_days=5 forbids a 6th consecutive assignment.`
- `Unit: ineligible staff variable pinned to 0.`
- `Unit: unknown constraint type → CompileError before solve.`

#### 4.2 — Objective function: fairness & preferences

**What**: Weighted soft-constraint objective producing fair, preference-aware schedules.

**Design**: Minimise weighted sum of penalty terms: unmet soft rules (weekend distribution, holiday rotation), preference violations (preferred shifts/days), and **equity** terms approximating a Gini coefficient over weekend/night/holiday burden (research §DEI auditing, suggestion-3 `fairness_metrics`). Weights come from each rule's `weight`/`priority`. Output written to `schedule_periods.solver_results` JSONB (solve time, objective score, soft violations, fairness metrics).

**Testing**:
- `Unit: two equivalent solutions, one fairer → solver prefers lower Gini.`
- `Unit: honoring a preferred-shift request lowers objective.`
- `Integration: solver_results JSONB validates against schema and records soft violations.`

#### 4.3 — Solver runner (async) & decision support

**What**: Celery task running a solve with time limit, plus a feasibility/explanation report.

**Design**: `solver/runner.py::solve_period(period_id, params)` loads staff, eligibility matrix, applicable rule sets (resolved by `scope_type` priority), builds the model, solves with `solver_time_limit_seconds`. On infeasible, returns the minimal conflicting hard-constraint set (decision support — research §Scheduler decision support: "why is this infeasible?"). Writes draft assignments. `POST /api/v1/schedules/{period_id}/generate` → `202` + task id; `GET .../generate/{task_id}` polls status.

**Testing**:
- `Integration: feasible 20-staff/2-week ICU fixture → schedule with zero hard violations, coverage met.`
- `Integration: infeasible fixture (too few RNs) → status=infeasible, conflicting constraints reported.`
- `Integration: solve respects 60s limit → returns best-so-far if not optimal.`
- `Fixture: regression snapshot of objective score for a fixed seed.`

---

## Phase 5: Open-Shift Marketplace, Swaps & Real-Time Rescheduling

### Purpose
Connect generated schedules to live operations: post gaps as open shifts, let staff self-serve pickups and swaps with eligibility/approval gates, and handle call-outs with sub-minute notification of eligible staff. This delivers the NurseGrid self-service and Shift MedStaff float/agency strengths in one place.

### Tasks

#### 5.1 — Open shifts & bidding

**What**: Generate open shifts from coverage gaps; staff bid; managers approve.

**Design**: suggestion-3 `open_shifts` (JSONB `eligibility_criteria`) and `shift_bids`. Auto-generate from Phase 3.3 gaps with `eligibility_criteria` derived from the unfilled slot. Claiming runs the eligibility engine; ineligible bids rejected with reasons. Cost-tier ordering (employed→float→agency) ranks competing bids. Endpoints: `GET /api/v1/open-shifts` (filtered to eligible-for-me for staff role), `POST /api/v1/open-shifts/{id}/bid`, `POST /api/v1/bids/{id}/approve`.

**Testing**:
- `Integration: gap auto-creates open shift with correct eligibility_criteria.`
- `Integration: ineligible staff bid → 409 reasons; eligible bid → pending.`
- `Integration: approving a bid creates assignment, closes open shift, rejects other bids.`

#### 5.2 — Shift swaps

**What**: Peer-to-peer swap requests with two-stage (peer + manager) approval.

**Design**: suggestion-1 `shift_swap_requests`, status `pending_peer → pending_manager → approved|denied`. Both legs re-checked for eligibility and rest/overtime compliance before approval; on approval, both assignments' `staff_id` swap atomically in one transaction.

**Testing**:
- `Integration: valid swap → both assignments reassigned atomically.`
- `Integration: swap that would create a rest violation → denied with reason.`
- `Integration: peer declines → status pending_peer→denied, no manager step.`

#### 5.3 — Real-time call-out handling & notifications

**What**: Mark an assignment as call-out, instantly post the gap, and push to eligible staff over WebSocket/SMS in under a minute.

**Design**: `POST /api/v1/assignments/{id}/call-out` sets `status=no_show`, creates a `critical` open shift, computes the eligible-and-available set, and fans out via Redis pub/sub → WebSocket + SMS (notification preferences). `realtime/ws.py` hub keyed by `staff_id`.

**Testing**:
- `Integration: call-out creates critical open shift and publishes to eligible staff channel.`
- `Integration: staff with conflicting shift not notified.`
- `Unit: notification routing respects channel preferences (push vs sms).`

---

## Phase 6: Time & Attendance, Overtime & Compliance Monitoring

### Purpose
Close the loop between scheduled and actual hours, enforce FLSA/state overtime and rest rules proactively, and give managers a compliance dashboard that warns *before* violations occur. Required table-stakes and a precondition for payroll export.

### Tasks

#### 6.1 — Time entries & attendance variance

**What**: Clock-in/out capture and variance against scheduled hours.

**Design**: suggestion-1 `time_entries` with generated `total_hours`. `GET /api/v1/attendance/variance?period=` compares actual vs scheduled. Sources: mobile clock, EHR/time-clock integration (Phase 7).

**Testing**:
- `Unit: total_hours = (out-in) - break_minutes, computed correctly.`
- `Integration: clocked 13h on a 12h shift → variance +1h flagged.`

#### 6.2 — Overtime tracking & rule engine

**What**: Pay-period overtime accumulation enforcing FLSA + state rules.

**Design**: suggestion-1 `overtime_tracking`. Rule engine reads `facility.labor_rules` (weekly/daily thresholds, CA daily-OT, double-time) — `standards.md` FLSA §201 + state break/rest laws. Computes regular/OT/double-time per pay period.

**Testing**:
- `Unit: 45h week, 40h threshold → 5h OT.`
- `Unit: CA 13h day → 4h OT + 1h double-time.`
- `Unit: meal-break omission (CA §512) flagged.`

#### 6.3 — Compliance dashboard & proactive alerts

**What**: Real-time flags for impending overtime, rest violations, and fatigue thresholds with manager alerts.

**Design**: `GET /api/v1/compliance/alerts` returns active/projected violations (e.g., "assigning this shift pushes staff X to 42h → OT"). Evaluated at assignment time and on a beat schedule. Categories map to `scheduling_rules.rule_category` (labor_law, union_contract, fatigue).

**Testing**:
- `Integration: assignment projected to breach weekly OT → warning returned (assignment still allowed if soft).`
- `Integration: rest-period hard violation at assignment → blocked.`
- `Unit: alert dedupe — same projected violation not emitted twice per period.`

---

## Phase 7: External Integrations (EHR, HRIS, Payroll)

### Purpose
Wire the platform into the systems that own the source data — EHR census/acuity, HRIS employment/pay rates, payroll hours export — using FHIR and X12 per `standards.md`. Integration comes after the core engine so connectors feed a proven model.

### Tasks

#### 7.1 — Connector framework & FHIR PractitionerRole sync

**What**: Pluggable connector ABC and a FHIR R4 connector syncing provider credentials/roles.

**Design**:
```python
class Connector(ABC):
    name: str
    async def pull(self, since: datetime) -> list[ChangeEvent]: ...
    async def push(self, batch: list[ExportRecord]) -> PushResult: ...
```
- FHIR connector maps `PractitionerRole` → staff roles/specialties and competency; OAuth2 client-credentials per `standards.md`. SMART-on-FHIR token flow for EHR auth.

**Testing**:
- `Integration (mocked FHIR server): pull PractitionerRole → staff roles updated.`
- `Unit: FHIR resource missing specialty → mapped to None, no crash.`

#### 7.2 — EHR census/acuity ingestion

**What**: Poll/receive census snapshots feeding `census_snapshots`.

**Design**: suggestion-1 `census_snapshots`. Pull connectors for Epic/Cerner/Athena (research). Census → required headcount via `department.staffing_config.default_ratios` + `acuity_adjustments`. Drives acuity-based staffing (v1.1 feature).

**Testing**:
- `Integration (mocked EHR): census of 40 patients, ratio 4 → required_rn=10.`
- `Integration: high-acuity (>3.5) applies ratio reduction.`

#### 7.3 — Payroll export (X12 837/834) & HRIS sync

**What**: Export approved hours to payroll; sync employment/pay data from HRIS.

**Design**: Map `overtime_tracking` + `time_entries` to **X12 837/834** (`standards.md`, suggestion via `pyx12`); HRIS REST connector updates `employment_type`, `hourly_rate`, `fte_target`. `overtime_tracking.status` → `exported` after a successful push.

**Testing**:
- `Integration: approved pay period → valid X12 envelope (schema-validated segments).`
- `Integration (mocked HRIS): pay-rate change syncs to staff_members.`
- `Unit: export marks period exported; re-export blocked unless reopened.`

---

## Phase 8: Manager & Staff Frontend

### Purpose
Deliver the two human-facing surfaces: a manager dashboard (schedule grid, coverage heatmap, compliance alerts, solver controls) and a mobile-first staff PWA (schedule, availability, swaps, open-shift pickup). Built on the now-stable API.

### Tasks

#### 8.1 — Generated API client & auth flows

**What**: Typed TypeScript client from the OpenAPI 3.1 spec; OIDC/SAML + PWA password login.

**Design**: `openapi-typescript` generates `lib/api`. Auth context handles token refresh; SAML redirect for enterprise SSO.

**Testing**:
- `E2E (Playwright): OIDC login → dashboard; logout clears session.`
- `Unit: 401 triggers silent refresh, retries once.`

#### 8.2 — Manager dashboard

**What**: Schedule grid, coverage heatmap, generate-schedule control, compliance alert panel.

**Design**: Server components for initial schedule fetch; client grid for drag-assign (calls 3.2, shows eligibility 409 reasons inline). "Generate" triggers 4.3 and polls. Heatmap colours coverage cells (gap/met/over from 3.3).

**Testing**:
- `E2E: generate schedule → grid populates, zero-violation badge.`
- `E2E: drag ineligible staff onto shift → inline reason, no assignment.`

#### 8.3 — Staff PWA

**What**: Installable PWA: my-schedule, submit availability/time-off, request swaps, pick up open shifts, receive push.

**Design**: Next.js PWA (service worker, web-push). Open-shift list pre-filtered to eligible-for-me (5.1). Real-time call-out notifications via WebSocket (5.3).

**Testing**:
- `E2E: staff views schedule, submits time-off → appears pending.`
- `E2E: eligible open shift → bid → pending; ineligible shift hidden.`
- `E2E: PWA installable, service worker caches schedule for offline view.`

---

## Phase 9: AI — Demand Forecasting & Acuity-Based Staffing

### Purpose
Begin the AI-native differentiation: predict staffing demand by unit/shift from historical census, seasonality, and acuity, then auto-adjust staffing targets and open-shift generation — the underserved "predictive staffing" opportunity from `features.md`.

### Tasks

#### 9.1 — Demand forecasting model

**What**: Time-series model predicting census/required headcount per department/shift.

**Design**: Train on `census_snapshots` history with seasonality + known surge events; `prophet` or gradient-boosted baseline. Output to `ml_predictions(model_type='demand_forecast')` validated against `ml_prediction.schema.json` with 95% CI (suggestion-3 example). Retrained via scheduled Celery task; `model_version` recorded.

**Testing**:
- `Unit: forecast output validates against schema, includes CI.`
- `Integration: backtest on holdout → MAPE within configured threshold.`
- `Unit: insufficient history (<N weeks) → falls back to ratio-based target, flagged low-confidence.`

#### 9.2 — Acuity-based staffing target generation

**What**: Combine forecasts with acuity ratios to produce dynamic staffing targets feeding the solver and open-shift generation.

**Design**: Forecast census → `staffing_config` ratios/acuity adjustments → required headcount per slot, used as solver coverage targets (4.1) and gap detection (3.3) instead of static minimums.

**Testing**:
- `Integration: forecasted surge raises required_rn for affected shifts; open shifts auto-posted.`
- `Unit: acuity adjustment math matches Phase 7.2 deterministic path.`

---

## Phase 10: AI — Burnout, Call-Out & Fatigue Optimisation

### Purpose
Complete the AI-native vision with the differentiators no incumbent offers: burnout-risk prediction with intervention recommendations, call-out probability for proactive overstaffing, and fatigue-optimised scheduling beyond regulatory minimums.

### Tasks

#### 10.1 — Burnout risk prediction

**What**: Per-staff burnout risk score with contributing factors and recommended interventions.

**Design**: Features from assignment history (consecutive nights, OT hours 30d, weekend frequency, swap denials). Output to `ml_predictions(model_type='burnout_risk')` per suggestion-3 example (`risk_score`, `risk_level`, `contributing_factors`, `recommended_interventions`). Surfaced on manager dashboard.

**Testing**:
- `Unit: prediction validates against schema; factors sum to ≤1.`
- `Integration: high-night-burden staff → high risk with night-shift factor dominant.`

#### 10.2 — Call-out prediction & proactive overstaffing

**What**: Predict no-show likelihood per assignment; recommend pre-emptive coverage on high-risk shifts.

**Design**: `ml_predictions(model_type='callout_prediction')`; shifts above threshold flagged so 5.1 posts standby open shifts early (research §call-out prediction).

**Testing**:
- `Integration: high call-out-risk shift → standby open shift recommended pre-emptively.`
- `Unit: probability calibrated (reliability bins within tolerance on backtest).`

#### 10.3 — Fatigue-optimised scheduling objective

**What**: Add a fatigue penalty term (circadian disruption, turnaround time, consecutive-shift load) to the solver objective.

**Design**: Fatigue score per candidate assignment pattern (circadian science from `features.md` §fatigue management); added as a weighted soft term in 4.2. Configurable weight; OSHA fatigue-recordkeeping support (`standards.md`).

**Testing**:
- `Unit: rotating day→night→day pattern penalised vs. stable pattern.`
- `Integration: enabling fatigue weight reduces mean fatigue score with coverage still met.`
- `Fixture: objective regression snapshot with fatigue term enabled.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Multi-Tenant Data Core   ─── required by everything
    │
Phase 2: Staff, Credentialing & Eligibility    ─── requires P1
    │
Phase 3: Shifts, Schedules & Manual Assignment ─── requires P2
    │
Phase 4: Constraint-Based Generation (Solver)  ─── requires P3
    │
    ├── Phase 5: Marketplace, Swaps, Real-Time  ─── requires P3 (solver-independent); parallel with P6
    ├── Phase 6: Time & Attendance, Compliance  ─── requires P3; parallel with P5
    └── Phase 7: External Integrations          ─── requires P2/P3; parallel with P5/P6
                 │
Phase 8: Frontend (Manager + Staff PWA)         ─── requires stable API (P3–P7)
    │
Phase 9: AI — Demand Forecast & Acuity          ─── requires P7 (census data) + P4
    │
Phase 10: AI — Burnout, Call-Out, Fatigue       ─── requires P4 + P9 (assignment history & forecasts)
```

**Parallelism opportunities**
- After Phase 4, **Phases 5, 6, and 7** can be built concurrently (distinct domains over a shared model).
- Frontend (Phase 8) sub-tasks can begin against each API domain as it stabilises, rather than waiting for all of P5–P7.
- ML model training (9.1, 10.1, 10.2) can be developed offline against historical fixtures in parallel once census/assignment data exists.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`), including Testcontainers Postgres/Redis integration suites.
3. `ruff check` and `ruff format --check` pass; `mypy --strict` passes on backend.
4. Any new JSONB structure has a committed JSON Schema in `schemas/` and is validated at the API boundary.
5. New/changed endpoints appear in the auto-generated OpenAPI 3.1 spec, and the frontend client (where relevant) regenerates without error.
6. Alembic migration created for the phase's schema changes; `alembic upgrade head` and `downgrade -1` both succeed on a clean DB.
7. `docker compose up` builds and the affected feature works end-to-end.
8. RLS policies cover every new tenant-scoped table; a cross-tenant access test proves isolation.
9. Audit log records create/update/delete for new tenant entities.
10. New config options documented in `docs/deployment.md`; new endpoints in `docs/api.md`.
11. For solver/ML phases: a fixed-seed regression fixture guards against unintended objective/accuracy drift.
