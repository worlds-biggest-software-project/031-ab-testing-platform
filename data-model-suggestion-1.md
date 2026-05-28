# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: A/B Testing Platform · Created: 2026-05-11

## Philosophy

This model follows classical relational database design: every concept gets its own table, relationships are enforced via foreign keys, and all data is fully normalized to at least 3NF. The schema prioritises data integrity, query flexibility, and clear domain boundaries over write throughput or schema evolution speed.

Real-world systems that use this pattern include Flagbase (which documents its models in 2NF), traditional SaaS platforms like LaunchDarkly's internal storage, and enterprise experimentation platforms like Optimizely that need rigid governance over experiment configurations. The normalized approach maps well to the experiment domain because experiments have well-defined lifecycle states, metrics have standardised definitions, and relationships between experiments, variants, metrics, and segments are inherently many-to-many.

This approach is best when the team values referential integrity, needs complex cross-entity reporting (e.g., "show all experiments that used metric X across projects Y and Z"), and expects to build a rich governance layer with audit logging and RBAC.

**Best for:** Teams building a long-lived enterprise platform where data integrity, governance, and complex cross-entity querying outweigh schema evolution speed.

**Trade-offs:**
- (+) Strong referential integrity prevents orphaned variants, dangling metric references, or invalid targeting rules
- (+) Complex analytical queries (e.g., cross-experiment metric impact reports) are straightforward SQL joins
- (+) Clear domain boundaries make the system easy to reason about and test
- (+) RBAC and audit logging fit naturally into normalised tables
- (-) High table count (~35-40 tables) increases migration complexity
- (-) Many-to-many junction tables add write overhead for experiment creation/updates
- (-) Targeting rule conditions stored relationally require multiple joins to reconstruct a complete rule set
- (-) Schema changes require migrations; adding a new field to experiments requires ALTER TABLE

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature (CNCF) | Evaluation context model informs the `evaluation_context` and `targeting_attribute` table structures; targeting key concept maps to `context_key` |
| ISO 3166-1 | Country/region codes for geo-targeting segments |
| JSON Schema (draft 2020-12) | Used to validate dynamic config values, following Statsig's schema validation pattern |
| MongoDB Query Syntax | Targeting conditions use MongoDB-style operators (`$eq`, `$in`, `$gte`, etc.) following GrowthBook's approach, but stored as normalised condition rows |
| OpenTelemetry | SDK event schema aligns with OTel span attributes for exposure tracking |

---

## Organisation & Multi-Tenancy

```sql
CREATE TABLE organization (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',  -- org-level defaults (stats engine, significance threshold)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    settings JSONB NOT NULL DEFAULT '{}',  -- project-level overrides
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE TABLE environment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,         -- e.g., 'production', 'staging', 'development'
    slug VARCHAR(100) NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

CREATE INDEX idx_project_org ON project(organization_id);
CREATE INDEX idx_environment_project ON environment(project_id);
```

---

## User & RBAC

```sql
CREATE TABLE app_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    password_hash VARCHAR(255),           -- NULL if SSO-only
    auth_provider VARCHAR(50),            -- 'local', 'google', 'saml'
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,           -- e.g., 'admin', 'experimenter', 'viewer'
    permissions JSONB NOT NULL DEFAULT '[]',
    -- permissions example: ["experiment:create", "experiment:read", "flag:create", "metric:manage"]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_org_membership (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES role(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, organization_id)
);

CREATE TABLE user_project_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES role(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, project_id)
);

CREATE INDEX idx_membership_user ON user_org_membership(user_id);
CREATE INDEX idx_membership_org ON user_org_membership(organization_id);
CREATE INDEX idx_project_role_user ON user_project_role(user_id);
```

---

## Data Source Connections

```sql
CREATE TABLE data_source (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,            -- 'bigquery', 'snowflake', 'databricks', 'redshift', 'postgres', 'clickhouse'
    connection_params_encrypted BYTEA NOT NULL,  -- encrypted JSON with host, credentials, dataset/schema
    settings JSONB NOT NULL DEFAULT '{}',        -- warehouse-specific settings (e.g., BigQuery project, Snowflake warehouse)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fact_table (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id UUID NOT NULL REFERENCES data_source(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    sql_query TEXT NOT NULL,              -- SELECT statement defining the fact table
    user_id_column VARCHAR(255) NOT NULL, -- column name for user identifier
    timestamp_column VARCHAR(255) NOT NULL, -- column name for event timestamp
    id_type VARCHAR(100) NOT NULL DEFAULT 'user_id',  -- identifier type: 'user_id', 'anonymous_id', 'device_id'
    columns JSONB NOT NULL DEFAULT '[]',  -- discovered/declared column metadata
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assignment_query (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id UUID NOT NULL REFERENCES data_source(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    sql_query TEXT NOT NULL,              -- must return: user_id, timestamp, experiment_id, variation_id
    id_type VARCHAR(100) NOT NULL DEFAULT 'user_id',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_source_project ON data_source(project_id);
CREATE INDEX idx_fact_table_source ON fact_table(data_source_id);
CREATE INDEX idx_assignment_query_source ON assignment_query(data_source_id);
```

---

## Metrics

```sql
CREATE TABLE metric (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50) NOT NULL,            -- 'binomial', 'count', 'duration', 'revenue', 'ratio', 'quantile'
    fact_table_id UUID REFERENCES fact_table(id),
    aggregation VARCHAR(50) NOT NULL DEFAULT 'sum',  -- 'sum', 'count', 'avg', 'countDistinct', 'max', 'min'
    value_column VARCHAR(255),            -- column to aggregate (NULL for binomial/count)
    denominator_metric_id UUID REFERENCES metric(id),  -- for ratio metrics
    sql_filter TEXT,                      -- optional WHERE clause to filter fact table rows
    cap_value NUMERIC,                    -- winsorization cap for outlier handling
    conversion_window_hours INTEGER DEFAULT 72,  -- hours after exposure to count conversions
    delay_hours INTEGER DEFAULT 0,        -- hours to wait before including data (novelty effect buffer)
    is_inverse BOOLEAN NOT NULL DEFAULT false,  -- true if lower is better (e.g., bounce rate, load time)
    min_sample_size INTEGER DEFAULT 100,
    tags TEXT[] DEFAULT '{}',
    owner_id UUID REFERENCES app_user(id),
    status VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'active', 'archived', 'draft'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

-- Pre-experiment covariate data for CUPED variance reduction
CREATE TABLE metric_covariate (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    metric_id UUID NOT NULL REFERENCES metric(id) ON DELETE CASCADE,
    covariate_metric_id UUID NOT NULL REFERENCES metric(id),
    lookback_days INTEGER NOT NULL DEFAULT 7,  -- days before exposure to collect pre-experiment data
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (metric_id, covariate_metric_id)
);

CREATE INDEX idx_metric_project ON metric(project_id);
CREATE INDEX idx_metric_slug ON metric(project_id, slug);
CREATE INDEX idx_metric_tags ON metric USING GIN(tags);
CREATE INDEX idx_metric_type ON metric(project_id, type);
```

---

## Segments & Targeting

```sql
CREATE TABLE segment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES app_user(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

-- Each segment has one or more rules (OR logic between rules)
CREATE TABLE segment_rule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id UUID NOT NULL REFERENCES segment(id) ON DELETE CASCADE,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Each rule has one or more conditions (AND logic within a rule)
CREATE TABLE segment_condition (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id UUID NOT NULL REFERENCES segment_rule(id) ON DELETE CASCADE,
    attribute VARCHAR(255) NOT NULL,      -- e.g., 'country', 'plan', 'browser', 'user.age'
    operator VARCHAR(50) NOT NULL,        -- '$eq', '$ne', '$in', '$nin', '$gt', '$gte', '$lt', '$lte', '$regex', '$exists'
    value TEXT NOT NULL,                  -- JSON-encoded value (string, number, array for $in)
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_segment_project ON segment(project_id);
CREATE INDEX idx_segment_rule ON segment_rule(segment_id);
CREATE INDEX idx_segment_condition ON segment_condition(rule_id);
```

---

## Feature Flags

```sql
CREATE TABLE feature_flag (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,            -- unique feature key used in SDK: e.g., 'dark-mode', 'new-checkout'
    description TEXT,
    value_type VARCHAR(20) NOT NULL DEFAULT 'boolean',  -- 'boolean', 'string', 'number', 'json'
    default_value TEXT NOT NULL DEFAULT 'false',         -- JSON-encoded default
    tags TEXT[] DEFAULT '{}',
    owner_id UUID REFERENCES app_user(id),
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- 'draft', 'active', 'archived'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, key)
);

-- Per-environment flag state
CREATE TABLE flag_environment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    feature_flag_id UUID NOT NULL REFERENCES feature_flag(id) ON DELETE CASCADE,
    environment_id UUID NOT NULL REFERENCES environment(id) ON DELETE CASCADE,
    enabled BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (feature_flag_id, environment_id)
);

-- Targeting rules per flag+environment (evaluated in order)
CREATE TABLE flag_rule (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environment(id) ON DELETE CASCADE,
    description VARCHAR(500),
    sort_order INTEGER NOT NULL DEFAULT 0,
    type VARCHAR(20) NOT NULL DEFAULT 'targeting',  -- 'targeting', 'force', 'rollout', 'experiment'
    enabled BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Conditions within a flag rule (AND logic)
CREATE TABLE flag_rule_condition (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_rule_id UUID NOT NULL REFERENCES flag_rule(id) ON DELETE CASCADE,
    attribute VARCHAR(255) NOT NULL,
    operator VARCHAR(50) NOT NULL,
    value TEXT NOT NULL,                  -- JSON-encoded
    sort_order INTEGER NOT NULL DEFAULT 0
);

-- Variations served by a rule (percentage rollout across variations)
CREATE TABLE flag_rule_variation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_rule_id UUID NOT NULL REFERENCES flag_rule(id) ON DELETE CASCADE,
    variation_value TEXT NOT NULL,         -- JSON-encoded variation value
    weight INTEGER NOT NULL DEFAULT 0,    -- 0-10000 (basis points for percentage, e.g., 5000 = 50%)
    name VARCHAR(100)
);

-- Explicit user overrides per flag+environment
CREATE TABLE flag_override (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flag_environment_id UUID NOT NULL REFERENCES flag_environment(id) ON DELETE CASCADE,
    context_key VARCHAR(500) NOT NULL,    -- user ID, device ID, etc.
    variation_value TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (flag_environment_id, context_key)
);

CREATE INDEX idx_flag_project ON feature_flag(project_id);
CREATE INDEX idx_flag_key ON feature_flag(project_id, key);
CREATE INDEX idx_flag_env ON flag_environment(feature_flag_id);
CREATE INDEX idx_flag_rule_env ON flag_rule(flag_environment_id, sort_order);
CREATE INDEX idx_flag_rule_cond ON flag_rule_condition(flag_rule_id);
CREATE INDEX idx_flag_override_env ON flag_override(flag_environment_id);
```

---

## Experiments

```sql
CREATE TABLE experiment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    hypothesis TEXT,
    type VARCHAR(30) NOT NULL DEFAULT 'ab',  -- 'ab', 'multivariate', 'split_url', 'bandit'
    status VARCHAR(30) NOT NULL DEFAULT 'draft',  -- 'draft', 'running', 'paused', 'stopped', 'completed', 'archived'
    feature_flag_id UUID REFERENCES feature_flag(id),  -- optional link to feature flag
    assignment_query_id UUID REFERENCES assignment_query(id),  -- warehouse assignment source
    hash_attribute VARCHAR(100) NOT NULL DEFAULT 'id',  -- attribute used for consistent hashing
    hash_version INTEGER NOT NULL DEFAULT 2,
    targeting_segment_id UUID REFERENCES segment(id),  -- who is eligible
    stats_engine VARCHAR(20) NOT NULL DEFAULT 'frequentist',  -- 'frequentist', 'bayesian'
    significance_level NUMERIC(5,4) NOT NULL DEFAULT 0.0500,  -- p-value threshold (e.g., 0.05)
    power NUMERIC(5,4) NOT NULL DEFAULT 0.8000,
    sequential_testing_enabled BOOLEAN NOT NULL DEFAULT false,
    sequential_tuning NUMERIC(5,4) DEFAULT 0.0001,
    min_sample_size INTEGER,
    max_duration_days INTEGER,
    activation_metric_id UUID REFERENCES metric(id),   -- optional activation/trigger metric
    exclusion_group_id UUID REFERENCES exclusion_group(id),
    owner_id UUID REFERENCES app_user(id),
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

CREATE TABLE experiment_variation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    key VARCHAR(100) NOT NULL,            -- e.g., 'control', 'variation-1', 'variation-2'
    name VARCHAR(255) NOT NULL,
    description TEXT,
    value TEXT,                           -- JSON-encoded variation value (maps to flag value)
    weight INTEGER NOT NULL DEFAULT 0,    -- traffic allocation in basis points (0-10000)
    is_control BOOLEAN NOT NULL DEFAULT false,
    sort_order INTEGER NOT NULL DEFAULT 0,
    UNIQUE (experiment_id, key)
);

-- Many-to-many: which metrics does each experiment track?
CREATE TABLE experiment_metric (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    metric_id UUID NOT NULL REFERENCES metric(id),
    role VARCHAR(20) NOT NULL DEFAULT 'secondary',  -- 'primary', 'secondary', 'guardrail'
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, metric_id)
);

-- Per-environment experiment activation
CREATE TABLE experiment_environment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    environment_id UUID NOT NULL REFERENCES environment(id),
    enabled BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, environment_id)
);

CREATE INDEX idx_experiment_project ON experiment(project_id);
CREATE INDEX idx_experiment_status ON experiment(project_id, status);
CREATE INDEX idx_experiment_flag ON experiment(feature_flag_id);
CREATE INDEX idx_experiment_variation ON experiment_variation(experiment_id);
CREATE INDEX idx_experiment_metric ON experiment_metric(experiment_id);
CREATE INDEX idx_experiment_metric_role ON experiment_metric(experiment_id, role);
```

---

## Mutual Exclusion Groups

```sql
CREATE TABLE exclusion_group (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Each experiment in a group gets a slice of the traffic namespace
CREATE TABLE exclusion_group_slot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    exclusion_group_id UUID NOT NULL REFERENCES exclusion_group(id) ON DELETE CASCADE,
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    range_start INTEGER NOT NULL,         -- 0-10000 basis points
    range_end INTEGER NOT NULL,           -- 0-10000 basis points
    CHECK (range_start >= 0 AND range_end <= 10000 AND range_start < range_end),
    UNIQUE (exclusion_group_id, experiment_id)
);

CREATE INDEX idx_exclusion_group_project ON exclusion_group(project_id);
```

---

## Experiment Results & Snapshots

```sql
CREATE TABLE experiment_snapshot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    run_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'success',  -- 'success', 'error', 'partial'
    error_message TEXT,
    query_language VARCHAR(20),           -- 'bigquery_sql', 'snowflake_sql', etc.
    queries_run JSONB,                    -- actual SQL queries executed for auditability
    total_users INTEGER,
    date_range_start TIMESTAMPTZ,
    date_range_end TIMESTAMPTZ,
    srm_p_value NUMERIC(10,8),           -- Sample Ratio Mismatch p-value
    srm_detected BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Per-variation, per-metric results within a snapshot
CREATE TABLE experiment_result (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_id UUID NOT NULL REFERENCES experiment_snapshot(id) ON DELETE CASCADE,
    variation_id UUID NOT NULL REFERENCES experiment_variation(id),
    metric_id UUID NOT NULL REFERENCES metric(id),
    users INTEGER NOT NULL DEFAULT 0,
    mean NUMERIC,
    stddev NUMERIC,
    cr NUMERIC,                           -- conversion rate (for binomial metrics)
    value NUMERIC,                        -- aggregated metric value
    ci_lower NUMERIC,                     -- confidence interval lower bound
    ci_upper NUMERIC,                     -- confidence interval upper bound
    p_value NUMERIC(10,8),
    uplift_percent NUMERIC,               -- relative change vs. control
    uplift_mean NUMERIC,                  -- absolute change vs. control
    chance_to_beat_control NUMERIC(7,4),  -- Bayesian: posterior probability
    is_statistically_significant BOOLEAN,
    risk NUMERIC,                         -- Bayesian: expected loss
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- SRM per-variation counts
CREATE TABLE experiment_srm_check (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_id UUID NOT NULL REFERENCES experiment_snapshot(id) ON DELETE CASCADE,
    variation_id UUID NOT NULL REFERENCES experiment_variation(id),
    expected_weight INTEGER NOT NULL,     -- expected basis points
    observed_count INTEGER NOT NULL,
    expected_count INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_experiment ON experiment_snapshot(experiment_id, run_at DESC);
CREATE INDEX idx_result_snapshot ON experiment_result(snapshot_id);
CREATE INDEX idx_result_variation ON experiment_result(variation_id);
CREATE INDEX idx_result_metric ON experiment_result(metric_id);
```

---

## Multi-Armed Bandit

```sql
CREATE TABLE bandit_config (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    algorithm VARCHAR(50) NOT NULL DEFAULT 'thompson_sampling',  -- 'thompson_sampling', 'epsilon_greedy', 'ucb1'
    update_cadence_minutes INTEGER NOT NULL DEFAULT 60,  -- how often to recalculate weights
    burn_in_period_hours INTEGER NOT NULL DEFAULT 24,     -- minimum time before adaptive allocation starts
    exploration_rate NUMERIC(5,4) DEFAULT 0.1000,         -- epsilon for epsilon_greedy
    min_weight INTEGER NOT NULL DEFAULT 100,              -- minimum basis points per variation (prevents 0%)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id)
);

CREATE TABLE bandit_weight_update (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bandit_config_id UUID NOT NULL REFERENCES bandit_config(id) ON DELETE CASCADE,
    variation_id UUID NOT NULL REFERENCES experiment_variation(id),
    new_weight INTEGER NOT NULL,          -- updated basis points
    reward_estimate NUMERIC,              -- estimated reward at time of update
    calculated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bandit_weight_config ON bandit_weight_update(bandit_config_id, calculated_at DESC);
```

---

## SDK Configuration & API Keys

```sql
CREATE TABLE sdk_connection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    environment_id UUID NOT NULL REFERENCES environment(id),
    name VARCHAR(255) NOT NULL,
    key_type VARCHAR(20) NOT NULL DEFAULT 'client',  -- 'client', 'server', 'mobile'
    api_key VARCHAR(255) NOT NULL UNIQUE,             -- the actual SDK key
    encrypted_server_secret VARCHAR(500),              -- for server-side SDKs
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Cached SDK payload (regenerated on flag/experiment changes)
CREATE TABLE sdk_payload_cache (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sdk_connection_id UUID NOT NULL REFERENCES sdk_connection(id) ON DELETE CASCADE,
    payload JSONB NOT NULL,               -- full feature definitions + experiment configs
    version BIGINT NOT NULL DEFAULT 1,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (sdk_connection_id)
);

CREATE INDEX idx_sdk_conn_project ON sdk_connection(project_id);
CREATE INDEX idx_sdk_conn_key ON sdk_connection(api_key);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id),
    project_id UUID REFERENCES project(id),
    user_id UUID REFERENCES app_user(id),
    entity_type VARCHAR(50) NOT NULL,     -- 'experiment', 'feature_flag', 'metric', 'segment', etc.
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,          -- 'created', 'updated', 'deleted', 'started', 'stopped', 'archived'
    details JSONB,                        -- diff of changes: { "field": { "old": X, "new": Y } }
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at DESC);
```

---

## Webhooks & Integrations

```sql
CREATE TABLE webhook (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    signing_secret VARCHAR(255) NOT NULL,
    events TEXT[] NOT NULL,               -- e.g., '{experiment.started, experiment.completed, flag.updated}'
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification_integration (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,            -- 'slack', 'pagerduty', 'email'
    config_encrypted BYTEA NOT NULL,      -- encrypted channel URL / API key
    events TEXT[] NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Multi-Tenancy | 3 | organization, project, environment |
| User & RBAC | 4 | app_user, role, user_org_membership, user_project_role |
| Data Sources | 3 | data_source, fact_table, assignment_query |
| Metrics | 2 | metric, metric_covariate |
| Segments & Targeting | 3 | segment, segment_rule, segment_condition |
| Feature Flags | 5 | feature_flag, flag_environment, flag_rule, flag_rule_condition, flag_rule_variation, flag_override |
| Experiments | 4 | experiment, experiment_variation, experiment_metric, experiment_environment |
| Mutual Exclusion | 2 | exclusion_group, exclusion_group_slot |
| Results & Stats | 3 | experiment_snapshot, experiment_result, experiment_srm_check |
| Multi-Armed Bandit | 2 | bandit_config, bandit_weight_update |
| SDK & API Keys | 2 | sdk_connection, sdk_payload_cache |
| Audit & Governance | 1 | audit_log |
| Webhooks & Integrations | 2 | webhook, notification_integration |
| **Total** | **~36** | |

---

## Key Design Decisions

1. **Basis-point weights (0-10000) instead of floats** — Using integers for traffic allocation percentages (5000 = 50.00%) avoids floating-point rounding errors that can cause SRM issues. This is critical for statistical validity.

2. **Separate fact_table and assignment_query entities** — Warehouse-native architecture requires explicit separation of "where do we find metric events?" (fact tables) and "where do we find experiment assignments?" (assignment queries). This mirrors GrowthBook's data source model.

3. **Normalised targeting conditions** — While competitors like GrowthBook store conditions as JSONB, this model normalises them into `segment_condition` and `flag_rule_condition` tables. This enables SQL queries like "find all rules that target country='US'" without JSONB containment operators.

4. **Metric covariates as a separate table** — CUPED variance reduction requires pre-experiment covariate data. The `metric_covariate` table explicitly models which metrics serve as covariates for other metrics, with configurable lookback windows.

5. **Snapshot-based results** — Each statistical analysis run creates a `experiment_snapshot` with child `experiment_result` rows. This preserves the full history of how results evolved over time, which is essential for debugging SRM issues and understanding sequential testing behaviour.

6. **Exclusion groups with range-based slots** — Mutual exclusion is implemented via traffic namespace ranges (0-10000), where each experiment claims a contiguous slice. This is the same approach used by Statsig, Optimizely, and Eppo.

7. **Flag overrides separate from rules** — Explicit user-level overrides (for QA, debugging, or customer-specific configurations) are stored in `flag_override` rather than as special-case rules. This simplifies the rule evaluation engine.

8. **Encrypted credentials** — Data source connection parameters and notification integration configs use `BYTEA` columns for encrypted storage, rather than plaintext JSON. The application layer handles encryption/decryption.

9. **OpenFeature-aligned evaluation context** — The targeting attribute names and operator vocabulary (`$eq`, `$in`, `$gt`, etc.) align with the OpenFeature evaluation context specification and MongoDB query syntax, ensuring SDK portability.

10. **Audit log with JSONB diff** — The `details` column captures field-level before/after changes, enabling full reconstruction of any entity's state at any point in time without event sourcing overhead.
