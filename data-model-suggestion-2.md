# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Statistical Process Control · Created: 2026-05-22

## Philosophy

This model treats every action in the SPC system as an immutable event recorded in an append-only event store. The event store is the single source of truth: measurements recorded, alarms triggered, causes assigned, control limits recalculated, corrective actions taken. Materialised read models (projections) are built from the event stream to serve dashboards, control charts, and capability reports. The architecture follows the CQRS (Command Query Responsibility Segregation) pattern — writes go to the event store, reads come from optimised projections.

This approach is a natural fit for SPC in regulated industries because FDA 21 CFR Part 11 requires that "record changes shall not obscure previously recorded information" and demands "secure, computer-generated, time-stamped audit trails." An event-sourced system provides this by design: the audit trail is not a secondary log bolted onto a mutable database — it IS the database. Every state the system has ever been in can be reconstructed by replaying events up to any point in time. This is particularly valuable for SPC because process investigations often require answering temporal questions: "what were the control limits when this alarm fired?", "what was the Cpk trend for this characteristic over the last 6 months?", "who changed the sampling plan and when?"

Event sourcing is used in financial systems (banking ledgers, trading platforms), healthcare systems (clinical event records), and aviation (flight data recorders). In manufacturing, the ISA-95 Quality Operations Management model already conceptualises quality events (inspection events, test events, deviation events) as discrete occurrences — this model extends that concept to encompass the entire SPC workflow.

**Best for:** FDA-regulated environments (pharma, medical devices) where immutable audit trails are mandatory; organisations that need full temporal query capability ("what was true on date X?"); systems planning AI/ML analytics on process change patterns.

**Trade-offs:**
- (+) Audit trail is inherent — no separate audit log table needed; the event store IS the audit trail
- (+) Full temporal reconstruction: replay events to any point in time for investigation
- (+) Naturally supports bi-temporal queries (event time vs. recording time)
- (+) Event stream is ideal input for ML models analysing process drift patterns
- (+) Schema evolution is easier: new event types can be added without migrating existing data
- (-) Higher storage requirements — events are never deleted, only appended
- (-) Read model complexity — projections must be maintained and can fall out of sync
- (-) Eventual consistency between write (event store) and read (projections) adds application complexity
- (-) Developers unfamiliar with CQRS face a steeper learning curve
- (-) Simple queries ("show me the current Cpk") require reading from projections, not the event store directly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDA 21 CFR Part 11 | The append-only event store inherently satisfies "record changes shall not obscure previously recorded information"; each event has user_id, timestamp, and meaning |
| IATF 16949 | Control plan changes, limit recalculations, and alarm responses are all traceable events with full causal chain |
| ISA-95 Quality Operations Management | Quality events (inspection, test, deviation) map directly to SPC event types in the store |
| ISO 7870-2 (Shewhart Charts) | Chart type configurations stored as events; chart projections built from event replay |
| ISO 22514 (Capability Indices) | Capability calculations stored as CapabilityStudyCompleted events with all inputs preserved |
| AIAG-VDA SPC Yellow Volume 2026 | Terminology in event names and payload fields follows the harmonised manual |
| AQDEF K-fields | Measurement event payloads include AQDEF-aligned field names for export compatibility |
| OPC UA / MQTT | DataSourceConfigured events capture connection details; MeasurementReceived events carry source metadata |

---

## Event Store (Core)

```sql
-- The single append-only event store. This is the source of truth.
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,  -- aggregate type: 'ControlPlanItem', 'Alarm', 'CapabilityStudy', etc.
    stream_id       UUID NOT NULL,          -- aggregate instance ID
    event_type      VARCHAR(150) NOT NULL,  -- e.g., 'MeasurementRecorded', 'AlarmTriggered', 'ControlLimitsRecalculated'
    event_version   INT NOT NULL,           -- monotonically increasing per stream
    payload         JSONB NOT NULL,         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"user_id": "...", "ip_address": "...", "source": "api", "correlation_id": "..."}
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the event was persisted
    occurred_at     TIMESTAMPTZ NOT NULL,                -- when the event actually happened (bi-temporal)
    
    UNIQUE (stream_id, event_version)       -- optimistic concurrency control
);

-- Critical indexes for event replay and temporal queries
CREATE INDEX idx_es_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_es_tenant_type ON event_store(tenant_id, stream_type);
CREATE INDEX idx_es_type_time ON event_store(event_type, occurred_at);
CREATE INDEX idx_es_tenant_time ON event_store(tenant_id, occurred_at);

-- Partition by month for storage management and query performance
-- In production, use declarative partitioning:
-- CREATE TABLE event_store (...) PARTITION BY RANGE (occurred_at);
```

## Event Type Catalogue

The following event types are recorded in the event store. Each has a defined payload schema.

```sql
-- Reference table documenting valid event types and their JSON schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(150) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,         -- JSON Schema for validation
    version         INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads:
--
-- MeasurementRecorded (stream: ControlPlanItem)
-- {
--   "subgroup_number": 142,
--   "samples": [10.023, 10.018, 10.031, 10.027, 10.022],
--   "mean": 10.0242,
--   "range": 0.013,
--   "characteristic_id": "uuid",
--   "station_id": "uuid",
--   "operator_id": "uuid",
--   "data_source": "mqtt",
--   "lot_number": "LOT-2026-0547",
--   "gage_id": "uuid"
-- }
--
-- AttributeMeasurementRecorded (stream: ControlPlanItem)
-- {
--   "subgroup_number": 88,
--   "inspected_count": 200,
--   "defective_count": 3,
--   "defect_count": null,
--   "operator_id": "uuid",
--   "station_id": "uuid"
-- }
--
-- AlarmTriggered (stream: Alarm)
-- {
--   "control_plan_item_id": "uuid",
--   "subgroup_id": "uuid",
--   "rule_name": "One point beyond 3-sigma",
--   "rule_number": 1,
--   "severity": "alarm",
--   "violating_value": 10.089,
--   "ucl": 10.065,
--   "lcl": 9.935,
--   "center_line": 10.000
-- }
--
-- AlarmAcknowledged (stream: Alarm)
-- {
--   "acknowledged_by": "uuid",
--   "notes": "Investigating tool wear on Station 3"
-- }
--
-- AssignableCauseRecorded (stream: Alarm)
-- {
--   "cause_code": "M02",
--   "cause_category": "machine",
--   "cause_name": "Tool wear beyond tolerance",
--   "recorded_by": "uuid"
-- }
--
-- CorrectiveActionCreated (stream: Alarm)
-- {
--   "action_type": "corrective",
--   "description": "Replace cutting insert on Station 3, CNC-07",
--   "assigned_to": "uuid",
--   "due_date": "2026-05-25"
-- }
--
-- ControlLimitsRecalculated (stream: ControlPlanItem)
-- {
--   "phase": "II",
--   "ucl": 10.065,
--   "lcl": 9.935,
--   "center_line": 10.000,
--   "ucl_range": 0.028,
--   "lcl_range": 0.000,
--   "center_line_range": 0.013,
--   "subgroup_count_used": 25,
--   "reason": "Phase II limits established from 25-subgroup trial",
--   "calculated_by": "uuid"
-- }
--
-- CapabilityStudyCompleted (stream: ControlPlanItem)
-- {
--   "study_type": "short_term",
--   "cp": 1.67,
--   "cpk": 1.52,
--   "pp": 1.45,
--   "ppk": 1.38,
--   "process_mean": 10.002,
--   "process_stddev": 0.0089,
--   "sample_count": 125,
--   "from_date": "2026-05-01T00:00:00Z",
--   "to_date": "2026-05-15T23:59:59Z"
-- }
--
-- ControlPlanCreated (stream: ControlPlan)
-- {
--   "plan_number": "CP-2026-0042",
--   "part_id": "uuid",
--   "part_number": "P-10042",
--   "items": [{"characteristic_id": "uuid", "chart_type": "xbar_r", "subgroup_size": 5, "sampling_frequency": "every 30 minutes"}]
-- }
--
-- ControlPlanApproved (stream: ControlPlan)
-- {
--   "approved_by": "uuid",
--   "electronic_signature_meaning": "I approve this control plan for production use",
--   "effective_date": "2026-05-20"
-- }
--
-- PartDefined (stream: Part)
-- {
--   "part_number": "P-10042",
--   "part_name": "Crankshaft Bearing Cap",
--   "revision": "C",
--   "characteristics": [{"number": 1, "name": "Bore Diameter", "usl": 50.025, "lsl": 49.975, "nominal": 50.000, "unit": "mm"}]
-- }
--
-- GageCalibrationRecorded (stream: Gage)
-- {
--   "gage_id_number": "G-0047",
--   "calibration_date": "2026-05-20",
--   "next_due_date": "2026-08-20",
--   "result": "pass",
--   "calibrated_by": "uuid",
--   "certificate_number": "CAL-2026-1847"
-- }
```

## Reference Data (Non-Event Tables)

Some data is truly reference/configuration and does not benefit from event sourcing. These are standard relational tables.

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    plant_id        UUID,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

-- SPC statistical constants (read-only reference)
CREATE TABLE spc_constants (
    subgroup_size   INT PRIMARY KEY,
    a2              NUMERIC(10,6) NOT NULL,
    a3              NUMERIC(10,6) NOT NULL,
    b3              NUMERIC(10,6) NOT NULL,
    b4              NUMERIC(10,6) NOT NULL,
    c4              NUMERIC(10,6) NOT NULL,
    d2              NUMERIC(10,6) NOT NULL,
    d3              NUMERIC(10,6) NOT NULL,
    d4              NUMERIC(10,6) NOT NULL
);

-- Pre-populate with standard AIAG values
INSERT INTO spc_constants (subgroup_size, a2, a3, b3, b4, c4, d2, d3, d4) VALUES
    (2, 1.880000, 2.659000, 0.000000, 3.267000, 0.797900, 1.128000, 0.000000, 3.267000),
    (3, 1.023000, 1.954000, 0.000000, 2.568000, 0.886200, 1.693000, 0.000000, 2.574000),
    (4, 0.729000, 1.628000, 0.000000, 2.266000, 0.921300, 2.059000, 0.000000, 2.282000),
    (5, 0.577000, 1.427000, 0.000000, 2.089000, 0.940000, 2.326000, 0.000000, 2.114000),
    (6, 0.483000, 1.287000, 0.030000, 1.970000, 0.951500, 2.534000, 0.000000, 2.004000),
    (7, 0.419000, 1.182000, 0.118000, 1.882000, 0.959400, 2.704000, 0.076000, 1.924000),
    (8, 0.373000, 1.099000, 0.185000, 1.815000, 0.965000, 2.847000, 0.136000, 1.864000),
    (9, 0.337000, 1.032000, 0.239000, 1.761000, 0.969300, 2.970000, 0.184000, 1.816000),
    (10, 0.308000, 0.975000, 0.284000, 1.716000, 0.972700, 3.078000, 0.223000, 1.777000);

-- Assignable cause library (reference data, not event-sourced)
CREATE TABLE assignable_cause_library (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),           -- 5M categories
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, code)
);
```

## Materialised Read Models (Projections)

These tables are rebuilt from the event store. They can be dropped and reconstructed at any time.

```sql
-- ============================================================
-- PROJECTION: Current state of all control plan items and their charts
-- Built from: ControlPlanCreated, ControlLimitsRecalculated, ControlPlanApproved events
-- ============================================================
CREATE TABLE proj_control_plan_item (
    id                  UUID PRIMARY KEY,   -- stream_id from ControlPlanItem stream
    tenant_id           UUID NOT NULL,
    control_plan_id     UUID NOT NULL,
    plan_number         VARCHAR(100) NOT NULL,
    part_id             UUID NOT NULL,
    part_number         VARCHAR(100) NOT NULL,
    characteristic_id   UUID NOT NULL,
    characteristic_name VARCHAR(255) NOT NULL,
    chart_type          VARCHAR(30) NOT NULL,
    subgroup_size       INT NOT NULL,
    sampling_frequency  VARCHAR(100),
    station_id          UUID,
    -- Current control limits (from latest ControlLimitsRecalculated event)
    current_ucl         NUMERIC(20,6),
    current_lcl         NUMERIC(20,6),
    current_center_line NUMERIC(20,6),
    current_ucl_range   NUMERIC(20,6),
    current_lcl_range   NUMERIC(20,6),
    current_center_line_range NUMERIC(20,6),
    limits_phase        VARCHAR(10),
    limits_updated_at   TIMESTAMPTZ,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',
    last_event_version  INT NOT NULL,       -- for idempotent projection updates
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Latest subgroup data for each control plan item
-- Built from: MeasurementRecorded events
-- ============================================================
CREATE TABLE proj_subgroup (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    control_plan_item_id UUID NOT NULL,
    subgroup_number     BIGINT NOT NULL,
    occurred_at         TIMESTAMPTZ NOT NULL,
    mean_value          NUMERIC(20,6),
    range_value         NUMERIC(20,6),
    std_dev_value       NUMERIC(20,6),
    sample_count        INT NOT NULL,
    samples             JSONB NOT NULL,     -- array of individual values
    operator_id         UUID,
    station_id          UUID,
    data_source         VARCHAR(50),
    event_id            UUID NOT NULL,      -- back-reference to event_store
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_sg_cpi_time ON proj_subgroup(control_plan_item_id, occurred_at);
CREATE INDEX idx_proj_sg_tenant ON proj_subgroup(tenant_id);

-- ============================================================
-- PROJECTION: Current alarm state
-- Built from: AlarmTriggered, AlarmAcknowledged, AssignableCauseRecorded,
--             CorrectiveActionCreated, CorrectiveActionCompleted events
-- ============================================================
CREATE TABLE proj_alarm (
    id                  UUID PRIMARY KEY,   -- stream_id from Alarm stream
    tenant_id           UUID NOT NULL,
    control_plan_item_id UUID NOT NULL,
    subgroup_id         UUID,
    rule_name           VARCHAR(255) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    triggered_at        TIMESTAMPTZ NOT NULL,
    acknowledged_by     UUID,
    acknowledged_at     TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    cause_codes         JSONB DEFAULT '[]', -- array of assigned cause codes
    corrective_actions  JSONB DEFAULT '[]', -- array of corrective action summaries
    last_event_version  INT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_alarm_cpi ON proj_alarm(control_plan_item_id);
CREATE INDEX idx_proj_alarm_status ON proj_alarm(status);
CREATE INDEX idx_proj_alarm_tenant ON proj_alarm(tenant_id);

-- ============================================================
-- PROJECTION: Latest capability indices per control plan item
-- Built from: CapabilityStudyCompleted events
-- ============================================================
CREATE TABLE proj_capability (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    control_plan_item_id UUID NOT NULL,
    study_type          VARCHAR(30) NOT NULL,
    cp                  NUMERIC(10,4),
    cpk                 NUMERIC(10,4),
    pp                  NUMERIC(10,4),
    ppk                 NUMERIC(10,4),
    process_mean        NUMERIC(20,6),
    process_stddev      NUMERIC(20,6),
    calculated_at       TIMESTAMPTZ NOT NULL,
    event_id            UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_cap_cpi ON proj_capability(control_plan_item_id);

-- ============================================================
-- PROJECTION: Current part and characteristic definitions
-- Built from: PartDefined, CharacteristicUpdated events
-- ============================================================
CREATE TABLE proj_part (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    part_number         VARCHAR(100) NOT NULL,
    part_name           VARCHAR(255) NOT NULL,
    revision            VARCHAR(50),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_event_version  INT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_characteristic (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    part_id             UUID NOT NULL,
    characteristic_number INT NOT NULL,
    name                VARCHAR(255) NOT NULL,
    data_type           VARCHAR(20) NOT NULL,
    unit_of_measure     VARCHAR(50),
    nominal_value       NUMERIC(20,6),
    upper_spec_limit    NUMERIC(20,6),
    lower_spec_limit    NUMERIC(20,6),
    is_special_char     BOOLEAN NOT NULL DEFAULT false,
    last_event_version  INT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Plant / equipment hierarchy
-- Built from: PlantCreated, AreaCreated, StationCreated events
-- ============================================================
CREATE TABLE proj_plant (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_station (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    plant_id        UUID NOT NULL,
    area_id         UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    station_type    VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Snapshot Store (Performance Optimisation)

```sql
-- Periodic snapshots of aggregate state to avoid replaying entire event history
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    event_version   INT NOT NULL,           -- version at which snapshot was taken
    state           JSONB NOT NULL,         -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, event_version DESC);
```

## Example Queries

### Replay all events for a control plan item (temporal reconstruction)
```sql
-- "What happened to this control plan item from May 1 to May 15?"
SELECT event_type, payload, occurred_at, metadata->>'user_id' AS user_id
FROM event_store
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND occurred_at BETWEEN '2026-05-01' AND '2026-05-15'
ORDER BY event_version;
```

### Get current control chart data from projection
```sql
-- "Show me the last 50 subgroups for this chart"
SELECT subgroup_number, occurred_at, mean_value, range_value, sample_count
FROM proj_subgroup
WHERE control_plan_item_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY occurred_at DESC
LIMIT 50;
```

### Investigate an alarm's full history
```sql
-- "Show me everything that happened after alarm X was triggered"
SELECT event_type, payload, occurred_at,
       metadata->>'user_id' AS actor
FROM event_store
WHERE stream_id = 'alarm-uuid-here'
ORDER BY event_version;
-- Returns: AlarmTriggered → AlarmAcknowledged → AssignableCauseRecorded → CorrectiveActionCreated → CorrectiveActionCompleted → AlarmResolved
```

### Bi-temporal query (what did we know and when?)
```sql
-- "What control limits were in effect on May 10 at 2pm?"
SELECT payload
FROM event_store
WHERE stream_id = 'cpi-uuid-here'
  AND event_type = 'ControlLimitsRecalculated'
  AND occurred_at <= '2026-05-10 14:00:00+00'
ORDER BY event_version DESC
LIMIT 1;
```

### Feed ML model with process change events
```sql
-- "Get all measurement and alarm events for ML training dataset"
SELECT event_type, payload, occurred_at
FROM event_store
WHERE tenant_id = 'tenant-uuid'
  AND event_type IN ('MeasurementRecorded', 'AlarmTriggered', 'ControlLimitsRecalculated')
  AND occurred_at >= '2026-01-01'
ORDER BY occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (core) | 2 | event_store, event_type_registry |
| Reference Data | 5 | tenant, app_user, role, user_role, assignable_cause_library |
| SPC Constants | 1 | spc_constants |
| Projections | 8 | proj_control_plan_item, proj_subgroup, proj_alarm, proj_capability, proj_part, proj_characteristic, proj_plant, proj_station |
| Snapshots | 1 | event_snapshot |
| **Total** | **17** | Projections are rebuildable from event store |

---

## Key Design Decisions

1. **Single event store table** — all domain events share one table with a JSONB payload. This simplifies event replay, subscription, and projection rebuilding. The event_type column enables filtering without parsing JSON.

2. **Bi-temporal timestamps** (`occurred_at` vs. `recorded_at`) — `occurred_at` is when the event actually happened (e.g., when the measurement was taken on the shop floor); `recorded_at` is when the event was persisted to the database. This distinction is critical for manufacturing SPC where measurements may be batched or delayed in transit.

3. **Optimistic concurrency via (stream_id, event_version) UNIQUE constraint** — prevents concurrent writes from corrupting aggregate state. Two operators recording measurements for the same chart item simultaneously will have their events serialised correctly.

4. **Projections are disposable and rebuildable** — if a projection becomes corrupt or a bug is discovered, the projection table can be dropped and rebuilt by replaying events from the event store. The `last_event_version` column enables incremental rebuilds.

5. **Event type registry** — a reference table documenting all valid event types and their JSON Schema payloads. This enables runtime validation of events before they are persisted and serves as living documentation of the event catalogue.

6. **Snapshots for performance** — rather than replaying thousands of events for long-lived aggregates (a control plan item may accumulate millions of measurement events), periodic snapshots capture aggregate state at a point in time. Replay starts from the latest snapshot.

7. **Reference data remains relational** — user accounts, roles, SPC constants, and assignable cause libraries are not event-sourced because they are true reference data with simple CRUD semantics. Over-applying event sourcing to everything adds complexity without benefit.

8. **FDA 21 CFR Part 11 compliance by construction** — every event immutably records who (metadata.user_id), what (event_type + payload), when (occurred_at + recorded_at), and why (payload contains reason/meaning fields). Electronic signatures are captured as ControlPlanApproved events with signature meaning in the payload.

9. **ML-friendly event stream** — the event store can be directly queried by ML pipelines looking for patterns in process changes. A time-ordered stream of MeasurementRecorded events followed by AlarmTriggered events is the natural training data format for predictive drift detection models.

10. **Event partitioning by occurred_at** — in production, the event_store table should use declarative range partitioning by month on occurred_at. This enables efficient retention policies (archive old partitions) and fast time-range queries without full table scans.
