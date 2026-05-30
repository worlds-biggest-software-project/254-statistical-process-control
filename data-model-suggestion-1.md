# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Statistical Process Control · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational design where every domain concept — part, characteristic, control plan, subgroup, measurement, alarm, corrective action — is represented by its own table with strict foreign key relationships. The schema mirrors the vocabulary and structure of the AIAG-VDA SPC Yellow Volume 2026 and AQDEF (Advanced Quality Data Exchange Format), so that database columns map directly to industry-standard fields (K1001 part number, K2101 nominal value, K2110/K2111 specification limits).

The normalized approach treats data integrity as the highest priority. Every measurement traces back through a chain of foreign keys to its characteristic, part, control plan, plant, and tenant. This makes regulatory compliance queries straightforward: "show me every measurement for characteristic X on part Y at plant Z between dates A and B, with the operator who recorded it and the control limits in effect at that time" is a standard JOIN across well-indexed tables.

Real-world systems that follow this pattern include InfinityQS ProFicient (which models processes, parts, features, and data collections as separate linked entities) and DataLyzer SPC (which links FMEA, control plans, SPC charts, and MSA studies through a unified relational data model). The AQDEF K-field hierarchy (K0xxx for values, K1xxx for parts, K2xxx for characteristics, K5xxx for structures) maps naturally to a normalized table structure.

**Best for:** Organisations in regulated industries (automotive, aerospace, medical devices) that need demonstrable traceability from raw measurement to control plan to corrective action, and that run complex cross-entity analytical queries.

**Trade-offs:**
- (+) Maximum data integrity through foreign key constraints and referential integrity
- (+) Direct alignment with AQDEF K-field structure and AIAG-VDA terminology
- (+) Complex cross-entity queries (e.g., "all out-of-control events for a part family across all plants") are natural SQL JOINs
- (+) Schema is self-documenting — column names match industry vocabulary
- (-) High table count (~30 tables) increases schema migration complexity
- (-) Junction tables for many-to-many relationships add write overhead
- (-) Adding industry-specific or jurisdiction-specific fields requires schema migrations
- (-) Performance of multi-table JOINs may degrade at very high measurement volumes (>100M rows) without careful indexing and partitioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| AQDEF K-fields (K1xxx, K2xxx) | Part and characteristic table columns map to AQDEF K-field numbers; enables direct DFQ file import/export |
| ISO 7870-2 (Shewhart Charts) | Control chart type enumeration and rule set definitions align with ISO 7870-2 chart classifications |
| ISO 22514 (Capability Indices) | Capability study table stores Cp, Cpk, Pp, Ppk per ISO 22514-2 definitions |
| AIAG-VDA SPC Yellow Volume 2026 | Terminology (special characteristics, control limits, capability vs. performance) follows the harmonised manual |
| AIAG MSA Manual | Gage R&R study tables model ANOVA and Average & Range methods per AIAG MSA |
| FDA 21 CFR Part 11 | Audit trail table captures who/what/when/why for every record change; electronic signature fields included |
| IATF 16949 | Control plan entity links characteristics to monitoring method, frequency, reaction plan, and responsible role |
| ISA-95 / IEC 62264 | Plant/area/line/station hierarchy follows ISA-95 Level 3 equipment model |
| ISO 3166-1/2 | Jurisdiction codes for multi-national plant locations use ISO 3166 |
| OPC UA | Data source configuration table stores OPC UA node IDs for automated measurement ingestion |

---

## Organisation & Tenant Management

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

CREATE TABLE plant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,       -- ISO 3166-1 alpha-2
    region_code     VARCHAR(6),             -- ISO 3166-2 subdivision
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    address         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE production_area (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plant_id        UUID NOT NULL REFERENCES plant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    area_type       VARCHAR(50),            -- 'line', 'cell', 'workstation'
    parent_area_id  UUID REFERENCES production_area(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (plant_id, code)
);

CREATE TABLE station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    area_id         UUID NOT NULL REFERENCES production_area(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    external_station_id VARCHAR(255),       -- maps to Minitab API external_station_id
    station_type    VARCHAR(50),            -- 'manual', 'automated', 'hybrid'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plant_tenant ON plant(tenant_id);
CREATE INDEX idx_area_plant ON production_area(plant_id);
CREATE INDEX idx_station_area ON station(area_id);
```

## User & Access Control

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),           -- NULL if SSO-only
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- 'operator', 'engineer', 'manager', 'admin'
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    plant_id        UUID REFERENCES plant(id),  -- NULL = all plants
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, plant_id)
);

CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    meaning         VARCHAR(255) NOT NULL,  -- FDA 21 CFR Part 11: meaning of signature
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    record_type     VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    ip_address      INET
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_esig_record ON electronic_signature(record_type, record_id);
```

## Part & Characteristic Definitions

```sql
CREATE TABLE part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,  -- AQDEF K1001
    part_name       VARCHAR(255) NOT NULL,  -- AQDEF K1002
    revision        VARCHAR(50),            -- AQDEF K1010
    customer        VARCHAR(255),           -- AQDEF K1042
    drawing_number  VARCHAR(100),           -- AQDEF K1041
    part_family     VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number, revision)
);

CREATE TABLE characteristic (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id             UUID NOT NULL REFERENCES part(id),
    characteristic_number INT NOT NULL,     -- AQDEF K2001
    name                VARCHAR(255) NOT NULL, -- AQDEF K2002
    description         TEXT,               -- AQDEF K2003
    data_type           VARCHAR(20) NOT NULL DEFAULT 'variable',  -- 'variable' or 'attribute'
    unit_of_measure     VARCHAR(50),        -- AQDEF K2005 (e.g., 'mm', 'kg', 'degC')
    nominal_value       NUMERIC(20,6),      -- AQDEF K2101
    upper_spec_limit    NUMERIC(20,6),      -- AQDEF K2111
    lower_spec_limit    NUMERIC(20,6),      -- AQDEF K2110
    target_value        NUMERIC(20,6),      -- AQDEF K2100
    is_special_char     BOOLEAN NOT NULL DEFAULT false, -- IATF 16949 special characteristic
    special_char_class  VARCHAR(50),        -- 'safety', 'critical', 'significant' per OEM
    decimal_places      INT DEFAULT 3,      -- AQDEF K2022
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (part_id, characteristic_number)
);

CREATE INDEX idx_part_tenant ON part(tenant_id);
CREATE INDEX idx_char_part ON characteristic(part_id);
CREATE INDEX idx_char_special ON characteristic(is_special_char) WHERE is_special_char = true;
```

## Control Plan & Chart Configuration

```sql
CREATE TABLE control_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    plan_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    part_id         UUID NOT NULL REFERENCES part(id),
    revision        VARCHAR(50),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft', -- 'draft', 'active', 'superseded'
    effective_date  DATE,
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE control_plan_item (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_id     UUID NOT NULL REFERENCES control_plan(id),
    characteristic_id   UUID NOT NULL REFERENCES characteristic(id),
    station_id          UUID REFERENCES station(id),
    chart_type          VARCHAR(30) NOT NULL, -- 'xbar_r', 'xbar_s', 'i_mr', 'p', 'np', 'c', 'u', 'ewma'
    subgroup_size       INT NOT NULL DEFAULT 5,
    sampling_frequency  VARCHAR(100),       -- e.g., 'every 30 minutes', 'every 50 parts'
    sample_size         INT,
    reaction_plan       TEXT,               -- what to do on out-of-control
    monitoring_method   VARCHAR(255),       -- measurement tool or method
    responsible_role    VARCHAR(100),       -- 'operator', 'engineer'
    sort_order          INT NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chart_config (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_item_id UUID NOT NULL REFERENCES control_plan_item(id),
    ucl                 NUMERIC(20,6),      -- upper control limit
    lcl                 NUMERIC(20,6),      -- lower control limit
    center_line         NUMERIC(20,6),      -- process mean or target
    ucl_range           NUMERIC(20,6),      -- UCL for R or S chart
    lcl_range           NUMERIC(20,6),      -- LCL for R or S chart
    center_line_range   NUMERIC(20,6),      -- center line for R or S chart
    phase               VARCHAR(10) NOT NULL DEFAULT 'I',  -- 'I' (trial) or 'II' (standard)
    effective_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    effective_to        TIMESTAMPTZ,        -- NULL = currently active
    calculated_from_subgroups INT,          -- number of subgroups used to compute limits
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spc_rule_set (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- 'western_electric', 'nelson', 'custom'
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spc_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_set_id     UUID NOT NULL REFERENCES spc_rule_set(id),
    rule_number     INT NOT NULL,
    name            VARCHAR(255) NOT NULL,  -- e.g., 'One point beyond 3-sigma'
    description     TEXT,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning', -- 'warning', 'alarm', 'critical'
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    parameters      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"sigma_multiple": 3, "consecutive_points": 1}
    UNIQUE (rule_set_id, rule_number)
);

CREATE TABLE chart_rule_assignment (
    chart_config_id UUID NOT NULL REFERENCES chart_config(id),
    rule_set_id     UUID NOT NULL REFERENCES spc_rule_set(id),
    PRIMARY KEY (chart_config_id, rule_set_id)
);

CREATE INDEX idx_cp_tenant ON control_plan(tenant_id);
CREATE INDEX idx_cp_part ON control_plan(part_id);
CREATE INDEX idx_cpi_plan ON control_plan_item(control_plan_id);
CREATE INDEX idx_cpi_char ON control_plan_item(characteristic_id);
CREATE INDEX idx_chartcfg_cpi ON chart_config(control_plan_item_id);
CREATE INDEX idx_chartcfg_effective ON chart_config(effective_from, effective_to);
```

## Measurement Data

```sql
CREATE TABLE subgroup (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_item_id UUID NOT NULL REFERENCES control_plan_item(id),
    subgroup_number     BIGINT NOT NULL,
    collected_at        TIMESTAMPTZ NOT NULL,
    collected_by        UUID REFERENCES app_user(id),
    station_id          UUID REFERENCES station(id),
    lot_number          VARCHAR(100),
    batch_number        VARCHAR(100),
    mean_value          NUMERIC(20,6),      -- computed Xbar
    range_value         NUMERIC(20,6),      -- computed R
    std_dev_value       NUMERIC(20,6),      -- computed S
    sample_count        INT NOT NULL,
    data_source         VARCHAR(50) NOT NULL DEFAULT 'manual',
    -- 'manual', 'api', 'mqtt', 'opcua', 'gage'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE measurement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subgroup_id     UUID NOT NULL REFERENCES subgroup(id),
    sample_number   INT NOT NULL,           -- position within subgroup (1..n)
    value           NUMERIC(20,6) NOT NULL, -- AQDEF K0001 measured value
    is_excluded     BOOLEAN NOT NULL DEFAULT false,
    exclusion_reason TEXT,
    raw_value       NUMERIC(20,6),          -- pre-transformation value if applicable
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- For attribute charts (P, NP, C, U)
CREATE TABLE attribute_subgroup (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_item_id UUID NOT NULL REFERENCES control_plan_item(id),
    subgroup_number     BIGINT NOT NULL,
    collected_at        TIMESTAMPTZ NOT NULL,
    collected_by        UUID REFERENCES app_user(id),
    station_id          UUID REFERENCES station(id),
    inspected_count     INT NOT NULL,       -- n (sample size)
    defective_count     INT,                -- np (for P and NP charts)
    defect_count        INT,                -- c (for C and U charts)
    lot_number          VARCHAR(100),
    data_source         VARCHAR(50) NOT NULL DEFAULT 'manual',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subgroup_cpi ON subgroup(control_plan_item_id);
CREATE INDEX idx_subgroup_time ON subgroup(collected_at);
CREATE INDEX idx_subgroup_cpi_time ON subgroup(control_plan_item_id, collected_at);
CREATE INDEX idx_measurement_subgroup ON measurement(subgroup_id);
CREATE INDEX idx_attr_subgroup_cpi ON attribute_subgroup(control_plan_item_id);
CREATE INDEX idx_attr_subgroup_time ON attribute_subgroup(collected_at);
```

## Alarms & Corrective Actions

```sql
CREATE TABLE alarm (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_item_id UUID NOT NULL REFERENCES control_plan_item(id),
    subgroup_id         UUID REFERENCES subgroup(id),
    attr_subgroup_id    UUID REFERENCES attribute_subgroup(id),
    rule_id             UUID NOT NULL REFERENCES spc_rule(id),
    severity            VARCHAR(20) NOT NULL, -- 'warning', 'alarm', 'critical'
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    -- 'open', 'acknowledged', 'resolved', 'false_positive'
    triggered_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_by     UUID REFERENCES app_user(id),
    acknowledged_at     TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assignable_cause (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),           -- Ishikawa 5M: 'man', 'machine', 'material', 'method', 'measurement'
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE alarm_cause (
    alarm_id        UUID NOT NULL REFERENCES alarm(id),
    cause_id        UUID NOT NULL REFERENCES assignable_cause(id),
    recorded_by     UUID NOT NULL REFERENCES app_user(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT,
    PRIMARY KEY (alarm_id, cause_id)
);

CREATE TABLE corrective_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alarm_id        UUID NOT NULL REFERENCES alarm(id),
    action_type     VARCHAR(50) NOT NULL,   -- 'containment', 'corrective', 'preventive'
    description     TEXT NOT NULL,
    assigned_to     UUID REFERENCES app_user(id),
    due_date        DATE,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    -- 'open', 'in_progress', 'completed', 'verified'
    completed_at    TIMESTAMPTZ,
    verified_by     UUID REFERENCES app_user(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alarm_cpi ON alarm(control_plan_item_id);
CREATE INDEX idx_alarm_status ON alarm(status);
CREATE INDEX idx_alarm_time ON alarm(triggered_at);
CREATE INDEX idx_ca_alarm ON corrective_action(alarm_id);
CREATE INDEX idx_ca_status ON corrective_action(status);
```

## Capability Studies

```sql
CREATE TABLE capability_study (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_item_id UUID NOT NULL REFERENCES control_plan_item(id),
    study_type          VARCHAR(30) NOT NULL, -- 'short_term', 'long_term', 'preliminary'
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    subgroup_count      INT NOT NULL,
    total_measurements  INT NOT NULL,
    process_mean        NUMERIC(20,6) NOT NULL,
    process_stddev      NUMERIC(20,6) NOT NULL,
    cp                  NUMERIC(10,4),      -- ISO 22514-2
    cpk                 NUMERIC(10,4),      -- ISO 22514-2
    pp                  NUMERIC(10,4),      -- ISO 22514-2
    ppk                 NUMERIC(10,4),      -- ISO 22514-2
    cpm                 NUMERIC(10,4),      -- Taguchi capability index
    ppm_above_usl       NUMERIC(12,2),
    ppm_below_lsl       NUMERIC(12,2),
    ppm_total           NUMERIC(12,2),
    is_normal           BOOLEAN,            -- result of normality test
    normality_p_value   NUMERIC(10,6),
    from_date           TIMESTAMPTZ NOT NULL,
    to_date             TIMESTAMPTZ NOT NULL,
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_capstudy_cpi ON capability_study(control_plan_item_id);
CREATE INDEX idx_capstudy_time ON capability_study(calculated_at);
```

## Gage / Measurement System Analysis

```sql
CREATE TABLE gage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    gage_id_number  VARCHAR(100) NOT NULL,  -- AQDEF K2402
    name            VARCHAR(255) NOT NULL,
    manufacturer    VARCHAR(255),           -- AQDEF K2406
    gage_type       VARCHAR(100),
    resolution      NUMERIC(20,8),          -- AQDEF K2404
    calibration_due DATE,
    calibration_interval_days INT,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    -- 'active', 'due', 'expired', 'retired'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, gage_id_number)
);

CREATE TABLE gage_rr_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gage_id         UUID NOT NULL REFERENCES gage(id),
    characteristic_id UUID NOT NULL REFERENCES characteristic(id),
    study_type      VARCHAR(30) NOT NULL,
    -- 'average_range', 'anova', 'bias', 'linearity', 'stability'
    operator_count  INT NOT NULL,
    trial_count     INT NOT NULL,
    part_count      INT NOT NULL,
    repeatability   NUMERIC(10,4),          -- %EV (equipment variation)
    reproducibility NUMERIC(10,4),          -- %AV (appraiser variation)
    gage_rr         NUMERIC(10,4),          -- %GRR
    part_variation  NUMERIC(10,4),          -- %PV
    total_variation NUMERIC(10,4),          -- %TV
    ndc             NUMERIC(10,2),          -- number of distinct categories
    is_acceptable   BOOLEAN,
    -- GRR < 10% acceptable, 10-30% marginal, >30% unacceptable
    study_date      DATE NOT NULL,
    conducted_by    UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gage_tenant ON gage(tenant_id);
CREATE INDEX idx_gage_status ON gage(status);
CREATE INDEX idx_grr_gage ON gage_rr_study(gage_id);
```

## Data Source Configuration

```sql
CREATE TABLE data_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES station(id),
    source_type     VARCHAR(30) NOT NULL,
    -- 'rest_api', 'mqtt', 'opcua', 'tcp', 'rs232', 'manual'
    name            VARCHAR(255) NOT NULL,
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- MQTT: {"broker": "mqtt://10.0.1.50:1883", "topic": "line1/station3/#", "qos": 1}
    -- OPC UA: {"endpoint": "opc.tcp://10.0.1.100:4840", "node_id": "ns=2;s=Temperature"}
    -- REST: {"url": "https://erp.example.com/api/measurements", "auth_type": "bearer"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    polling_interval_ms INT,
    last_received_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_datasource_station ON data_source(station_id);
```

## Audit Trail (FDA 21 CFR Part 11)

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,
    -- 'create', 'update', 'delete', 'login', 'approve', 'sign'
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    old_values      JSONB,                  -- previous state (NULL for creates)
    new_values      JSONB,                  -- new state (NULL for deletes)
    reason          TEXT,                   -- 21 CFR Part 11: reason for change
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition by month for performance at scale
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
```

## Notification Configuration

```sql
CREATE TABLE notification_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    -- 'alarm_triggered', 'capability_degraded', 'gage_due'
    severity_filter VARCHAR(20),            -- NULL = all severities
    channel         VARCHAR(30) NOT NULL,   -- 'email', 'webhook', 'in_app', 'sms'
    config          JSONB NOT NULL DEFAULT '{}',
    -- Email: {"recipients": ["qe@example.com"], "template": "alarm_notification"}
    -- Webhook: {"url": "https://hooks.example.com/spc", "secret": "..."}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES notification_rule(id),
    alarm_id        UUID REFERENCES alarm(id),
    channel         VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL,   -- 'sent', 'failed', 'delivered'
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    response_code   INT,
    error_message   TEXT
);

CREATE INDEX idx_notif_rule_tenant ON notification_rule(tenant_id);
CREATE INDEX idx_notif_log_alarm ON notification_log(alarm_id);
```

---

## Example Queries

**Find all characteristics with Cpk below target across a plant:**

```sql
SELECT p.part_number, c.name AS characteristic, cs.cpk, cs.calculated_at
FROM capability_study cs
JOIN control_plan_item cpi ON cs.control_plan_item_id = cpi.id
JOIN characteristic c ON cpi.characteristic_id = c.id
JOIN part p ON c.part_id = p.id
JOIN station s ON cpi.station_id = s.id
JOIN production_area a ON s.area_id = a.id
WHERE a.plant_id = :plant_id
  AND cs.cpk < 1.33
  AND cs.calculated_at >= now() - INTERVAL '30 days'
ORDER BY cs.cpk ASC;
```

**Get subgroup data for a control chart:**

```sql
SELECT sg.subgroup_number, sg.collected_at, sg.mean_value, sg.range_value,
       cc.ucl, cc.lcl, cc.center_line
FROM subgroup sg
JOIN control_plan_item cpi ON sg.control_plan_item_id = cpi.id
JOIN chart_config cc ON cc.control_plan_item_id = cpi.id
WHERE sg.control_plan_item_id = :chart_item_id
  AND sg.collected_at BETWEEN :start_date AND :end_date
  AND cc.effective_from <= sg.collected_at
  AND (cc.effective_to IS NULL OR cc.effective_to > sg.collected_at)
ORDER BY sg.collected_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Tenant | 4 | tenant, plant, production_area, station |
| User & Access Control | 4 | app_user, role, user_role, electronic_signature |
| Part & Characteristic | 2 | part, characteristic |
| Control Plan & Charts | 6 | control_plan, control_plan_item, chart_config, spc_rule_set, spc_rule, chart_rule_assignment |
| Measurement Data | 3 | subgroup, measurement, attribute_subgroup |
| Alarms & Corrective Actions | 4 | alarm, assignable_cause, alarm_cause, corrective_action |
| Capability Studies | 1 | capability_study |
| Gage / MSA | 2 | gage, gage_rr_study |
| Data Sources | 1 | data_source |
| Audit & Compliance | 1 | audit_log |
| Notifications | 2 | notification_rule, notification_log |
| **Total** | **30** | |

---

## Key Design Decisions

1. **UUID primary keys** throughout — standard for multi-tenant SaaS; avoids sequential ID leakage between tenants and supports distributed ID generation.

2. **ISA-95 equipment hierarchy** (tenant > plant > production_area > station) — mirrors the ISA-95 Level 3 model and matches InfinityQS's hierarchical data views (line > plant > enterprise).

3. **Separate tables for variable and attribute subgroups** — variable charts (Xbar-R, I-MR) and attribute charts (P, NP, C, U) have fundamentally different data shapes. A single polymorphic table would require many nullable columns; separate tables are cleaner and more type-safe.

4. **Chart config with temporal validity** (`effective_from`/`effective_to`) — control limits change when processes are re-studied. Storing historical configs enables Phase I/Phase II analysis and retrospective investigation ("what limits were in effect when this alarm fired?").

5. **AQDEF K-field column naming** — characteristic columns (nominal_value, upper_spec_limit, lower_spec_limit) explicitly reference their AQDEF K-field numbers in comments, enabling straightforward DFQ file import/export for automotive customers.

6. **5M assignable cause taxonomy** — the assignable_cause.category column uses the classic Ishikawa 5M categories (man, machine, material, method, measurement), which aligns with AIAG-VDA root cause analysis methodology.

7. **FDA 21 CFR Part 11 audit trail** — the audit_log table captures old/new values, reason for change, user identity, and timestamp for every record modification. Combined with electronic_signature, this satisfies FDA Part 11 Section 11.10(e) requirements.

8. **JSONB used sparingly** — only for configuration blobs (data_source.connection_config, spc_rule.parameters, tenant.settings, role.permissions) where the structure varies by type. All analytical/queryable data uses typed columns.

9. **Composite indexes on (control_plan_item_id, collected_at)** for measurement queries — the most common SPC query pattern is "get subgroups for this chart item in this time range," so this index is critical for performance.

10. **Multi-tenant via tenant_id foreign keys** — supports PostgreSQL row-level security policies for tenant isolation while keeping all tenants in a shared database for operational simplicity.
