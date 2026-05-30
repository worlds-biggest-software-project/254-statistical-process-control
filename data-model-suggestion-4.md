# Data Model Suggestion 4: Time-Series / Analytics-First

> Project: Statistical Process Control · Created: 2026-05-22

## Philosophy

This model is designed around the insight that SPC is fundamentally a time-series problem: measurements flow in continuously from production equipment, and the primary query pattern is "get measurements for characteristic X in time range Y, compute statistics, and detect rule violations." Rather than treating time-series measurement data with a general-purpose relational schema, this model uses PostgreSQL with the TimescaleDB extension to partition measurement data into time-based chunks (hypertables), enabling high-throughput ingestion, fast time-range queries, automatic data compression, and built-in retention policies.

The architecture splits the schema into two tiers: a **configuration tier** (standard relational tables for parts, characteristics, control plans, users) and a **data tier** (TimescaleDB hypertables for measurements, alarms, and audit events). The configuration tier changes infrequently and benefits from standard relational indexing; the data tier grows continuously and benefits from time-partitioned storage, chunk exclusion, and continuous aggregates.

This pattern is established in industrial IoT platforms: TimescaleDB is used by manufacturing data pipelines to handle sensor streams from PLCs and CNCs at hundreds of thousands of inserts per second. For SPC, the time-series approach means that a factory with 100 stations each recording 5-sample subgroups every 30 minutes (240,000 measurements per day) can be ingested, queried, and charted without the performance degradation that affects standard PostgreSQL tables at scale. Continuous aggregates pre-compute hourly and daily summary statistics, enabling instant dashboard rendering for enterprise-wide views.

**Best for:** High-volume manufacturing environments with many production lines and IoT data feeds; organisations prioritising real-time dashboard performance and long-term data retention; deployments expecting measurement volumes in the hundreds of millions to billions of rows.

**Trade-offs:**
- (+) TimescaleDB hypertables maintain consistent query performance regardless of total data volume
- (+) Ingestion throughput of hundreds of thousands of measurements per second via bulk COPY or batch INSERT
- (+) Continuous aggregates pre-compute hourly/daily/weekly statistics for instant dashboard rendering
- (+) Built-in compression (10-20x) and retention policies manage storage automatically
- (+) Chunk exclusion eliminates irrelevant time partitions from queries, accelerating time-range scans by 10-100x
- (+) Native PostgreSQL — no separate database technology to learn or operate
- (-) Requires TimescaleDB extension — not available on all managed PostgreSQL providers
- (-) Hypertables have constraints: no foreign key references TO hypertables, updates on compressed chunks require decompression
- (-) Continuous aggregates add configuration complexity and can lag real-time by the refresh interval
- (-) Schema design is optimised for time-series access patterns — ad-hoc cross-entity joins are possible but not the primary strength
- (-) TimescaleDB community edition lacks some enterprise features (multi-node, continuous aggregate policies on compressed data)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 7870-2 (Shewhart Charts) | Chart types enumerated in configuration; measurement hypertable stores raw data for chart computation |
| ISO 22514 (Capability Indices) | Continuous aggregates pre-compute statistics needed for capability index calculation |
| AIAG-VDA SPC Yellow Volume 2026 | Terminology in column names and chart type enumerations follows the harmonised manual |
| AQDEF K-fields | Part and characteristic configuration tables include AQDEF K-field references |
| FDA 21 CFR Part 11 | Audit event hypertable provides immutable, time-partitioned audit trail |
| ISA-95 / IEC 62264 | Equipment hierarchy in configuration tier; station_id is the primary dimensional key in measurement hypertable |
| OPC UA / MQTT / ISO 20922 | Data ingestion layer writes directly to measurement hypertable; source metadata captured per measurement |
| ISO 3166-1 | Plant country codes for multi-national deployments |

---

## Configuration Tier (Standard Relational Tables)

```sql
-- ============================================================
-- TENANT & ORGANISATION (standard PostgreSQL)
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE plant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plant_id        UUID NOT NULL REFERENCES plant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    parent_station_id UUID REFERENCES station(id),
    station_type    VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plant_tenant ON plant(tenant_id);
CREATE INDEX idx_station_plant ON station(plant_id);

-- ============================================================
-- USERS & ACCESS
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    role            VARCHAR(30) NOT NULL DEFAULT 'operator',
    plant_access    UUID[] DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- ============================================================
-- PART & CHARACTERISTIC DEFINITIONS
-- ============================================================

CREATE TABLE part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,  -- AQDEF K1001
    part_name       VARCHAR(255) NOT NULL,  -- AQDEF K1002
    revision        VARCHAR(50),
    metadata        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number, revision)
);

CREATE TABLE characteristic (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id             UUID NOT NULL REFERENCES part(id),
    number              INT NOT NULL,       -- AQDEF K2001
    name                VARCHAR(255) NOT NULL,
    data_type           VARCHAR(20) NOT NULL DEFAULT 'variable',
    unit_of_measure     VARCHAR(50),
    nominal_value       NUMERIC(20,6),      -- AQDEF K2101
    upper_spec_limit    NUMERIC(20,6),      -- AQDEF K2111
    lower_spec_limit    NUMERIC(20,6),      -- AQDEF K2110
    is_special_char     BOOLEAN NOT NULL DEFAULT false,
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (part_id, number)
);

CREATE INDEX idx_part_tenant ON part(tenant_id);
CREATE INDEX idx_char_part ON characteristic(part_id);

-- ============================================================
-- CONTROL PLAN & CHART CONFIGURATION
-- ============================================================

CREATE TABLE control_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    plan_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    part_id         UUID NOT NULL REFERENCES part(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    effective_date  DATE,
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chart_config (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_id     UUID NOT NULL REFERENCES control_plan(id),
    characteristic_id   UUID NOT NULL REFERENCES characteristic(id),
    station_id          UUID REFERENCES station(id),
    chart_type          VARCHAR(30) NOT NULL,
    subgroup_size       INT NOT NULL DEFAULT 5,
    sampling_frequency  VARCHAR(100),
    -- Current control limits
    ucl                 NUMERIC(20,6),
    lcl                 NUMERIC(20,6),
    center_line         NUMERIC(20,6),
    ucl_secondary       NUMERIC(20,6),
    lcl_secondary       NUMERIC(20,6),
    cl_secondary        NUMERIC(20,6),
    phase               VARCHAR(10) NOT NULL DEFAULT 'I',
    rule_set            VARCHAR(30) NOT NULL DEFAULT 'nelson',
    chart_parameters    JSONB NOT NULL DEFAULT '{}',
    reaction_plan       TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cp_tenant ON control_plan(tenant_id);
CREATE INDEX idx_cc_plan ON chart_config(control_plan_id);
CREATE INDEX idx_cc_char ON chart_config(characteristic_id);

-- ============================================================
-- GAGE MANAGEMENT
-- ============================================================

CREATE TABLE gage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    gage_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    resolution      NUMERIC(20,8),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    calibration_due DATE,
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, gage_number)
);

-- ============================================================
-- DATA SOURCE CONFIGURATION
-- ============================================================

CREATE TABLE data_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES station(id),
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(30) NOT NULL,
    config          JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_received_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SPC STATISTICAL CONSTANTS
-- ============================================================

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
```

## Data Tier (TimescaleDB Hypertables)

```sql
-- ============================================================
-- MEASUREMENT HYPERTABLE — the core time-series table
-- Stores individual sample measurements with dimensional context
-- ============================================================

CREATE TABLE measurement (
    time                TIMESTAMPTZ NOT NULL,          -- when the measurement was taken
    tenant_id           UUID NOT NULL,
    chart_config_id     UUID NOT NULL,                 -- FK to chart_config (enforced in app layer)
    characteristic_id   UUID NOT NULL,                 -- denormalised for query performance
    station_id          UUID,                          -- denormalised for query performance
    subgroup_number     BIGINT NOT NULL,
    sample_number       INT NOT NULL,                  -- position within subgroup (1..n)
    value               DOUBLE PRECISION NOT NULL,     -- measured value (double for time-series perf)
    operator_id         UUID,
    data_source         VARCHAR(50) NOT NULL DEFAULT 'manual',
    lot_number          VARCHAR(100),
    batch_number        VARCHAR(100),
    gage_id             UUID,
    context             JSONB                          -- additional contextual metadata
);

-- Convert to TimescaleDB hypertable, partitioned by time (1-week chunks)
SELECT create_hypertable('measurement', 'time',
    chunk_time_interval => INTERVAL '1 week',
    if_not_exists => TRUE
);

-- Add space partitioning by tenant for multi-tenant isolation
SELECT add_dimension('measurement', 'tenant_id', number_partitions => 4);

-- Indexes optimised for SPC query patterns
CREATE INDEX idx_meas_chart_time ON measurement (chart_config_id, time DESC);
CREATE INDEX idx_meas_char_time ON measurement (characteristic_id, time DESC);
CREATE INDEX idx_meas_tenant_time ON measurement (tenant_id, time DESC);
CREATE INDEX idx_meas_subgroup ON measurement (chart_config_id, subgroup_number);
CREATE INDEX idx_meas_lot ON measurement (lot_number, time DESC) WHERE lot_number IS NOT NULL;

-- ============================================================
-- SUBGROUP SUMMARY HYPERTABLE
-- Pre-computed subgroup statistics for fast chart rendering
-- Written by the ingestion pipeline after each complete subgroup
-- ============================================================

CREATE TABLE subgroup_summary (
    time                TIMESTAMPTZ NOT NULL,          -- subgroup collection time
    tenant_id           UUID NOT NULL,
    chart_config_id     UUID NOT NULL,
    characteristic_id   UUID NOT NULL,
    station_id          UUID,
    subgroup_number     BIGINT NOT NULL,
    sample_count        INT NOT NULL,
    mean_value          DOUBLE PRECISION,              -- Xbar
    range_value         DOUBLE PRECISION,              -- R
    std_dev_value       DOUBLE PRECISION,              -- S
    min_value           DOUBLE PRECISION,
    max_value           DOUBLE PRECISION,
    median_value        DOUBLE PRECISION,
    operator_id         UUID,
    data_source         VARCHAR(50) NOT NULL,
    lot_number          VARCHAR(100)
);

SELECT create_hypertable('subgroup_summary', 'time',
    chunk_time_interval => INTERVAL '1 week',
    if_not_exists => TRUE
);

CREATE INDEX idx_ss_chart_time ON subgroup_summary (chart_config_id, time DESC);
CREATE INDEX idx_ss_tenant_time ON subgroup_summary (tenant_id, time DESC);

-- ============================================================
-- ATTRIBUTE MEASUREMENT HYPERTABLE
-- For P, NP, C, U charts
-- ============================================================

CREATE TABLE attribute_measurement (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    chart_config_id     UUID NOT NULL,
    characteristic_id   UUID NOT NULL,
    station_id          UUID,
    subgroup_number     BIGINT NOT NULL,
    inspected_count     INT NOT NULL,
    defective_count     INT,
    defect_count        INT,
    operator_id         UUID,
    data_source         VARCHAR(50) NOT NULL DEFAULT 'manual',
    lot_number          VARCHAR(100),
    defect_details      JSONB
);

SELECT create_hypertable('attribute_measurement', 'time',
    chunk_time_interval => INTERVAL '1 week',
    if_not_exists => TRUE
);

CREATE INDEX idx_am_chart_time ON attribute_measurement (chart_config_id, time DESC);

-- ============================================================
-- ALARM HYPERTABLE
-- Alarms as time-series events for trend analysis
-- ============================================================

CREATE TABLE alarm (
    time                TIMESTAMPTZ NOT NULL,          -- when alarm was triggered
    tenant_id           UUID NOT NULL,
    alarm_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    chart_config_id     UUID NOT NULL,
    characteristic_id   UUID NOT NULL,
    station_id          UUID,
    rule_violated       VARCHAR(100) NOT NULL,
    severity            VARCHAR(20) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    violating_value     DOUBLE PRECISION,
    ucl_at_time         DOUBLE PRECISION,
    lcl_at_time         DOUBLE PRECISION,
    center_line_at_time DOUBLE PRECISION,
    subgroup_number     BIGINT,
    acknowledged_by     UUID,
    acknowledged_at     TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    causes              JSONB DEFAULT '[]',
    corrective_actions  JSONB DEFAULT '[]',
    notes               TEXT
);

SELECT create_hypertable('alarm', 'time',
    chunk_time_interval => INTERVAL '1 month',
    if_not_exists => TRUE
);

CREATE INDEX idx_alarm_chart_time ON alarm (chart_config_id, time DESC);
CREATE INDEX idx_alarm_tenant_status ON alarm (tenant_id, status, time DESC);
CREATE UNIQUE INDEX idx_alarm_id ON alarm (alarm_id, time);

-- ============================================================
-- AUDIT EVENT HYPERTABLE
-- Immutable audit trail as time-series
-- ============================================================

CREATE TABLE audit_event (
    time            TIMESTAMPTZ NOT NULL DEFAULT now(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    reason          TEXT,
    ip_address      INET
);

SELECT create_hypertable('audit_event', 'time',
    chunk_time_interval => INTERVAL '1 month',
    if_not_exists => TRUE
);

CREATE INDEX idx_audit_tenant_time ON audit_event (tenant_id, time DESC);
CREATE INDEX idx_audit_entity ON audit_event (entity_type, entity_id, time DESC);
```

## Continuous Aggregates

```sql
-- ============================================================
-- HOURLY SUBGROUP STATISTICS
-- Automatically maintained by TimescaleDB; used for dashboards
-- ============================================================

CREATE MATERIALIZED VIEW subgroup_stats_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    tenant_id,
    chart_config_id,
    characteristic_id,
    station_id,
    COUNT(*) AS subgroup_count,
    AVG(mean_value) AS avg_mean,
    MIN(mean_value) AS min_mean,
    MAX(mean_value) AS max_mean,
    AVG(range_value) AS avg_range,
    MAX(range_value) AS max_range,
    AVG(std_dev_value) AS avg_stddev
FROM subgroup_summary
GROUP BY bucket, tenant_id, chart_config_id, characteristic_id, station_id;

-- Refresh policy: update every 30 minutes, with 2-hour lag
SELECT add_continuous_aggregate_policy('subgroup_stats_hourly',
    start_offset    => INTERVAL '2 hours',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes'
);

-- ============================================================
-- DAILY SUBGROUP STATISTICS
-- For trend analysis and management dashboards
-- ============================================================

CREATE MATERIALIZED VIEW subgroup_stats_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    tenant_id,
    chart_config_id,
    characteristic_id,
    station_id,
    COUNT(*) AS subgroup_count,
    AVG(mean_value) AS avg_mean,
    STDDEV(mean_value) AS stddev_mean,
    MIN(mean_value) AS min_mean,
    MAX(mean_value) AS max_mean,
    AVG(range_value) AS avg_range,
    MAX(range_value) AS max_range,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY mean_value) AS median_mean
FROM subgroup_summary
GROUP BY bucket, tenant_id, chart_config_id, characteristic_id, station_id;

SELECT add_continuous_aggregate_policy('subgroup_stats_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- ============================================================
-- HOURLY ALARM COUNTS
-- For alarm rate monitoring and trend dashboards
-- ============================================================

CREATE MATERIALIZED VIEW alarm_stats_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    tenant_id,
    chart_config_id,
    severity,
    rule_violated,
    COUNT(*) AS alarm_count
FROM alarm
GROUP BY bucket, tenant_id, chart_config_id, severity, rule_violated;

SELECT add_continuous_aggregate_policy('alarm_stats_hourly',
    start_offset    => INTERVAL '2 hours',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes'
);
```

## Data Retention & Compression Policies

```sql
-- ============================================================
-- COMPRESSION: Compress measurement data older than 2 weeks
-- Achieves 10-20x storage reduction on numeric time-series
-- ============================================================

ALTER TABLE measurement SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'chart_config_id, tenant_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('measurement', INTERVAL '2 weeks');

ALTER TABLE subgroup_summary SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'chart_config_id, tenant_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('subgroup_summary', INTERVAL '2 weeks');

ALTER TABLE attribute_measurement SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'chart_config_id, tenant_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('attribute_measurement', INTERVAL '2 weeks');

-- ============================================================
-- RETENTION: Drop raw measurements older than 2 years
-- Aggregated data in continuous aggregates is retained longer
-- ============================================================

SELECT add_retention_policy('measurement', INTERVAL '2 years');

-- Keep subgroup summaries for 5 years (much smaller than raw measurements)
SELECT add_retention_policy('subgroup_summary', INTERVAL '5 years');

-- Keep alarms for 7 years (regulatory retention requirement)
SELECT add_retention_policy('alarm', INTERVAL '7 years');

-- Keep audit events for 10 years (FDA/IATF retention requirements)
SELECT add_retention_policy('audit_event', INTERVAL '10 years');
```

## Example Queries

### Render an Xbar-R control chart (last 50 subgroups)
```sql
-- Fast: hits only the most recent chunk(s), index-driven
SELECT subgroup_number, time, mean_value, range_value, sample_count
FROM subgroup_summary
WHERE chart_config_id = 'chart-uuid'
ORDER BY time DESC
LIMIT 50;
```

### Compute process capability from recent data
```sql
-- Calculate Cp and Cpk from the last 25 subgroups
WITH recent_data AS (
    SELECT mean_value, range_value
    FROM subgroup_summary
    WHERE chart_config_id = 'chart-uuid'
    ORDER BY time DESC
    LIMIT 25
),
stats AS (
    SELECT
        AVG(mean_value) AS x_bar,
        AVG(range_value) AS r_bar,
        COUNT(*) AS n
    FROM recent_data
),
capability AS (
    SELECT
        x_bar,
        r_bar,
        r_bar / sc.d2 AS sigma_hat,
        (c.upper_spec_limit - c.lower_spec_limit) / (6 * (r_bar / sc.d2)) AS cp,
        LEAST(
            (c.upper_spec_limit - x_bar) / (3 * (r_bar / sc.d2)),
            (x_bar - c.lower_spec_limit) / (3 * (r_bar / sc.d2))
        ) AS cpk
    FROM stats
    CROSS JOIN characteristic c
    CROSS JOIN spc_constants sc
    WHERE c.id = 'characteristic-uuid'
      AND sc.subgroup_size = 5
)
SELECT cp, cpk, x_bar, r_bar, sigma_hat FROM capability;
```

### Enterprise dashboard: all charts with recent alarms
```sql
-- Uses continuous aggregate for fast aggregation
SELECT
    cc.id AS chart_config_id,
    p.part_number,
    ch.name AS characteristic_name,
    pl.name AS plant_name,
    st.name AS station_name,
    agg.subgroup_count AS subgroups_today,
    agg.avg_mean,
    agg.max_range,
    COALESCE(alm.alarm_count, 0) AS alarms_today
FROM chart_config cc
JOIN control_plan cp ON cp.id = cc.control_plan_id
JOIN part p ON p.id = cp.part_id
JOIN characteristic ch ON ch.id = cc.characteristic_id
LEFT JOIN station st ON st.id = cc.station_id
LEFT JOIN plant pl ON pl.id = st.plant_id
LEFT JOIN LATERAL (
    SELECT subgroup_count, avg_mean, max_range
    FROM subgroup_stats_daily
    WHERE chart_config_id = cc.id
      AND bucket = date_trunc('day', now())
    LIMIT 1
) agg ON true
LEFT JOIN LATERAL (
    SELECT SUM(alarm_count) AS alarm_count
    FROM alarm_stats_hourly
    WHERE chart_config_id = cc.id
      AND bucket >= date_trunc('day', now())
) alm ON true
WHERE cp.tenant_id = 'tenant-uuid'
  AND cc.is_active = true
ORDER BY alm.alarm_count DESC NULLS LAST;
```

### Cross-plant trend: weekly Xbar averages for 6 months
```sql
-- Uses daily continuous aggregate, time_bucket for weekly rollup
SELECT
    time_bucket('1 week', bucket) AS week,
    pl.name AS plant_name,
    AVG(avg_mean) AS weekly_avg_mean,
    MAX(max_range) AS weekly_max_range,
    SUM(subgroup_count) AS total_subgroups
FROM subgroup_stats_daily sd
JOIN chart_config cc ON cc.id = sd.chart_config_id
JOIN station st ON st.id = sd.station_id
JOIN plant pl ON pl.id = st.plant_id
WHERE sd.characteristic_id = 'characteristic-uuid'
  AND sd.tenant_id = 'tenant-uuid'
  AND bucket >= now() - INTERVAL '6 months'
GROUP BY week, pl.name
ORDER BY week, pl.name;
```

### High-frequency ingestion batch (MQTT pipeline)
```sql
-- Bulk insert from IoT pipeline: multiple subgroups in one INSERT
INSERT INTO measurement (time, tenant_id, chart_config_id, characteristic_id,
                         station_id, subgroup_number, sample_number, value,
                         data_source, lot_number)
VALUES
    ('2026-05-22 14:30:00+00', 'tenant-uuid', 'chart-uuid', 'char-uuid',
     'station-uuid', 5001, 1, 10.023, 'mqtt', 'LOT-2026-0547'),
    ('2026-05-22 14:30:00+00', 'tenant-uuid', 'chart-uuid', 'char-uuid',
     'station-uuid', 5001, 2, 10.018, 'mqtt', 'LOT-2026-0547'),
    ('2026-05-22 14:30:00+00', 'tenant-uuid', 'chart-uuid', 'char-uuid',
     'station-uuid', 5001, 3, 10.031, 'mqtt', 'LOT-2026-0547'),
    ('2026-05-22 14:30:00+00', 'tenant-uuid', 'chart-uuid', 'char-uuid',
     'station-uuid', 5001, 4, 10.027, 'mqtt', 'LOT-2026-0547'),
    ('2026-05-22 14:30:00+00', 'tenant-uuid', 'chart-uuid', 'char-uuid',
     'station-uuid', 5001, 5, 10.022, 'mqtt', 'LOT-2026-0547');

-- Corresponding subgroup summary (written by ingestion pipeline after computing stats)
INSERT INTO subgroup_summary (time, tenant_id, chart_config_id, characteristic_id,
                              station_id, subgroup_number, sample_count,
                              mean_value, range_value, std_dev_value,
                              min_value, max_value, median_value,
                              data_source, lot_number)
VALUES
    ('2026-05-22 14:30:00+00', 'tenant-uuid', 'chart-uuid', 'char-uuid',
     'station-uuid', 5001, 5,
     10.0242, 0.013, 0.00489,
     10.018, 10.031, 10.023,
     'mqtt', 'LOT-2026-0547');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Configuration: Organisation | 3 | tenant, plant, station |
| Configuration: Users | 1 | app_user |
| Configuration: Parts | 2 | part, characteristic |
| Configuration: Control Plans | 2 | control_plan, chart_config |
| Configuration: Gages | 1 | gage |
| Configuration: Data Sources | 1 | data_source |
| Configuration: SPC Constants | 1 | spc_constants |
| **Config subtotal** | **11** | Standard relational tables |
| Data: Hypertables | 4 | measurement, subgroup_summary, attribute_measurement, alarm |
| Data: Audit | 1 | audit_event |
| Data: Continuous Aggregates | 3 | subgroup_stats_hourly, subgroup_stats_daily, alarm_stats_hourly |
| **Data subtotal** | **8** | TimescaleDB hypertables + continuous aggregates |
| **Total** | **19** | 11 config + 4 hypertables + 1 audit hypertable + 3 continuous aggregates |

---

## Key Design Decisions

1. **TimescaleDB hypertables for measurement data** — the measurement table is converted to a hypertable with 1-week chunk intervals. At 240,000 measurements/day (a modest factory), each chunk contains ~1.7M rows, keeping chunk-level operations fast. Chunk exclusion means that "get last 50 subgroups" only scans the most recent 1-2 chunks regardless of total data volume.

2. **Dual measurement tables** — raw individual measurements go into the `measurement` hypertable; pre-computed subgroup summaries go into `subgroup_summary`. Control chart rendering reads from `subgroup_summary` (one row per subgroup) rather than aggregating raw measurements on every chart load. The raw data is retained for drill-down, capability recalculation, and audit purposes.

3. **Denormalised dimensional columns in hypertables** — `tenant_id`, `characteristic_id`, and `station_id` are copied into the measurement hypertable even though they could be derived via `chart_config_id`. This eliminates JOINs in the most frequent query patterns (time-range scans filtered by tenant or station), which is critical for hypertable performance.

4. **DOUBLE PRECISION instead of NUMERIC for measurements** — hypertables use `DOUBLE PRECISION` (8 bytes) instead of `NUMERIC` (variable-length) for measured values. This provides faster arithmetic operations and better compression ratios in TimescaleDB's columnar compression. For SPC measurements with typical precision (3-6 decimal places), the ~15 digits of DOUBLE PRECISION precision is more than sufficient.

5. **Space partitioning by tenant_id** — in addition to time-based partitioning, the measurement hypertable is space-partitioned by `tenant_id` with 4 partitions. This improves query performance for multi-tenant workloads by isolating each tenant's data into separate chunk sets.

6. **Continuous aggregates replace materialised views** — instead of building and maintaining custom aggregate tables, TimescaleDB's continuous aggregates automatically compute hourly and daily statistics as new data arrives. The refresh policy ensures aggregates are updated within 30 minutes of new data, which is acceptable for SPC dashboards that typically refresh every few minutes.

7. **Compression with segment-by on chart_config_id** — compressed chunks use `chart_config_id` as the segment-by column and `time DESC` as the order-by. This means decompression during queries only extracts the rows for the requested chart config, not the entire compressed chunk. Typical compression ratios of 10-20x reduce storage costs significantly for long-term measurement retention.

8. **Tiered retention policies** — raw measurements are retained for 2 years (configurable per tenant); subgroup summaries for 5 years; alarms for 7 years; audit events for 10 years. Continuous aggregates provide statistical summaries beyond the raw data retention period. This tiered approach balances regulatory retention requirements (FDA: audit trails for life of product; IATF 16949: quality records for minimum 1 year past production) with storage costs.

9. **No foreign keys from hypertables** — TimescaleDB hypertables do not support being the target of foreign key references from other tables. The `chart_config_id` in the measurement hypertable is enforced at the application layer rather than the database layer. This is a deliberate trade-off for ingestion performance.

10. **Alarm as hypertable** — alarms are stored as a hypertable rather than a standard table, enabling time-series analysis of alarm patterns (alarm frequency trends, time-to-acknowledge analysis, seasonal alarm patterns). This supports the AI-powered alarm pattern analysis described in the project's backlog features.
