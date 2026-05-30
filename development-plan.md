# Statistical Process Control — Phased Development Plan

> Project: 254-statistical-process-control · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into an implementation specification for an AI-native, open-source SPC platform. The database schema is based primarily on **Data Model 1 (Entity-Centric Normalized Relational)** for its regulated-industry traceability and AQDEF alignment, augmented with the `spc_constants` reference table from Data Model 2 and the time-partitioning strategy for measurement tables from Data Model 4.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend/core) | Python 3.12 | The core value is statistics (control charts, Cp/Cpk, Gage R&R, normality tests) and AI/ML (predictive drift, LLM analytics). NumPy/SciPy/pandas/statsmodels give exact, validated implementations of every required algorithm against ISO 7870/22514 reference data. Python is also the lingua franca for the LSTM/EWMA models in the backlog. |
| API framework | FastAPI | Generates an OpenAPI 3.1 spec automatically (required by `standards.md`), uses Pydantic v2 for request/response validation (maps to JSON Schema contracts), supports async for MQTT/webhook fan-out, and has first-class WebSocket support for live chart streaming. |
| Validation / typing | Pydantic v2 | Single source of truth for API schemas and the ISO/TR 11462-5 / AQDEF exchange contracts. Pydantic models serialise directly to JSON Schema. |
| Database | PostgreSQL 16 | The normalized model (Data Model 1) needs strong referential integrity for traceability; JSONB columns (sparingly used) cover config blobs; native range partitioning handles measurement volume; supports row-level security for multi-tenancy. PostgreSQL is available everywhere, satisfying the "cloud or private-intranet" deployment requirement without proprietary extensions. |
| Time-series partitioning | Native PG declarative partitioning (monthly range on `collected_at`) | Avoids the TimescaleDB dependency (Data Model 4 notes it is unavailable on some managed providers and complicates self-hosting) while still giving chunk-exclusion performance for the dominant "subgroups for chart item in time range" query. |
| ORM / migrations | SQLAlchemy 2.0 + Alembic | Mature, supports composite indexes, partitioned tables via DDL hooks, and versioned migrations needed for an evolving 30-table schema. |
| Task queue | Celery + Redis | Async workloads: capability recalculation, notification fan-out (email/webhook), MQTT/OPC-UA ingestion buffering, ML inference. Redis doubles as the broker and the cache/pub-sub backplane for WebSocket chart updates. |
| Real-time transport | WebSocket (FastAPI) + Redis Pub/Sub | Operators/engineers need live chart updates and alarm pushes; Redis pub/sub fans new subgroups out to all connected dashboard clients. |
| IoT ingestion | `paho-mqtt` (MQTT, ISO/IEC 20922) + `asyncua` (OPC UA) | The two standards named in `standards.md` as the preferred SPC ingestion path; both have mature Python clients. v1.1 phase. |
| Frontend | TypeScript + React 18 + Vite | Multi-user web UI with role-based progressive disclosure (operator vs engineer views). Vite for fast dev; React ecosystem has mature charting. |
| Charting | D3.js (custom SPC chart components) | Control charts have non-standard requirements (control/spec limits, zone shading for Western Electric rules, violation markers, phase boundaries) that off-the-shelf chart libraries do not render correctly. D3 gives full control. |
| UI component library | Mantine | Accessible, dense data-grid + form components suited to shop-floor data entry; good mobile responsiveness for the v1.1 mobile operator interface. |
| Auth | OAuth 2.0 / OIDC (Authlib) + JWT bearer tokens | `standards.md` requires OAuth 2.0 bearer for system-to-system and OIDC for enterprise IdP federation (AD/Okta). FDA 21 CFR Part 11 prohibits shared accounts → per-user identity enforced. |
| PDF reporting | WeasyPrint (HTML→PDF) | Customer-submission reports (capability studies, control charts) rendered from HTML templates; deterministic, no headless browser dependency. |
| Excel export | `openpyxl` | Engineers expect XLSX export of raw subgroup data and capability summaries. |
| LLM integration | Provider-agnostic via LiteLLM | Conversational analytics + chart interpretation (backlog). LiteLLM lets self-hosted deployments point at a local model and SaaS at a hosted provider — supports the data-isolation requirement of regulated industries. |
| ML framework | PyTorch (LSTM) + statsmodels (EWMA) | Predictive drift detection ensemble (backlog). |
| Testing | pytest + pytest-asyncio + Vitest + Playwright | pytest for backend unit/integration; Vitest for React units; Playwright for E2E. SPC algorithm correctness validated against ISO/TR 11462-3 reference data sets. |
| Code quality | ruff + mypy (Python); eslint + prettier + tsc (TS) | Lint, format, type-check both stacks. |
| Containerisation | Docker + docker-compose | `README.md` requires both cloud and private-intranet deployment; compose stands up api + worker + postgres + redis + frontend for self-hosting. |
| Package managers | uv (Python), pnpm (frontend) | Fast, reproducible installs. |

### Project Structure

```
statistical-process-control/
├── README.md
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── backend/
│   ├── pyproject.toml                 # uv-managed
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/                  # one migration per phase
│   ├── src/
│   │   └── spc/
│   │       ├── __init__.py
│   │       ├── main.py                # FastAPI app factory
│   │       ├── config.py              # Pydantic Settings (env-driven)
│   │       ├── db/
│   │       │   ├── base.py            # SQLAlchemy declarative base + engine
│   │       │   ├── session.py         # session + RLS tenant context
│   │       │   └── models/            # ORM models, grouped by domain
│   │       │       ├── org.py         # tenant, plant, area, station
│   │       │       ├── auth.py        # user, role, user_role, esignature
│   │       │       ├── part.py        # part, characteristic
│   │       │       ├── plan.py        # control_plan(_item), chart_config, rule_set
│   │       │       ├── measurement.py # subgroup, measurement, attribute_subgroup
│   │       │       ├── alarm.py       # alarm, assignable_cause, corrective_action
│   │       │       ├── capability.py  # capability_study
│   │       │       ├── gage.py        # gage, gage_rr_study
│   │       │       ├── ingestion.py   # data_source
│   │       │       ├── audit.py       # audit_log
│   │       │       └── notify.py      # notification_rule, notification_log
│   │       ├── schemas/               # Pydantic request/response models
│   │       ├── api/
│   │       │   ├── deps.py            # auth, tenant, pagination deps
│   │       │   ├── v1/                # one router module per resource
│   │       │   └── ws.py             # WebSocket chart/alarm streaming
│   │       ├── core/                  # PURE statistical engine (no I/O)
│   │       │   ├── constants.py       # A2,A3,B3,B4,c4,d2,d3,d4 tables
│   │       │   ├── variable_charts.py # Xbar-R, Xbar-S, I-MR, EWMA
│   │       │   ├── attribute_charts.py# P, NP, C, U
│   │       │   ├── rules.py           # Western Electric + Nelson rule engine
│   │       │   ├── capability.py      # Cp, Cpk, Pp, Ppk, ppm, normality
│   │       │   └── msa.py             # Gage R&R (Average-Range, ANOVA)
│   │       ├── services/              # business logic orchestrating core + db
│   │       │   ├── charting.py
│   │       │   ├── alarms.py
│   │       │   ├── notifications.py
│   │       │   ├── audit.py
│   │       │   ├── reports.py         # PDF/Excel
│   │       │   └── exchange.py        # AQDEF/ISO 11462-5 import/export
│   │       ├── ingestion/             # MQTT, OPC-UA, gage serial (v1.1)
│   │       ├── ai/                    # LLM analytics + ML drift (backlog)
│   │       └── workers/               # Celery tasks
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/                  # ISO 11462-3 reference datasets, DFQ files
└── frontend/
    ├── package.json                   # pnpm
    ├── Dockerfile
    ├── vite.config.ts
    ├── src/
    │   ├── main.tsx
    │   ├── api/                       # generated TS client from OpenAPI
    │   ├── auth/
    │   ├── components/
    │   │   └── charts/                # D3 SPC chart components
    │   ├── features/                  # data-entry, dashboard, plans, alarms, capability
    │   └── routes/
    └── tests/
```

---

## Phase 1: Foundation & Tenancy

### Purpose
Establish the runnable skeleton: project scaffolding, configuration, database connection with multi-tenant isolation, migrations, and the organisation/equipment hierarchy. Nothing in this phase computes statistics, but every later phase depends on the tenant context, the ISA-95 hierarchy, and the migration pipeline existing.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Create the FastAPI app factory, Pydantic settings, Docker Compose stack, and CI lint/type/test pipeline.

**Design**:
- `config.py` exposes a `Settings(BaseSettings)`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_minutes: int = 30
    cors_origins: list[str] = ["http://localhost:5173"]
    deployment_mode: Literal["cloud", "intranet"] = "cloud"
    model_config = SettingsConfigDict(env_prefix="SPC_", env_file=".env")
```
- `main.py` `create_app()` registers routers, CORS, exception handlers, and a `/healthz` endpoint returning `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `api`, `worker`, `frontend`. `.env.example` documents every `SPC_*` var.
- All API errors return RFC 9457 problem+json: `{"type","title","status","detail","instance"}`.

**Testing**:
- `Unit: Settings loads from env vars with SPC_ prefix → correct typed fields`.
- `Unit: missing SPC_DATABASE_URL → ValidationError naming the field`.
- `Integration: GET /healthz with db+redis up → 200, all flags true`.
- `Integration: GET /healthz with redis down → 200, redis flag false`.
- `CI: ruff, mypy, pytest all exit 0 on empty skeleton`.

#### 1.2 — Database base, sessions, and tenant RLS

**What**: SQLAlchemy engine/session, declarative base with shared columns, and PostgreSQL row-level-security tenant isolation.

**Design**:
- `TimestampMixin` (`created_at`, `updated_at`) and `UUIDPKMixin` (`id UUID default gen_random_uuid()`).
- A `tenant_id` context var set per request from the JWT; the session sets `SET LOCAL app.current_tenant = :tenant_id` so RLS policies filter rows.
- RLS policy template applied to every tenant-scoped table:
```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON <t>
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

**Testing**:
- `Integration (real PG): insert rows for tenant A and B → query under tenant A context returns only A's rows`.
- `Integration: query with no tenant context set → returns zero rows (fail-closed)`.
- `Unit: session dependency yields and closes session, rolls back on exception`.

#### 1.3 — Organisation & equipment hierarchy (ISA-95)

**What**: `tenant`, `plant`, `production_area`, `station` tables + CRUD endpoints.

**Design**:
- DDL exactly as Data Model 1 "Organisation & Tenant Management" (UUID PKs, `plant.country_code` ISO 3166-1 alpha-2, `production_area.parent_area_id` self-ref for cell→line nesting, `station.external_station_id` for Minitab-style mapping).
- Endpoints (all tenant-scoped, admin role required for writes):
  - `POST/GET/PATCH/DELETE /v1/plants`
  - `POST/GET/PATCH/DELETE /v1/plants/{plant_id}/areas`
  - `POST/GET/PATCH/DELETE /v1/areas/{area_id}/stations`
- Alembic migration `0001_foundation`.

**Testing**:
- `Unit: create plant with invalid country_code "XXX" → 422`.
- `Integration: create area with parent_area_id in a different plant → 400`.
- `Integration: full hierarchy create → GET /v1/plants/{id}/tree returns nested area/station structure`.
- `Integration: delete plant with child areas → 409 (referential guard) unless cascade flag`.

---

## Phase 2: Identity, Access Control & Audit Trail

### Purpose
Add authentication, role-based authorisation (operator/engineer/manager/admin), and the FDA 21 CFR Part 11 audit trail and electronic-signature primitives. Because Part 11 forbids shared accounts and requires tamper-evident audit logs, this must exist before any data-recording phase so every measurement and change is attributable.

### Tasks

#### 2.1 — Users, roles, and OAuth2/OIDC auth

**What**: `app_user`, `role`, `user_role` tables; password + OIDC login; JWT issuance; RBAC dependency.

**Design**:
- DDL per Data Model 1 "User & Access Control". Passwords hashed with `argon2`.
- System roles seeded per tenant: `operator`, `engineer`, `manager`, `admin` with `permissions` JSONB arrays of scopes (e.g. `["measurement:write","alarm:ack"]`).
- `user_role.plant_id` NULL = all plants; non-null = plant-scoped grant.
- Endpoints:
  - `POST /v1/auth/login` → `{access_token, refresh_token, token_type}` (OAuth2 password grant).
  - `POST /v1/auth/refresh`.
  - `GET /v1/auth/oidc/login` / `GET /v1/auth/oidc/callback` (Authlib OIDC code flow → AD/Okta).
  - `GET/POST/PATCH /v1/users`, `GET/POST /v1/roles`, `POST /v1/users/{id}/roles`.
- `require_scope(*scopes)` FastAPI dependency that reads JWT, resolves effective permissions (union of roles, plant-filtered), and 403s on missing scope.

**Testing**:
- `Unit: argon2 hash+verify round-trips; wrong password fails`.
- `Integration: login with valid creds → JWT decodes to correct user_id, tenant_id, scopes`.
- `Integration: operator token calls engineer-only endpoint → 403`.
- `Integration (mocked OIDC): callback with valid code → user provisioned, JWT issued`.
- `Integration: plant-scoped engineer queries another plant's data → 403`.

#### 2.2 — Audit trail & electronic signatures (21 CFR Part 11)

**What**: `audit_log`, `electronic_signature` tables; automatic change capture; signature capture endpoint.

**Design**:
- DDL per Data Model 1 "Audit Trail" and `electronic_signature`.
- An SQLAlchemy `after_flush` event listener writes `audit_log` rows (`action`, `table_name`, `record_id`, `old_values`, `new_values` JSONB diff, `user_id`, `ip_address`, `reason`) for every INSERT/UPDATE/DELETE on a registered set of compliance-critical tables.
- `reason` is required on mutations of signed/approved records; absence → 422.
- Electronic signature: `POST /v1/signatures` with `{record_type, record_id, meaning, password}`; re-authenticates password (Part 11 §11.200), stores full name + timestamp + meaning.
- `audit_log` is append-only: no UPDATE/DELETE endpoints; DB trigger rejects mutation.

**Testing**:
- `Integration: update a characteristic spec limit → audit_log row with old/new values and user_id`.
- `Integration: attempt UPDATE on audit_log → DB error (immutable)`.
- `Integration: sign a control plan without re-entering password → 401`.
- `Unit: diff function produces only changed fields in old_values/new_values`.
- `Integration: mutate signed record without reason → 422 naming reason`.

---

## Phase 3: SPC Statistical Engine (Core)

### Purpose
Implement the pure statistical heart of the product — control-limit computation, capability indices, and the rule engine — with **no I/O dependencies** so it can be exhaustively validated against ISO/TR 11462-3 reference datasets. This is the differentiated core; everything else is plumbing around it.

### Tasks

#### 3.1 — SPC constants & variable control charts

**What**: Constant tables and limit/centre-line computation for Xbar-R, Xbar-S, I-MR, and EWMA.

**Design**:
- `constants.py` exposes `A2,A3,B3,B4,c4,d2,d3,d4` for subgroup sizes 2–25 (seed values per Data Model 2 `spc_constants`). Also persisted to a `spc_constants` reference table for app queries.
- Pure functions, all returning a dataclass:
```python
@dataclass(frozen=True)
class ChartLimits:
    center_line: float
    ucl: float
    lcl: float
    center_line_2: float | None = None  # R/S chart CL
    ucl_2: float | None = None
    lcl_2: float | None = None

def xbar_r_limits(subgroups: list[list[float]]) -> ChartLimits: ...
def xbar_s_limits(subgroups: list[list[float]]) -> ChartLimits: ...
def i_mr_limits(values: list[float]) -> ChartLimits: ...     # MR window=2
def ewma_series(values, target, lambda_=0.2, L=3.0) -> tuple[list[float], ChartLimits]: ...
```
- Formulae per ISO 7870-2 / AIAG SPC Manual: e.g. Xbar UCL = `x̄̄ + A2·R̄`; R UCL = `D4·R̄`; S chart uses `B3/B4·s̄`, centre `c4`-corrected.

**Testing**:
- `Unit (fixture, ISO 11462-3): xbar_r_limits on reference subgroups → UCL/LCL/CL match published values within 1e-6`.
- `Unit: i_mr_limits with constant input → range=0, UCL=LCL=CL`.
- `Unit: subgroup of size 1 passed to xbar_r → ValueError`.
- `Unit: ewma_series with lambda=1 → equals raw values (no smoothing)`.

#### 3.2 — Attribute control charts

**What**: P, NP, C, U chart limits with variable-sample-size handling.

**Design**:
```python
def p_limits(defectives: list[int], sizes: list[int]) -> list[ChartLimits]   # per-point limits if n varies
def np_limits(defectives: list[int], n: int) -> ChartLimits
def c_limits(defects: list[int]) -> ChartLimits
def u_limits(defects: list[int], sizes: list[int]) -> list[ChartLimits]
```
- P-chart: `p̄ ± 3·sqrt(p̄(1-p̄)/nᵢ)`; LCL floored at 0. U-chart similarly per-point.

**Testing**:
- `Unit: p_limits with constant n → single limit set; with varying n → stepped limits`.
- `Unit: c_limits where 3-sigma LCL<0 → LCL clamped to 0`.
- `Unit: np_limits negative count → ValueError`.

#### 3.3 — Western Electric & Nelson rule engine

**What**: Evaluate a plotted series against configurable rule sets and emit violations.

**Design**:
```python
@dataclass(frozen=True)
class RuleViolation:
    rule_number: int
    rule_name: str
    point_indices: list[int]   # contributing points
    severity: str              # 'warning'|'alarm'|'critical'

def evaluate_rules(points, limits, rule_set: RuleSet) -> list[RuleViolation]
```
- Implement all 8 Western Electric rules and 8 Nelson rules (1 pt beyond 3σ; 2/3 beyond 2σ; 4/5 beyond 1σ; 8 on one side; 6 trending; 14 alternating; 15 within 1σ; 8 beyond 1σ). Zones derived from `limits` (σ = (UCL−CL)/3).
- `RuleSet` carries enabled rule numbers + per-rule severity + parameters (sigma multiple, run length).

**Testing**:
- `Unit: single point above UCL → rule 1 violation at that index`.
- `Unit: 8 consecutive points below CL → rule (one-side run) violation listing all 8`.
- `Unit: 6 monotonically increasing points → trend rule violation`.
- `Unit: in-control random series → no violations`.
- `Unit: disabled rule does not fire even when pattern present`.

#### 3.4 — Process capability & normality

**What**: Cp, Cpk, Pp, Ppk, Cpm, ppm, and normality testing.

**Design**:
```python
@dataclass(frozen=True)
class CapabilityResult:
    cp: float | None; cpk: float | None
    pp: float | None; ppk: float | None
    cpm: float | None
    process_mean: float; process_stddev_within: float; process_stddev_overall: float
    ppm_below_lsl: float; ppm_above_usl: float; ppm_total: float
    is_normal: bool; normality_p_value: float

def capability(values, subgroups, usl, lsl, target=None) -> CapabilityResult
```
- Within-σ from `R̄/d2` (or `s̄/c4`); overall-σ from sample stddev (ISO 22514-2). Cp=(USL−LSL)/6σ_within; Cpk=min((USL−μ),(μ−LSL))/3σ_within; Pp/Ppk use σ_overall. ppm via normal CDF tails. Normality: Anderson–Darling (statsmodels).
- One-sided spec (only USL or LSL) → Cp/Pp None, Cpk/Ppk single-sided.

**Testing**:
- `Unit (fixture, ISO 11462-3): capability on reference data → Cp/Cpk/Pp/Ppk match published within 1e-4`.
- `Unit: perfectly centred process, USL/LSL symmetric → Cp==Cpk`.
- `Unit: only USL provided → Cp None, Cpk computed from USL side`.
- `Unit: clearly non-normal input → is_normal False`.

---

## Phase 4: Parts, Control Plans & Chart Configuration

### Purpose
Model the quality-planning entities that tell the engine *what* to monitor and *how*: parts and their characteristics (with spec limits), control plans linking characteristics to chart types and sampling, and time-versioned chart configurations holding control limits. After this phase a chart can be fully configured but not yet fed data.

### Tasks

#### 4.1 — Parts & characteristics (AQDEF-aligned)

**What**: `part`, `characteristic` tables + CRUD.

**Design**:
- DDL per Data Model 1 "Part & Characteristic Definitions" with AQDEF K-field comments (K1001 part_number, K2110/K2111 spec limits, K2101 nominal). `is_special_char` + `special_char_class` for IATF 16949 special characteristics.
- `POST /v1/parts`, `POST /v1/parts/{id}/characteristics`, etc. Spec-limit edits trigger audit + require `reason`.
- Validation: `lower_spec_limit < upper_spec_limit` when both present; `data_type ∈ {variable, attribute}`.

**Testing**:
- `Integration: create part + characteristic → readable with correct AQDEF-mapped fields`.
- `Unit: LSL >= USL → 422`.
- `Integration: edit USL on existing characteristic without reason → 422`.

#### 4.2 — Control plans & items

**What**: `control_plan`, `control_plan_item` tables + lifecycle.

**Design**:
- DDL per Data Model 1 "Control Plan & Chart Configuration". `control_plan.status ∈ {draft, active, superseded}`; activation requires electronic signature (Phase 2) → sets `approved_by/approved_at`.
- `control_plan_item` binds characteristic → station → `chart_type` + `subgroup_size` + `sampling_frequency` + `reaction_plan`.
- Endpoints: `POST /v1/control-plans`, `POST /v1/control-plans/{id}/items`, `POST /v1/control-plans/{id}/activate` (signature required), `POST /v1/control-plans/{id}/supersede`.

**Testing**:
- `Integration: activate plan → status active, signature recorded, audit logged`.
- `Integration: add attribute chart_type to a variable characteristic → 400 (mismatch)`.
- `Integration: edit item on an active plan → creates new plan revision (immutability of approved plan)`.

#### 4.3 — Chart config & rule-set assignment

**What**: `chart_config` (temporal control limits), `spc_rule_set`, `spc_rule`, `chart_rule_assignment`.

**Design**:
- DDL per Data Model 1. `chart_config` has `effective_from`/`effective_to` (NULL = current), `phase ∈ {I, II}`. Recomputing limits closes the old config (`effective_to = now`) and inserts a new current one — the historical record needed for "what limits were in effect when this alarm fired?".
- Seed two system rule sets per tenant: `western_electric` and `nelson`, each with its rules and default severities (maps to Phase 3.3 `RuleSet`).
- `POST /v1/items/{id}/recalculate-limits?phase=I|II` runs Phase 3 engine over a subgroup window, writes new `chart_config`.

**Testing**:
- `Integration: recalc limits twice → first config gets effective_to set, second is current`.
- `Integration: assign nelson rule set to a chart item → evaluation uses Nelson rules`.
- `Unit: phase II recalc with <20 subgroups → warning flag in response`.

---

## Phase 5: Measurement Ingestion, Live Charting & Alarming

### Purpose
The product becomes operational: record subgroups (manual, file, REST), compute plotted statistics, evaluate rules, raise alarms, and stream live updates to dashboards. This is the MVP heartbeat that delivers real-time SPC monitoring.

### Tasks

#### 5.1 — Measurement recording (variable & attribute)

**What**: `subgroup`, `measurement`, `attribute_subgroup` tables (monthly-partitioned) + ingestion endpoints.

**Design**:
- DDL per Data Model 1 "Measurement Data". `subgroup` and `attribute_subgroup` declared `PARTITION BY RANGE (collected_at)`; an Alembic-managed routine creates next-month partitions.
- `POST /v1/items/{id}/subgroups` body: `{collected_at, station_id, lot_number, samples:[float], data_source}`. On write: compute `mean_value`, `range_value`, `std_dev_value` via Phase 3, persist subgroup + child measurements, then trigger 5.2 evaluation.
- `POST /v1/items/{id}/attribute-subgroups` body: `{inspected_count, defective_count|defect_count}`.
- REST ingestion endpoint mirrors Minitab station semantics: accept up to 50 subgroups / 1000 observations per request; partial-failure response lists rejected rows.
- Bulk file import: `POST /v1/items/{id}/import` (CSV) reusing the same validation path.

**Testing**:
- `Unit: subgroup with wrong sample count vs item.subgroup_size → 422`.
- `Integration: post variable subgroup → mean/range computed and stored correctly`.
- `Integration: batch of 50 subgroups, one malformed → 207 multi-status, 49 stored`.
- `Integration: subgroup collected_at in a future month → partition auto-created, row stored`.

#### 5.2 — Alarm generation pipeline

**What**: `alarm` table + evaluation that runs rule engine on each new point.

**Design**:
- DDL per Data Model 1 "Alarms & Corrective Actions" (`alarm`). On each new subgroup, the `alarms` service loads the current `chart_config`, the trailing N points needed by enabled rules, runs `evaluate_rules`, and inserts one `alarm` per new `RuleViolation` (deduplicated by rule+point). `status` lifecycle: `open → acknowledged → resolved | false_positive`.
- Emits a Redis pub/sub event `alarm.triggered` for WebSocket + notification fan-out (Phase 6).
- `POST /v1/alarms/{id}/acknowledge` (scope `alarm:ack`).

**Testing**:
- `Integration: post point beyond UCL → exactly one rule-1 alarm, status open`.
- `Integration: re-post same in-control data → no duplicate alarms`.
- `Integration: acknowledge alarm → status acknowledged, acknowledged_by/at set, audit logged`.
- `Integration: alarm.triggered event published to Redis on violation`.

#### 5.3 — Live chart data API & WebSocket streaming

**What**: Read endpoint returning everything a chart needs, plus a WebSocket for live appends.

**Design**:
- `GET /v1/items/{id}/chart?from=&to=&limit=` returns:
```json
{
  "chart_type": "xbar_r",
  "points": [{"subgroup_number":142,"collected_at":"...","mean":10.02,"range":0.013}],
  "limits": {"ucl":10.06,"lcl":9.94,"center_line":10.0,"ucl_2":0.028,"lcl_2":0.0,"center_line_2":0.013,"phase":"II"},
  "violations": [{"rule_number":1,"point_indices":[140],"severity":"alarm"}],
  "zones": {"sigma": 0.02}
}
```
  Limits resolved per-point against the temporally-correct `chart_config`.
- `WS /v1/items/{id}/stream`: on connect, replay last N points; subscribe to Redis `subgroup.recorded:{item_id}` and `alarm.triggered:{item_id}`; push appended points/violations as JSON frames. Auth via token query param; scope-checked.

**Testing**:
- `Integration: chart endpoint returns points within range with correct historical limits`.
- `Integration (mocked redis): WS client receives appended point after a subgroup POST`.
- `Integration: WS connect without valid token → closed with 1008`.

---

## Phase 6: Alerting, Corrective Actions, Capability & Reporting

### Purpose
Close the MVP loop: notify the right people (in-app/email/webhook), capture assignable causes and corrective actions at the point of alarm, run on-demand capability studies, and export audit-ready PDF/Excel reports. After this phase the MVP feature set from `features.md` is complete.

### Tasks

#### 6.1 — Notification engine

**What**: `notification_rule`, `notification_log` + Celery fan-out.

**Design**:
- DDL per Data Model 1 "Notification Configuration". A Redis subscriber for `alarm.triggered` enqueues a Celery task that matches `notification_rule`s (by `event_type` + `severity_filter`) and dispatches per channel:
  - `email` (SMTP, templated),
  - `webhook` (POST JSON with HMAC `X-SPC-Signature` header using rule `config.secret`),
  - `in_app` (persisted + WS push).
- Every attempt logged to `notification_log` with status + response_code; webhook failures retried with exponential backoff (Celery `retry`).

**Testing**:
- `Integration (mocked SMTP): alarm matching an email rule → email task sent, notification_log status sent`.
- `Integration (mocked HTTP): webhook dispatch includes valid HMAC signature`.
- `Integration: webhook returns 500 → retried, then logged failed after max retries`.
- `Integration: alarm not matching any rule → no notifications`.

#### 6.2 — Assignable cause & corrective action workflow

**What**: `assignable_cause`, `alarm_cause`, `corrective_action` tables + endpoints.

**Design**:
- DDL per Data Model 1. `assignable_cause.category` uses Ishikawa 5M (`man/machine/material/method/measurement`). Tenant-managed cause library.
- `POST /v1/alarms/{id}/causes` links causes; `POST /v1/alarms/{id}/corrective-actions` creates an action (`action_type ∈ {containment,corrective,preventive}`, `assigned_to`, `due_date`). Action lifecycle `open → in_progress → completed → verified` (verification requires manager scope + signature).
- Resolving an alarm requires at least one assigned cause (configurable per tenant).

**Testing**:
- `Integration: record cause + corrective action on alarm → linked and audited`.
- `Integration: resolve alarm with no cause (strict tenant) → 409`.
- `Integration: verify corrective action as operator → 403; as manager → 200 + signature`.

#### 6.3 — Capability studies & reporting

**What**: `capability_study` table, study endpoint, and PDF/Excel export.

**Design**:
- DDL per Data Model 1 "Capability Studies". `POST /v1/items/{id}/capability-study?from=&to=&type=short_term|long_term` runs Phase 3.4, persists the result.
- `GET /v1/items/{id}/report.pdf` (WeasyPrint): control chart image (server-rendered SVG via the same point/limit data), histogram, normal probability plot, capability summary, and the audit header (part, characteristic, limits, operator, period) for customer submission — terminology aligned to AIAG-VDA SPC Yellow Volume 2026.
- `GET /v1/items/{id}/export.xlsx` (openpyxl): raw subgroups + computed stats + capability sheet.

**Testing**:
- `Integration: run capability study → row persisted with Cp/Cpk/Pp/Ppk matching engine`.
- `Integration: GET report.pdf → 200, application/pdf, non-empty, contains part number text`.
- `Integration: export.xlsx → openpyxl reopens it, subgroup count matches DB`.

---

## Phase 7: Frontend Web Application

### Purpose
Deliver the multi-user web UI with role-based progressive disclosure: simple data entry + live charts for operators, deep capability/configuration for engineers, and summary dashboards for managers. This makes the MVP usable by its actual personas rather than via API only.

### Tasks

#### 7.1 — App shell, auth & generated API client

**What**: React shell, OIDC/password login, route guards, typed API client.

**Design**:
- Generate `frontend/src/api/` TypeScript client from the backend OpenAPI spec (`openapi-typescript` + a thin fetch wrapper attaching the JWT).
- Auth context stores token, decodes scopes; `<RequireScope scope="...">` guards routes. Token refresh on 401.
- Layout: top nav (plant selector), left nav filtered by role.

**Testing**:
- `Vitest: RequireScope renders children when scope present, redirects otherwise`.
- `Vitest: API client attaches Authorization header and retries once after refresh on 401`.
- `Playwright (E2E): login → land on role-appropriate dashboard`.

#### 7.2 — D3 SPC chart components

**What**: Reusable control-chart components for variable and attribute charts.

**Design**:
- `<ControlChart type points limits violations zones />`: plots points + connecting line, CL/UCL/LCL lines, optional R/S subchart, 1σ/2σ zone shading, violation markers coloured by severity, phase-boundary verticals. Tooltip shows subgroup detail.
- `<CapabilityHistogram />` (bars + fitted normal + spec lines) and `<ProbabilityPlot />`.
- Charts consume the 5.3 chart payload directly and subscribe to the item WebSocket for live appends.

**Testing**:
- `Vitest: ControlChart renders one marker per point and N violation markers`.
- `Vitest: point beyond UCL gets the 'alarm' CSS class`.
- `Playwright (E2E): open chart → new subgroup POST → point appears live without reload`.

#### 7.3 — Feature screens

**What**: Data entry, alarm management, control-plan editor, capability, manager dashboard.

**Design**:
- **Data entry** (operator): pick item, enter subgroup samples in a focused grid, submit → immediate chart feedback + alarm banner.
- **Alarm management**: list/filter alarms, acknowledge, assign cause (5M picker), create corrective action.
- **Control-plan editor** (engineer): build plan items, set chart type/subgroup/rules, activate with signature dialog.
- **Capability** (engineer): run study, view indices + plots, export report.
- **Manager dashboard**: tiles per line/plant showing OOC counts, worst Cpk, open alarms (cross-line view per InfinityQS pattern).

**Testing**:
- `Playwright (E2E): operator enters out-of-spec subgroup → alarm banner + red point`.
- `Playwright (E2E): engineer activates control plan via signature dialog → status active`.
- `Playwright (E2E): manager dashboard shows tile with non-zero open-alarm count after E2E above`.

---

## Phase 8: Standards Interoperability & Open API Surface

### Purpose
Deliver the documented-open-API and data-exchange differentiators identified as market gaps: a published OpenAPI 3.1 spec, OAuth2 client-credentials for system integration, and AQDEF / ISO/TR 11462-5 import/export so the platform interoperates with other SPC tools and ERP/MES.

### Tasks

#### 8.1 — Published OpenAPI spec & API keys / client-credentials

**What**: Polished OpenAPI 3.1 document, Swagger/Redoc UI, and machine API credentials.

**Design**:
- Tag/describe every endpoint; publish at `/openapi.json` + `/docs`. Generate the frontend client and a versioned changelog from it.
- `api_key` issuance for stations/ERP: OAuth2 client-credentials grant returning a scoped token; per-key rate limiting.

**Testing**:
- `Integration: /openapi.json validates against the OpenAPI 3.1 meta-schema`.
- `Integration: client-credentials grant → token with only ingestion scopes`.
- `Integration: spec contains every v1 route with documented request/response schemas`.

#### 8.2 — AQDEF / ISO 11462-5 exchange

**What**: Import/export of DFQ (AQDEF) files and the ISO/TR 11462-5 quality data exchange format.

**Design**:
- `exchange.py`: DFQ parser mapping K-fields → part/characteristic/measurement (K1001→part_number, K2101→nominal, K2110/K2111→spec limits, K0001→measured value); inverse exporter.
- `POST /v1/import/dfq` (upload) → creates/updates parts + characteristics + subgroups; `GET /v1/items/{id}/export/dfq`.
- JSON-Schema-validated ISO 11462-5 export for tool-to-tool interchange.

**Testing**:
- `Unit (fixture DFQ): parse sample .dfq → correct part/characteristic/measurement objects`.
- `Integration: import DFQ then export DFQ → round-trips key K-fields without loss`.
- `Unit: malformed DFQ K-field → descriptive parse error with line number`.

---

## Phase 9: IoT Ingestion & MSA (v1.1)

### Purpose
Add the v1.1 "should-have" capabilities: direct machine data ingestion via MQTT and OPC-UA (the standards-preferred path) and the AIAG-compliant Gage R&R module required before SPC for automotive customers.

### Tasks

#### 9.1 — MQTT & OPC-UA connectors

**What**: `data_source` table + background ingestion workers writing subgroups.

**Design**:
- DDL per Data Model 1 "Data Source Configuration" (`connection_config` JSONB per protocol).
- MQTT worker (`paho-mqtt`): subscribes to configured topics, maps payloads to `(item_id, samples)` via a per-source mapping template, dispatches into the 5.1 ingestion path.
- OPC-UA worker (`asyncua`): polls/subscribes to node IDs at `polling_interval_ms`.
- `last_received_at` heartbeat; stale-source alert.

**Testing**:
- `Integration (embedded MQTT broker): publish a measurement message → subgroup recorded via standard path`.
- `Integration (mock OPC-UA server): node value change → subgroup recorded`.
- `Integration: source silent past threshold → stale-source alarm raised`.

#### 9.2 — Gage R&R (MSA) module

**What**: `gage`, `gage_rr_study` tables + AIAG-compliant computation.

**Design**:
- DDL per Data Model 1 "Gage / MSA". `msa.py` implements Average-&-Range and ANOVA methods → `%EV, %AV, %GRR, %PV, ndc`; acceptance bands per AIAG MSA (GRR <10% acceptable, 10–30% marginal, >30% unacceptable).
- `POST /v1/gages/{id}/rr-studies` body: operators × parts × trials matrix.
- Calibration tracking: `gage.calibration_due` → due/expired status + notification.

**Testing**:
- `Unit (fixture, AIAG MSA reference): ANOVA GRR matches manual example within tolerance`.
- `Unit: %GRR 8% → is_acceptable True; 25% → marginal; 40% → unacceptable`.
- `Integration: gage past calibration_due → status expired + notification`.

---

## Phase 10: AI-Native Capabilities (Backlog)

### Purpose
Deliver the AI-native differentiators that distinguish this platform from every incumbent: conversational analytics, LLM chart interpretation, predictive drift detection, and assisted control-plan generation.

### Tasks

#### 10.1 — Conversational quality analytics

**What**: Natural-language query over SPC data ("what lines are out of control this week?").

**Design**:
- LiteLLM-backed agent with read-only tools mapping to existing services (`list_out_of_control`, `get_capability_trend`, `top_alarms`). The LLM plans tool calls; results are summarised. **No raw SQL generation** — only whitelisted, tenant-scoped tool functions, so RLS and Part 11 access controls hold.
- `POST /v1/ai/ask` `{question}` → `{answer, citations:[{item_id, metric}]}`. Self-hosted deployments point LiteLLM at a local model for data isolation.

**Testing**:
- `Integration (mocked LLM): "open alarms today?" → calls top_alarms tool, answer cites returned alarms`.
- `Integration: question requesting another tenant's data → tools return nothing (RLS holds)`.
- `Unit: tool dispatch rejects any non-whitelisted tool name`.

#### 10.2 — LLM chart interpretation & root-cause hypotheses

**What**: Per-chart plain-language interpretation with assignable-cause hypotheses.

**Design**:
- `POST /v1/items/{id}/interpret` builds a structured prompt from the chart payload (points, limits, violations, recent causes) and returns trend description + ranked root-cause hypotheses drawn from the tenant's historical `alarm_cause` patterns (retrieval over past resolved alarms).
- Prompt template includes AIAG terminology and instructs the model to ground hypotheses in supplied data only.

**Testing**:
- `Integration (mocked LLM): trending chart → interpretation mentions the trend and proposes a hypothesis from history`.
- `Unit: prompt builder includes violations and excludes other tenants' causes`.

#### 10.3 — Predictive drift detection

**What**: ML early-warning before Western Electric/Nelson rules fire.

**Design**:
- EWMA + LSTM ensemble (`ai/drift.py`) trained per characteristic on historical subgroups; outputs a drift-probability score and projected time-to-violation. Inference runs as a Celery task on each new subgroup; high scores raise a `predictive` severity alarm distinct from rule alarms.
- Model registry table tracks trained models per `control_plan_item_id` with metrics.

**Testing**:
- `Unit: EWMA detector flags a gradual mean shift before the rule engine does (on synthetic drift series)`.
- `Integration: high drift score → predictive alarm raised and streamed`.
- `Unit: insufficient history → training skipped with clear status, no crash`.

#### 10.4 — Assisted control-plan generation

**What**: Draft control plans from characteristics + (optional) FMEA risk + historical Cpk.

**Design**:
- `POST /v1/parts/{id}/suggest-control-plan` → recommends chart type per characteristic (variable vs attribute, subgroup size), sampling frequency scaled by special-characteristic class / FMEA severity, and reaction plans, returned as an editable draft plan. Recommendations are deterministic-rule-based with optional LLM phrasing of reaction plans.

**Testing**:
- `Unit: special/safety characteristic → higher sampling frequency than non-special`.
- `Unit: attribute characteristic → suggests P/NP/C/U, never Xbar-R`.
- `Integration: suggested plan is a valid draft that can be activated unchanged`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Tenancy            ─── required by everything
    │
Phase 2: Identity, RBAC & Audit          ─── requires 1
    │
Phase 3: SPC Statistical Engine (core)   ─── requires 1 (pure; can start alongside 2)
    │
Phase 4: Parts, Plans & Chart Config     ─── requires 2, 3
    │
Phase 5: Ingestion, Charting & Alarms    ─── requires 4
    │
Phase 6: Alerting, CA, Capability, Reports ─ requires 5
    │
    ├── Phase 7: Frontend Web App         ─── requires 5 (6 for capability screens); parallel with 8
    └── Phase 8: Standards & Open API     ─── requires 5; parallel with 7
         │
Phase 9: IoT Ingestion & MSA (v1.1)      ─── requires 5 (IoT) and 3 (MSA)
    │
Phase 10: AI-Native Capabilities         ─── requires 5/6 (analytics, interpretation, drift) and 4 (plan gen)
```

**Parallelism opportunities**
- Phase 3 (pure statistical engine, no I/O) can be built concurrently with Phase 2.
- Phase 7 (frontend) and Phase 8 (open API / exchange) can proceed concurrently once Phase 5 is done.
- Within Phase 10, the four tasks are independent and can be parallelised once their dependencies exist.

**MVP boundary**: Phases 1–7 constitute the MVP from `features.md`. Phase 8 delivers the open-API/exchange differentiator. Phase 9 is v1.1. Phase 10 is the AI-native backlog.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented and merged.
2. All unit and integration tests pass; SPC-algorithm phases additionally pass the ISO/TR 11462-3 reference-dataset fixtures.
3. `ruff` + `mypy` (backend) and `eslint` + `tsc` (frontend) pass with zero errors.
4. New Alembic migration(s) created, and `alembic upgrade head` then `downgrade` runs cleanly.
5. New/changed endpoints appear in the auto-generated OpenAPI spec with documented request/response schemas.
6. `docker compose up` brings the stack to a healthy `/healthz`.
7. The phase's feature works end-to-end (verified by the named integration/E2E tests).
8. Any compliance-relevant change (data recording, edits, approvals) produces correct `audit_log` entries and, where required, electronic signatures.
9. New configuration options documented in `.env.example` and README.
10. New tenant-scoped tables have RLS policies enabled and a cross-tenant isolation test.
```
