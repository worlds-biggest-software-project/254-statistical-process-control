# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Statistical Process Control · Created: 2026-05-22

## Philosophy

This model uses a pragmatic hybrid approach: core entities with well-defined, stable structures (tenants, users, parts, measurements) are modelled as typed relational columns, while variable, industry-specific, or rapidly evolving aspects (jurisdiction-specific regulatory fields, custom metadata, gage connection parameters, OEM-specific characteristic attributes) are stored in JSONB columns. The hybrid pattern keeps the relational "spine" for queries that matter most (time-range measurement lookups, capability calculations, alarm joins) while absorbing domain variability through JSONB without requiring schema migrations.

This approach is particularly well-suited for SPC because the domain spans multiple industries (automotive, aerospace, medical devices, food, semiconductor) with different regulatory requirements, terminology conventions, and metadata needs. An automotive supplier using IATF 16949 attaches PPAP submission data and AQDEF K-fields to characteristics; a medical device manufacturer using ISO 13485 attaches device master record references and FDA establishment registration numbers; a food manufacturer attaches HACCP critical control point classifications. Rather than creating industry-specific columns or separate tables for each vertical, the hybrid model absorbs these differences into well-structured JSONB fields.

The pattern is widely used in modern SaaS platforms: Shopify uses JSONB metafields for merchant-specific product attributes; Stripe stores payment method details in JSONB while keeping core transaction data relational; GitHub stores repository settings and webhook configurations in JSONB columns. For SPC, this approach enables a single schema to serve all manufacturing verticals while maintaining query performance on the columns that drive control chart rendering and capability analysis.

**Best for:** Rapid MVP development targeting multiple manufacturing verticals; teams wanting to avoid frequent schema migrations; platforms that need to absorb OEM-specific or jurisdiction-specific metadata without per-customer database changes.

**Trade-offs:**
- (+) Fewer tables (~20) compared to fully normalized model — simpler to understand and deploy
- (+) New industry-specific fields can be added without schema migrations — just extend JSONB
- (+) JSONB containment queries (@>, ?) and GIN indexes enable efficient filtering on flexible fields
- (+) Single schema serves automotive, aerospace, medical device, food, and semiconductor verticals
- (+) Faster MVP delivery — fewer tables and migrations to manage
- (-) JSONB fields lack database-level type enforcement — validation must be in application layer
- (-) Complex JSONB queries can be slower than indexed relational columns for very large datasets
- (-) JSONB fields are harder to document and discover than typed columns — requires discipline in schema documentation
- (-) Reporting tools and BI systems may struggle with nested JSONB structures
- (-) Risk of "JSONB junk drawer" if governance on JSONB field schemas is not maintained

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| AQDEF K-fields | Part and characteristic JSONB metadata can store arbitrary AQDEF K-fields (K1042 customer, K2022 decimal places, etc.) without dedicated columns |
| ISO 7870-2 | Chart type enumeration is a relational column; chart-specific parameters (EWMA lambda, smoothing factor) go in JSONB |
| ISO 22514 | Capability indices are relational columns; supplementary analysis details (normality test results, distribution fit parameters) go in JSONB |
| IATF 16949 | PPAP-specific fields (submission level, customer approval date) stored in control_plan.industry_metadata JSONB |
| FDA 21 CFR Part 11 | Audit trail is a relational table with structured who/what/when columns; change details captured in JSONB old_values/new_values |
| ISO 13485 | Medical device-specific fields (device class, 510(k) number, UDI) stored in part.industry_metadata JSONB |
| ISA-95 | Equipment hierarchy is relational (plant → area → station); equipment-specific attributes in JSONB |
| OPC UA / MQTT | Data source connection details in JSONB config column — different structure per protocol type |

---

## Core Schema

```sql
-- ============================================================
-- TENANTS & ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    industry        VARCHAR(50),            -- 'automotive', 'aerospace', 'medical_device', 'food', 'semiconductor', 'general'
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "default_rule_set": "nelson",
    --   "cpk_minimum_target": 1.33,
    --   "timezone": "America/Detroit",
    --   "regulatory_framework": "iatf_16949",
    --   "branding": {"logo_url": "...", "primary_color": "#003366"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE plant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,       -- ISO 3166-1
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes example: {
    --   "address": {"street": "...", "city": "...", "state": "MI", "postal": "48124"},
    --   "fda_establishment_number": "3004567890",
    --   "iatf_certificate_number": "IATF-2026-12345",
    --   "oem_plant_codes": {"ford": "AAI1", "gm": "LAN"}
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plant_id        UUID NOT NULL REFERENCES plant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    parent_station_id UUID REFERENCES station(id),  -- for hierarchical areas/lines
    station_type    VARCHAR(50),
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes example: {
    --   "equipment_id": "CNC-07",
    --   "area": "Machining Cell 3",
    --   "line": "Line A",
    --   "isa95_level": 3,
    --   "opcua_endpoint": "opc.tcp://10.0.1.100:4840"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plant_tenant ON plant(tenant_id);
CREATE INDEX idx_station_plant ON station(plant_id);
CREATE INDEX idx_station_parent ON station(parent_station_id);

-- ============================================================
-- USERS & ACCESS CONTROL
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    role            VARCHAR(30) NOT NULL DEFAULT 'operator', -- 'operator', 'engineer', 'manager', 'admin'
    plant_access    UUID[] DEFAULT '{}',    -- array of plant IDs; empty = all plants
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example: {
    --   "dashboard_layout": "grid",
    --   "notification_channels": ["email", "in_app"],
    --   "default_chart_view": "last_50_subgroups"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);

-- ============================================================
-- PARTS & CHARACTERISTICS
-- ============================================================

CREATE TABLE part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,
    part_name       VARCHAR(255) NOT NULL,
    revision        VARCHAR(50),
    industry_metadata JSONB NOT NULL DEFAULT '{}',
    -- Automotive example: {
    --   "aqdef": {"K1042": "Ford Motor Company", "K1041": "DWG-2026-4502"},
    --   "ppap_level": 3,
    --   "customer_part_number": "6L2Z-6049-AA",
    --   "special_char_designation": "CC/SC"
    -- }
    -- Medical device example: {
    --   "device_class": "II",
    --   "fda_510k_number": "K223456",
    --   "udi_di": "00850012345678",
    --   "dmr_reference": "DMR-2026-0042"
    -- }
    -- Food example: {
    --   "haccp_plan_id": "HACCP-2026-003",
    --   "allergen_info": ["wheat", "soy"],
    --   "shelf_life_days": 180
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, part_number, revision)
);

CREATE TABLE characteristic (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id             UUID NOT NULL REFERENCES part(id),
    number              INT NOT NULL,
    name                VARCHAR(255) NOT NULL,
    data_type           VARCHAR(20) NOT NULL DEFAULT 'variable', -- 'variable', 'attribute'
    unit_of_measure     VARCHAR(50),
    nominal_value       NUMERIC(20,6),
    upper_spec_limit    NUMERIC(20,6),
    lower_spec_limit    NUMERIC(20,6),
    is_special_char     BOOLEAN NOT NULL DEFAULT false,
    metadata            JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "aqdef_fields": {"K2003": "Bore diameter at 90deg", "K2022": 3},
    --   "special_char_class": "safety_critical",
    --   "gdt_symbol": "⌀",
    --   "measurement_method": "CMM probe",
    --   "gage_id": "G-0047",
    --   "characteristic_group": "bore_dimensions"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (part_id, number)
);

CREATE INDEX idx_part_tenant ON part(tenant_id);
CREATE INDEX idx_char_part ON characteristic(part_id);
CREATE INDEX idx_char_special ON characteristic(is_special_char) WHERE is_special_char = true;
CREATE INDEX idx_part_metadata ON part USING GIN (industry_metadata);
CREATE INDEX idx_char_metadata ON characteristic USING GIN (metadata);

-- ============================================================
-- CONTROL PLANS & CHART CONFIGURATION
-- ============================================================

CREATE TABLE control_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    plan_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    part_id         UUID NOT NULL REFERENCES part(id),
    revision        VARCHAR(50),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    effective_date  DATE,
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    industry_metadata JSONB NOT NULL DEFAULT '{}',
    -- Automotive: {"ppap_submission_date": "2026-05-01", "customer_approval": true}
    -- Medical: {"design_verification_ref": "DV-2026-012", "process_validation_ref": "PV-2026-008"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chart_config (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    control_plan_id     UUID NOT NULL REFERENCES control_plan(id),
    characteristic_id   UUID NOT NULL REFERENCES characteristic(id),
    station_id          UUID REFERENCES station(id),
    chart_type          VARCHAR(30) NOT NULL, -- 'xbar_r', 'xbar_s', 'i_mr', 'p', 'np', 'c', 'u', 'ewma'
    subgroup_size       INT NOT NULL DEFAULT 5,
    sampling_frequency  VARCHAR(100),
    -- Control limits (current)
    ucl                 NUMERIC(20,6),
    lcl                 NUMERIC(20,6),
    center_line         NUMERIC(20,6),
    ucl_secondary       NUMERIC(20,6),       -- UCL for R/S chart
    lcl_secondary       NUMERIC(20,6),       -- LCL for R/S chart
    cl_secondary        NUMERIC(20,6),       -- CL for R/S chart
    phase               VARCHAR(10) NOT NULL DEFAULT 'I',
    -- Rules to apply
    rule_set            VARCHAR(30) NOT NULL DEFAULT 'nelson', -- 'nelson', 'western_electric', 'custom'
    custom_rules        JSONB,               -- only if rule_set = 'custom'
    -- custom_rules example: [
    --   {"rule": "1_beyond_3sigma", "enabled": true, "severity": "alarm"},
    --   {"rule": "2_of_3_beyond_2sigma", "enabled": true, "severity": "warning"},
    --   {"rule": "7_consecutive_one_side", "enabled": false}
    -- ]
    chart_parameters    JSONB NOT NULL DEFAULT '{}',
    -- EWMA example: {"lambda": 0.2, "L": 3.0}
    -- Laney P' example: {"overdispersion_sigma": 1.4}
    reaction_plan       TEXT,
    monitoring_method   VARCHAR(255),
    sort_order          INT NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cp_tenant ON control_plan(tenant_id);
CREATE INDEX idx_cp_part ON control_plan(part_id);
CREATE INDEX idx_cc_plan ON chart_config(control_plan_id);
CREATE INDEX idx_cc_char ON chart_config(characteristic_id);

-- ============================================================
-- MEASUREMENT DATA
-- ============================================================

CREATE TABLE subgroup (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chart_config_id     UUID NOT NULL REFERENCES chart_config(id),
    subgroup_number     BIGINT NOT NULL,
    collected_at        TIMESTAMPTZ NOT NULL,
    collected_by        UUID REFERENCES app_user(id),
    station_id          UUID REFERENCES station(id),
    -- Computed summary statistics (stored for fast chart rendering)
    mean_value          NUMERIC(20,6),
    range_value         NUMERIC(20,6),
    std_dev_value       NUMERIC(20,6),
    -- Individual sample values stored in JSONB array
    samples             JSONB NOT NULL,
    -- samples example: [10.023, 10.018, 10.031, 10.027, 10.022]
    sample_count        INT NOT NULL,
    data_source         VARCHAR(50) NOT NULL DEFAULT 'manual',
    context             JSONB NOT NULL DEFAULT '{}',
    -- context example: {
    --   "lot_number": "LOT-2026-0547",
    --   "batch_number": "B-1234",
    --   "tool_number": "T-07",
    --   "material_lot": "ML-2026-0089",
    --   "ambient_temp_c": 22.5,
    --   "humidity_pct": 45,
    --   "gage_id": "G-0047"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- For attribute charts
CREATE TABLE attribute_subgroup (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chart_config_id     UUID NOT NULL REFERENCES chart_config(id),
    subgroup_number     BIGINT NOT NULL,
    collected_at        TIMESTAMPTZ NOT NULL,
    collected_by        UUID REFERENCES app_user(id),
    station_id          UUID REFERENCES station(id),
    inspected_count     INT NOT NULL,
    defective_count     INT,
    defect_count        INT,
    data_source         VARCHAR(50) NOT NULL DEFAULT 'manual',
    context             JSONB NOT NULL DEFAULT '{}',
    defect_details      JSONB,
    -- defect_details example: [
    --   {"defect_type": "scratch", "count": 2, "location": "surface_A"},
    --   {"defect_type": "porosity", "count": 1, "location": "bore_interior"}
    -- ]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sg_chart_time ON subgroup(chart_config_id, collected_at);
CREATE INDEX idx_sg_time ON subgroup(collected_at);
CREATE INDEX idx_sg_context ON subgroup USING GIN (context);
CREATE INDEX idx_attr_sg_chart_time ON attribute_subgroup(chart_config_id, collected_at);

-- ============================================================
-- ALARMS & CORRECTIVE ACTIONS
-- ============================================================

CREATE TABLE alarm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chart_config_id UUID NOT NULL REFERENCES chart_config(id),
    subgroup_id     UUID REFERENCES subgroup(id),
    attr_subgroup_id UUID REFERENCES attribute_subgroup(id),
    rule_violated   VARCHAR(100) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    triggered_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    violation_detail JSONB NOT NULL DEFAULT '{}',
    -- violation_detail example: {
    --   "rule_description": "One point beyond 3-sigma",
    --   "violating_value": 10.089,
    --   "ucl_at_time": 10.065,
    --   "lcl_at_time": 9.935,
    --   "center_line_at_time": 10.000,
    --   "consecutive_points": [10.055, 10.062, 10.089]
    -- }
    acknowledged_by UUID REFERENCES app_user(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    resolution_notes TEXT,
    causes          JSONB DEFAULT '[]',
    -- causes example: [
    --   {"code": "M02", "category": "machine", "name": "Tool wear", "recorded_by": "uuid", "recorded_at": "..."}
    -- ]
    corrective_actions JSONB DEFAULT '[]',
    -- corrective_actions example: [
    --   {"type": "corrective", "description": "Replace insert", "assigned_to": "uuid", "due_date": "2026-05-25", "status": "completed", "completed_at": "..."}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alarm_chart ON alarm(chart_config_id);
CREATE INDEX idx_alarm_status ON alarm(status);
CREATE INDEX idx_alarm_time ON alarm(triggered_at);

-- ============================================================
-- CAPABILITY STUDIES
-- ============================================================

CREATE TABLE capability_study (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chart_config_id     UUID NOT NULL REFERENCES chart_config(id),
    study_type          VARCHAR(30) NOT NULL,
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    subgroup_count      INT NOT NULL,
    total_measurements  INT NOT NULL,
    -- Core indices (relational for fast queries and comparisons)
    cp                  NUMERIC(10,4),
    cpk                 NUMERIC(10,4),
    pp                  NUMERIC(10,4),
    ppk                 NUMERIC(10,4),
    process_mean        NUMERIC(20,6) NOT NULL,
    process_stddev      NUMERIC(20,6) NOT NULL,
    -- Extended analysis in JSONB
    analysis_detail     JSONB NOT NULL DEFAULT '{}',
    -- analysis_detail example: {
    --   "cpm": 1.45,
    --   "ppm_above_usl": 12.3,
    --   "ppm_below_lsl": 0.0,
    --   "ppm_total": 12.3,
    --   "normality_test": {"method": "anderson_darling", "p_value": 0.34, "is_normal": true},
    --   "distribution_fit": {"best_fit": "normal", "parameters": {"mean": 10.002, "sigma": 0.0089}},
    --   "confidence_intervals": {"cpk_lower_95": 1.28, "cpk_upper_95": 1.76}
    -- }
    from_date           TIMESTAMPTZ NOT NULL,
    to_date             TIMESTAMPTZ NOT NULL,
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cap_chart ON capability_study(chart_config_id);
CREATE INDEX idx_cap_time ON capability_study(calculated_at);

-- ============================================================
-- GAGE MANAGEMENT
-- ============================================================

CREATE TABLE gage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    gage_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    gage_type       VARCHAR(100),
    resolution      NUMERIC(20,8),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    calibration_due DATE,
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- attributes example: {
    --   "manufacturer": "Mitutoyo",
    --   "model": "293-340-30",
    --   "serial_number": "SN-2024-78901",
    --   "range_min": 0,
    --   "range_max": 25.0,
    --   "calibration_interval_days": 90,
    --   "last_calibration": {"date": "2026-03-15", "certificate": "CAL-2026-0891", "result": "pass"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, gage_number)
);

CREATE TABLE gage_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gage_id         UUID NOT NULL REFERENCES gage(id),
    characteristic_id UUID NOT NULL REFERENCES characteristic(id),
    study_type      VARCHAR(30) NOT NULL,
    study_date      DATE NOT NULL,
    results         JSONB NOT NULL,
    -- results example (GRR): {
    --   "method": "anova",
    --   "operators": 3,
    --   "trials": 3,
    --   "parts": 10,
    --   "repeatability_pct": 8.2,
    --   "reproducibility_pct": 3.1,
    --   "grr_pct": 8.8,
    --   "part_variation_pct": 99.6,
    --   "ndc": 15,
    --   "verdict": "acceptable"
    -- }
    conducted_by    UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gage_tenant ON gage(tenant_id);
CREATE INDEX idx_gage_study_gage ON gage_study(gage_id);

-- ============================================================
-- DATA SOURCE CONFIGURATION
-- ============================================================

CREATE TABLE data_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES station(id),
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL,
    -- MQTT: {"broker": "mqtt://10.0.1.50:1883", "topic": "line1/+/measurements", "qos": 1, "client_id": "spc-ingest-01"}
    -- OPC UA: {"endpoint": "opc.tcp://10.0.1.100:4840", "node_ids": ["ns=2;s=Temp", "ns=2;s=Pressure"], "security_policy": "Basic256Sha256"}
    -- REST poll: {"url": "https://erp.example.com/api/quality/measurements", "method": "GET", "auth": {"type": "bearer", "token_url": "..."}, "poll_interval_ms": 30000}
    -- RS-232: {"port": "/dev/ttyUSB0", "baud_rate": 9600, "data_bits": 8, "parity": "none"}
    last_received_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ds_station ON data_source(station_id);

-- ============================================================
-- AUDIT TRAIL
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    reason          TEXT,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_record ON audit_log(table_name, record_id);

-- ============================================================
-- NOTIFICATION RULES
-- ============================================================

CREATE TABLE notification_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    trigger_config  JSONB NOT NULL,
    -- trigger_config example: {
    --   "event": "alarm_triggered",
    --   "filters": {"severity": ["alarm", "critical"], "plant_ids": ["uuid1"]},
    --   "channels": [
    --     {"type": "email", "recipients": ["qe@example.com"]},
    --     {"type": "webhook", "url": "https://hooks.slack.com/..."}
    --   ],
    --   "throttle_minutes": 15
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notif_tenant ON notification_rule(tenant_id);
```

## Example Queries

### Find all parts with a specific AQDEF customer code
```sql
SELECT id, part_number, part_name, industry_metadata
FROM part
WHERE tenant_id = 'tenant-uuid'
  AND industry_metadata @> '{"aqdef": {"K1042": "Ford Motor Company"}}';
```

### Find subgroups with a specific lot number
```sql
SELECT s.id, s.collected_at, s.mean_value, s.range_value, s.samples
FROM subgroup s
WHERE s.chart_config_id = 'chart-uuid'
  AND s.context @> '{"lot_number": "LOT-2026-0547"}'
ORDER BY s.collected_at;
```

### Cross-industry query: all special characteristics with Cpk below target
```sql
SELECT p.part_number, c.name AS characteristic_name,
       cs.cpk, cs.calculated_at
FROM capability_study cs
JOIN chart_config cc ON cc.id = cs.chart_config_id
JOIN characteristic c ON c.id = cc.characteristic_id
JOIN part p ON p.id = c.part_id
WHERE p.tenant_id = 'tenant-uuid'
  AND c.is_special_char = true
  AND cs.cpk < 1.33
  AND cs.calculated_at = (
      SELECT MAX(cs2.calculated_at)
      FROM capability_study cs2
      WHERE cs2.chart_config_id = cs.chart_config_id
  )
ORDER BY cs.cpk;
```

### Defect analysis from attribute chart JSONB details
```sql
SELECT 
    dd->>'defect_type' AS defect_type,
    SUM((dd->>'count')::int) AS total_count,
    COUNT(*) AS occurrences
FROM attribute_subgroup asg,
     jsonb_array_elements(asg.defect_details) AS dd
WHERE asg.chart_config_id = 'chart-uuid'
  AND asg.collected_at >= '2026-05-01'
GROUP BY dd->>'defect_type'
ORDER BY total_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Tenant | 3 | tenant, plant, station (hierarchical via parent_station_id) |
| Users | 1 | app_user (role as column, not separate table) |
| Part & Characteristic | 2 | part, characteristic |
| Control Plan & Charts | 2 | control_plan, chart_config |
| Measurement Data | 2 | subgroup, attribute_subgroup |
| Alarms & Actions | 1 | alarm (causes and actions in JSONB arrays) |
| Capability | 1 | capability_study |
| Gage Management | 2 | gage, gage_study |
| Data Sources | 1 | data_source |
| Audit & Notifications | 2 | audit_log, notification_rule |
| **Total** | **17** | |

---

## Key Design Decisions

1. **Industry-specific metadata in JSONB rather than separate tables** — a `part.industry_metadata` JSONB column absorbs automotive AQDEF fields, medical device DMR references, and food HACCP classifications without requiring per-industry schema tables. GIN indexes enable efficient containment queries.

2. **Samples stored as JSONB array in subgroup table** — instead of a separate `measurement` table with one row per sample, the individual values are stored as a JSONB array (`[10.023, 10.018, 10.031, 10.027, 10.022]`). For subgroup sizes of 2-10, this eliminates the need for a massive child table while keeping the data accessible. Summary statistics (mean, range, std_dev) are stored as typed columns for chart rendering.

3. **Alarm causes and corrective actions as JSONB arrays** — rather than separate junction tables, the alarm table embeds causes and corrective actions as JSONB arrays. This trades write normalisation for read simplicity — a single query returns the complete alarm lifecycle. For typical SPC volumes (hundreds of alarms per month, not millions), this is practical.

4. **Flat station hierarchy with self-reference** — instead of separate plant → area → station tables, the `station` table uses a `parent_station_id` self-referencing column. This supports arbitrary hierarchy depth (plant → building → line → cell → station) without fixed schema levels.

5. **Role as a column, not a separate table** — for the MVP, user roles are a simple VARCHAR column (`operator`, `engineer`, `manager`, `admin`). This avoids the complexity of a full RBAC junction table structure. JSONB `plant_access` provides plant-level scoping. A separate role/permission table can be added later if needed.

6. **Chart-specific parameters in JSONB** — EWMA charts need lambda and L parameters; Laney P' charts need overdispersion sigma; CUSUM charts need slack value k and decision interval h. Rather than adding nullable columns for each chart type, the `chart_parameters` JSONB column stores chart-type-specific settings with application-level validation.

7. **Context JSONB on subgroups** — manufacturing context (lot number, tool number, material lot, ambient temperature) varies widely between industries and even between stations. The `context` JSONB column captures all environmental and traceability metadata with a GIN index for filtering. This is critical for root cause analysis.

8. **GIN indexes on JSONB columns** — PostgreSQL GIN (Generalised Inverted Index) indexes support efficient `@>` (containment) and `?` (key existence) queries on JSONB columns. These are applied to `industry_metadata`, `context`, and `metadata` columns that will be frequently filtered.

9. **Capability detail split** — core indices (Cp, Cpk, Pp, Ppk) are relational columns for fast sorting and comparison queries; extended analysis (distribution fitting, confidence intervals, normality tests) goes into `analysis_detail` JSONB since it is read but rarely filtered or sorted.

10. **17 tables total** — roughly half the table count of the normalized model, achieved primarily by folding junction tables, child tables, and variant-specific columns into JSONB. This significantly reduces migration complexity and makes the schema approachable for smaller development teams.
