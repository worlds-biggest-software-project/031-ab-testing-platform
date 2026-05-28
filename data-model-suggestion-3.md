# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: A/B Testing Platform · Created: 2026-05-11

## Philosophy

This model uses relational tables for core entities that benefit from referential integrity and indexing (experiments, metrics, flags, organisations) while storing inherently variable, nested, or rapidly evolving data in JSONB columns. The key insight is that not all data in an experimentation platform has the same stability profile: an experiment's name and status change infrequently and benefit from typed columns, but targeting rules, SDK payloads, and statistical results are complex nested structures that are always read and written as a unit.

This is the approach GrowthBook actually uses: a MongoDB document store (JSON-native) where each experiment, feature, and metric is a rich document with nested sub-objects for variations, targeting rules, and results. This model adapts that pattern to PostgreSQL, using relational structure where it adds value (foreign keys between experiments and metrics, unique constraints on slugs, indexed status fields) and JSONB where relational normalisation would create join-heavy queries without analytical benefit (targeting conditions, variation definitions, statistical results).

The hybrid approach is particularly well-suited to an A/B testing platform MVP because: (1) targeting rules vary by customer and evolve rapidly — storing them as JSONB avoids schema migrations for every new operator or attribute type; (2) SDK payloads are JSON by nature — the read path generates JSON directly from JSONB columns without ORM serialisation; (3) statistical results are a dense matrix (variations x metrics x statistics) that is always consumed whole, not queried column-by-column.

**Best for:** Teams building an MVP that needs to ship fast, iterate on schema frequently, and serve JSON payloads to SDKs with minimal serialisation overhead. Also ideal when targeting rules and configuration structures will evolve rapidly based on customer feedback.

**Trade-offs:**
- (+) Fastest path to MVP — fewer tables, fewer migrations, faster iteration cycles
- (+) JSONB columns map directly to SDK payload format, reducing serialisation overhead
- (+) Flexible targeting rules: new operators and attributes require no schema changes
- (+) Reduced join complexity: experiment + variations + targeting in one or two queries
- (+) PostgreSQL GIN indexes on JSONB provide acceptable query performance for most access patterns
- (-) JSONB columns bypass foreign key constraints — orphaned references within JSON are possible
- (-) Partial updates to JSONB require reading the full column, modifying in application code, and writing back
- (-) Complex cross-entity queries on JSONB fields (e.g., "find all experiments targeting country=US") require JSONB containment operators that are less intuitive than simple WHERE clauses
- (-) Schema evolution within JSONB is implicit — there is no DDL migration to enforce new required fields
- (-) Data quality depends more heavily on application-layer validation than database constraints

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature (CNCF) | Evaluation context model stored directly as JSONB; targeting conditions use OpenFeature-compatible attribute names |
| MongoDB Query Syntax | Targeting conditions in JSONB use MongoDB-style operators (`$eq`, `$in`, `$gt`, etc.), following GrowthBook's pattern for SDK evaluation |
| JSON Schema (draft 2020-12) | Application-layer validation of JSONB columns uses JSON Schema; `json_schema` column on metric stores the expected shape of custom metric configs |
| ISO 3166-1 | Country codes in geo-targeting conditions within the `conditions` JSONB |
| OpenTelemetry | SDK exposure event structure aligns with OTel span attributes |

---

## Organisation & Multi-Tenancy

```sql
CREATE TABLE organization (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_stats_engine": "frequentist",
    --   "default_significance_level": 0.05,
    --   "default_conversion_window_hours": 72,
    --   "require_hypothesis": true,
    --   "sso_config": {"provider": "okta", "domain": "company.okta.com"}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE project (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    settings JSONB NOT NULL DEFAULT '{}',  -- project-level overrides of org settings
    environments JSONB NOT NULL DEFAULT '[{"slug": "production", "name": "Production"}, {"slug": "staging", "name": "Staging"}]',
    -- Environments stored inline: simpler than a separate table for a small, stable list
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_project_org ON project(organization_id);
```

---

## User & RBAC

```sql
CREATE TABLE app_user (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(255),
    password_hash VARCHAR(255),
    auth_provider VARCHAR(50),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE membership (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organization(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- 'admin', 'experimenter', 'analyst', 'viewer'
    project_roles JSONB NOT NULL DEFAULT '{}',
    -- project_roles example: {"project-uuid-1": "admin", "project-uuid-2": "viewer"}
    -- NULL/missing key = inherit org-level role
    permissions JSONB NOT NULL DEFAULT '[]',
    -- permissions example: ["experiment:create", "experiment:start", "flag:create", "metric:manage"]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, organization_id)
);

CREATE INDEX idx_membership_user ON membership(user_id);
```

---

## Data Source Connections

```sql
CREATE TABLE data_source (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,            -- 'bigquery', 'snowflake', 'databricks', 'redshift', 'postgres', 'clickhouse'
    connection_params_encrypted BYTEA NOT NULL,
    settings JSONB NOT NULL DEFAULT '{}',
    -- settings example (BigQuery):
    -- {
    --   "project": "my-gcp-project",
    --   "dataset": "analytics",
    --   "default_schema": "public"
    -- }
    queries JSONB NOT NULL DEFAULT '{}',
    -- queries example:
    -- {
    --   "assignment": {
    --     "sql": "SELECT user_id, timestamp, experiment_id, variation_id FROM assignments",
    --     "id_type": "user_id"
    --   },
    --   "identifier_joins": [
    --     {"from": "user_id", "to": "anonymous_id", "sql": "SELECT user_id, anonymous_id FROM user_map"}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_source_project ON data_source(project_id);
```

---

## Fact Tables & Metrics

```sql
CREATE TABLE fact_table (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id UUID NOT NULL REFERENCES data_source(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    sql_query TEXT NOT NULL,
    user_id_column VARCHAR(255) NOT NULL,
    timestamp_column VARCHAR(255) NOT NULL,
    id_type VARCHAR(100) NOT NULL DEFAULT 'user_id',
    columns JSONB NOT NULL DEFAULT '[]',
    -- columns example: [{"name": "revenue", "type": "number"}, {"name": "plan", "type": "string"}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE metric (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50) NOT NULL,            -- 'binomial', 'count', 'duration', 'revenue', 'ratio', 'quantile'
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    owner_id UUID REFERENCES app_user(id),
    tags TEXT[] DEFAULT '{}',
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "fact_table_id": "uuid",
    --   "aggregation": "sum",
    --   "value_column": "revenue",
    --   "sql_filter": "event_type = 'purchase'",
    --   "cap_value": 1000,
    --   "conversion_window_hours": 72,
    --   "delay_hours": 0,
    --   "is_inverse": false,
    --   "min_sample_size": 100,
    --   "denominator_metric_id": null,
    --   "covariates": [
    --     {"metric_id": "uuid", "lookback_days": 7}
    --   ]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

CREATE INDEX idx_metric_project ON metric(project_id);
CREATE INDEX idx_metric_tags ON metric USING GIN(tags);
CREATE INDEX idx_fact_table_source ON fact_table(data_source_id);
```

---

## Segments

```sql
CREATE TABLE segment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES app_user(id),
    rules JSONB NOT NULL DEFAULT '[]',
    -- rules example (OR between rules, AND within conditions):
    -- [
    --   {
    --     "conditions": [
    --       {"attribute": "country", "operator": "$in", "value": ["US", "CA", "GB"]},
    --       {"attribute": "plan", "operator": "$eq", "value": "enterprise"}
    --     ]
    --   },
    --   {
    --     "conditions": [
    --       {"attribute": "employee_count", "operator": "$gte", "value": 500}
    --     ]
    --   }
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

CREATE INDEX idx_segment_project ON segment(project_id);
```

---

## Feature Flags

```sql
CREATE TABLE feature_flag (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,
    description TEXT,
    value_type VARCHAR(20) NOT NULL DEFAULT 'boolean',
    default_value TEXT NOT NULL DEFAULT 'false',
    tags TEXT[] DEFAULT '{}',
    owner_id UUID REFERENCES app_user(id),
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- Per-environment configuration stored as JSONB keyed by environment slug
    environment_settings JSONB NOT NULL DEFAULT '{}',
    -- environment_settings example:
    -- {
    --   "production": {
    --     "enabled": true,
    --     "rules": [
    --       {
    --         "description": "Beta users get new checkout",
    --         "type": "targeting",
    --         "enabled": true,
    --         "conditions": [
    --           {"attribute": "beta_user", "operator": "$eq", "value": true}
    --         ],
    --         "variations": [
    --           {"value": "true", "weight": 10000}
    --         ]
    --       },
    --       {
    --         "description": "10% rollout to everyone else",
    --         "type": "rollout",
    --         "enabled": true,
    --         "conditions": [],
    --         "variations": [
    --           {"value": "true", "weight": 1000},
    --           {"value": "false", "weight": 9000}
    --         ]
    --       }
    --     ],
    --     "overrides": {
    --       "user-123": "true",
    --       "user-456": "false"
    --     }
    --   },
    --   "staging": {
    --     "enabled": true,
    --     "rules": [],
    --     "overrides": {}
    --   }
    -- }
    revision INTEGER NOT NULL DEFAULT 1,  -- incremented on every update for optimistic concurrency
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, key)
);

CREATE INDEX idx_flag_project ON feature_flag(project_id);
CREATE INDEX idx_flag_key ON feature_flag(project_id, key);
CREATE INDEX idx_flag_tags ON feature_flag USING GIN(tags);
CREATE INDEX idx_flag_status ON feature_flag(project_id, status);
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
    type VARCHAR(30) NOT NULL DEFAULT 'ab',
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    feature_flag_id UUID REFERENCES feature_flag(id),
    owner_id UUID REFERENCES app_user(id),

    -- Variations stored inline as JSONB (always read/written as a set)
    variations JSONB NOT NULL DEFAULT '[]',
    -- variations example:
    -- [
    --   {"key": "control", "name": "Control", "value": "false", "weight": 5000, "is_control": true},
    --   {"key": "variation-1", "name": "New Checkout", "value": "true", "weight": 5000, "is_control": false}
    -- ]

    -- Metric assignments with roles
    metric_assignments JSONB NOT NULL DEFAULT '[]',
    -- metric_assignments example:
    -- [
    --   {"metric_id": "uuid-1", "role": "primary"},
    --   {"metric_id": "uuid-2", "role": "guardrail"},
    --   {"metric_id": "uuid-3", "role": "secondary"}
    -- ]

    -- Targeting & allocation configuration
    targeting JSONB NOT NULL DEFAULT '{}',
    -- targeting example:
    -- {
    --   "segment_id": "uuid",
    --   "hash_attribute": "id",
    --   "hash_version": 2,
    --   "assignment_query_id": "uuid",
    --   "exclusion_group": {
    --     "id": "uuid",
    --     "range_start": 0,
    --     "range_end": 5000
    --   },
    --   "environments": {
    --     "production": {"enabled": true},
    --     "staging": {"enabled": false}
    --   }
    -- }

    -- Statistical configuration
    stats_config JSONB NOT NULL DEFAULT '{}',
    -- stats_config example:
    -- {
    --   "engine": "frequentist",
    --   "significance_level": 0.05,
    --   "power": 0.80,
    --   "sequential_testing": {
    --     "enabled": true,
    --     "tuning_parameter": 0.0001
    --   },
    --   "min_sample_size": 1000,
    --   "max_duration_days": 30,
    --   "activation_metric_id": null
    -- }

    -- Bandit configuration (NULL if not a bandit experiment)
    bandit_config JSONB,
    -- bandit_config example:
    -- {
    --   "algorithm": "thompson_sampling",
    --   "update_cadence_minutes": 60,
    --   "burn_in_period_hours": 24,
    --   "exploration_rate": 0.10,
    --   "min_weight": 100
    -- }

    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    revision INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, slug)
);

CREATE INDEX idx_experiment_project ON experiment(project_id);
CREATE INDEX idx_experiment_status ON experiment(project_id, status);
CREATE INDEX idx_experiment_flag ON experiment(feature_flag_id);
CREATE INDEX idx_experiment_owner ON experiment(owner_id);

-- GIN index for querying experiments by metric assignment
CREATE INDEX idx_experiment_metrics ON experiment USING GIN(metric_assignments jsonb_path_ops);
```

### Query: Find all experiments using a specific metric

```sql
SELECT id, name, status
FROM experiment
WHERE metric_assignments @> '[{"metric_id": "target-metric-uuid"}]'::jsonb;
```

---

## Mutual Exclusion Groups

```sql
CREATE TABLE exclusion_group (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    -- Slot allocations tracked here (denormalised for fast reads)
    slots JSONB NOT NULL DEFAULT '[]',
    -- slots example:
    -- [
    --   {"experiment_id": "uuid-1", "range_start": 0, "range_end": 3000},
    --   {"experiment_id": "uuid-2", "range_start": 3000, "range_end": 6000},
    --   {"experiment_id": "uuid-3", "range_start": 6000, "range_end": 10000}
    -- ]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exclusion_group_project ON exclusion_group(project_id);
```

---

## Experiment Results

```sql
CREATE TABLE experiment_snapshot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES experiment(id) ON DELETE CASCADE,
    run_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status VARCHAR(20) NOT NULL DEFAULT 'success',
    error_message TEXT,

    -- Summary statistics
    summary JSONB NOT NULL DEFAULT '{}',
    -- summary example:
    -- {
    --   "total_users": 15200,
    --   "date_range": {"start": "2026-03-01", "end": "2026-03-15"},
    --   "srm": {
    --     "p_value": 0.82,
    --     "detected": false,
    --     "variation_counts": [
    --       {"key": "control", "expected": 7600, "observed": 7550},
    --       {"key": "variation-1", "expected": 7600, "observed": 7650}
    --     ]
    --   },
    --   "queries_run": ["SELECT ...", "SELECT ..."]
    -- }

    -- Per-variation, per-metric results
    results JSONB NOT NULL DEFAULT '[]',
    -- results example:
    -- [
    --   {
    --     "variation_key": "control",
    --     "users": 7550,
    --     "metrics": {
    --       "conversion-rate": {
    --         "mean": 0.121, "stddev": 0.326, "cr": 0.121,
    --         "ci_lower": null, "ci_upper": null,
    --         "p_value": null, "is_control": true
    --       },
    --       "revenue-per-user": {
    --         "mean": 12.50, "stddev": 45.3, "value": 94375.0,
    --         "ci_lower": null, "ci_upper": null,
    --         "p_value": null, "is_control": true
    --       }
    --     }
    --   },
    --   {
    --     "variation_key": "variation-1",
    --     "users": 7650,
    --     "metrics": {
    --       "conversion-rate": {
    --         "mean": 0.142, "stddev": 0.349, "cr": 0.142,
    --         "ci_lower": 0.005, "ci_upper": 0.037,
    --         "p_value": 0.0023, "uplift_percent": 17.4,
    --         "chance_to_beat_control": 0.9977,
    --         "is_statistically_significant": true
    --       },
    --       "revenue-per-user": {
    --         "mean": 14.10, "stddev": 48.1, "value": 107865.0,
    --         "ci_lower": 0.40, "ci_upper": 2.80,
    --         "p_value": 0.0089, "uplift_percent": 12.8,
    --         "is_statistically_significant": true
    --       }
    --     }
    --   }
    -- ]

    -- Bandit weight updates (if applicable)
    bandit_updates JSONB,
    -- bandit_updates example:
    -- [
    --   {"variation_key": "control", "new_weight": 3000, "reward_estimate": 0.12},
    --   {"variation_key": "variation-1", "new_weight": 7000, "reward_estimate": 0.15}
    -- ]

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_experiment ON experiment_snapshot(experiment_id, run_at DESC);
```

---

## SDK Configuration & API Keys

```sql
CREATE TABLE sdk_connection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    environment VARCHAR(100) NOT NULL,    -- matches environment slug in project.environments
    name VARCHAR(255) NOT NULL,
    key_type VARCHAR(20) NOT NULL DEFAULT 'client',
    api_key VARCHAR(255) NOT NULL UNIQUE,
    encrypted_server_secret VARCHAR(500),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Cached SDK payload (regenerated on flag/experiment changes)
CREATE TABLE sdk_payload_cache (
    sdk_connection_id UUID PRIMARY KEY REFERENCES sdk_connection(id) ON DELETE CASCADE,
    payload JSONB NOT NULL,               -- full feature definitions + experiment configs
    version BIGINT NOT NULL DEFAULT 1,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sdk_conn_project ON sdk_connection(project_id);
CREATE INDEX idx_sdk_conn_key ON sdk_connection(api_key);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    project_id UUID,
    user_id UUID,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    details JSONB,                        -- diff: {"field": {"old": X, "new": Y}} or full previous state
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organization_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
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
    events TEXT[] NOT NULL,
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "headers": {"X-Custom-Header": "value"},
    --   "slack_channel": "#experiments",
    --   "notification_types": ["slack", "webhook"]
    -- }
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example: SDK Payload Generation

The hybrid model makes SDK payload generation particularly efficient because flag configurations are already stored as JSON:

```sql
-- Generate SDK payload for a client SDK connection
SELECT jsonb_build_object(
    'features', (
        SELECT jsonb_object_agg(
            ff.key,
            jsonb_build_object(
                'defaultValue', ff.default_value::jsonb,
                'rules', COALESCE(
                    ff.environment_settings->sc.environment->'rules',
                    '[]'::jsonb
                )
            )
        )
        FROM feature_flag ff
        WHERE ff.project_id = sc.project_id
          AND ff.status = 'active'
          AND (ff.environment_settings->sc.environment->>'enabled')::boolean = true
    ),
    'experiments', (
        SELECT jsonb_agg(
            jsonb_build_object(
                'key', e.slug,
                'variations', e.variations,
                'weights', (
                    SELECT jsonb_agg(v->>'weight')
                    FROM jsonb_array_elements(e.variations) v
                ),
                'hashAttribute', e.targeting->>'hash_attribute',
                'hashVersion', (e.targeting->>'hash_version')::int
            )
        )
        FROM experiment e
        WHERE e.project_id = sc.project_id
          AND e.status = 'running'
          AND (e.targeting->'environments'->sc.environment->>'enabled')::boolean = true
    )
) AS payload
FROM sdk_connection sc
WHERE sc.api_key = 'sdk-client-key-here';
```

---

## Example: Cross-Entity JSONB Queries

### Find experiments with SRM issues

```sql
SELECT e.name, e.status, es.run_at,
       es.summary->'srm'->>'p_value' AS srm_p_value
FROM experiment_snapshot es
JOIN experiment e ON e.id = es.experiment_id
WHERE (es.summary->'srm'->>'detected')::boolean = true
ORDER BY es.run_at DESC;
```

### Find all flags targeting a specific country

```sql
SELECT key, environment_settings
FROM feature_flag
WHERE environment_settings @? '$.*.rules[*].conditions[*] ? (@.attribute == "country" && @.value[*] == "US")';
```

### Get metric impact across all experiments

```sql
SELECT
    e.name AS experiment_name,
    e.status,
    r->>'variation_key' AS variation,
    (r->'metrics'->m.slug->>'uplift_percent')::numeric AS uplift_pct,
    (r->'metrics'->m.slug->>'p_value')::numeric AS p_value
FROM experiment e
JOIN experiment_snapshot es ON es.experiment_id = e.id
CROSS JOIN LATERAL jsonb_array_elements(es.results) r
JOIN metric m ON m.id = (
    SELECT (ma->>'metric_id')::uuid
    FROM jsonb_array_elements(e.metric_assignments) ma
    WHERE ma->>'role' = 'primary'
    LIMIT 1
)
WHERE e.project_id = 'target-project-uuid'
  AND es.run_at = (
      SELECT MAX(run_at) FROM experiment_snapshot
      WHERE experiment_id = e.id AND status = 'success'
  )
ORDER BY uplift_pct DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Multi-Tenancy | 2 | organization, project (environments embedded) |
| User & RBAC | 2 | app_user, membership (roles + project_roles in JSONB) |
| Data Sources | 2 | data_source (queries embedded), fact_table |
| Metrics | 1 | metric (config, covariates in JSONB) |
| Segments | 1 | segment (rules in JSONB) |
| Feature Flags | 1 | feature_flag (env settings, rules, overrides all in JSONB) |
| Experiments | 1 | experiment (variations, metrics, targeting, stats config all in JSONB) |
| Mutual Exclusion | 1 | exclusion_group (slots in JSONB) |
| Results | 1 | experiment_snapshot (summary + results in JSONB) |
| SDK & API Keys | 2 | sdk_connection, sdk_payload_cache |
| Audit & Governance | 1 | audit_log |
| Webhooks | 1 | webhook |
| **Total** | **~16** | Significantly fewer than normalized model due to JSONB consolidation |

---

## Key Design Decisions

1. **Environments embedded in project** — Environments are a short, stable list (production, staging, development) that rarely changes. Embedding them as JSONB in the `project` table eliminates a join and simplifies project setup. Flag and experiment environment configs reference environment slugs as JSONB keys.

2. **Variations embedded in experiment** — Variations are always created, read, and updated as a set. They are never queried independently (e.g., "find all variations named 'control'" is not a real use case). Embedding them as JSONB eliminates a table and a join without losing query capability.

3. **Metric config as JSONB** — Metric configuration (aggregation type, value column, cap value, conversion window, covariates) varies significantly by metric type (binomial vs. ratio vs. quantile). A JSONB `config` column avoids adding nullable columns for every metric type's specific settings. The core queryable fields (name, slug, type, status, tags) remain relational.

4. **Feature flag environment_settings as JSONB** — This is the most significant JSONB usage in the model. Each flag's per-environment configuration (enabled state, targeting rules, percentage rollouts, overrides) is stored as a nested JSONB structure keyed by environment slug. This mirrors GrowthBook's MongoDB document structure and makes SDK payload generation a near-direct JSON extraction.

5. **Results as JSONB within snapshots** — Statistical results are a dense matrix (variations x metrics x statistics) that is always consumed as a complete unit in the results UI. Normalising this into separate rows (as in Suggestion 1) adds ~3 joins and significant row counts for multi-variant, multi-metric experiments. JSONB keeps it as a single read.

6. **Revision-based optimistic concurrency** — The `revision` column on experiments and flags provides optimistic concurrency control. API updates must include the current revision; the UPDATE statement uses `WHERE revision = :expected_revision` and the application checks for zero affected rows.

7. **Relational foreign keys where they matter** — Despite heavy JSONB usage, the model maintains FK relationships between core entities: experiment → project, experiment → feature_flag, metric → project, data_source → project. These prevent the most damaging data integrity issues (orphaned experiments, metrics referencing deleted projects).

8. **GIN indexes for JSONB queries** — The `jsonb_path_ops` GIN index on `metric_assignments` enables efficient containment queries for finding experiments by metric. Similar indexes can be added for other JSONB query patterns as they emerge.

9. **Single experiment table replaces 4 normalised tables** — The normalised model (Suggestion 1) uses `experiment`, `experiment_variation`, `experiment_metric`, and `experiment_environment` (4 tables with junction rows). This model consolidates all four into one `experiment` table with JSONB columns, reducing table count and write complexity.

10. **Application-layer validation is essential** — With JSONB columns, the database cannot enforce field presence, type constraints, or referential integrity within JSON structures. The application must validate JSONB payloads against JSON Schemas before writes. This is a conscious trade-off: faster iteration at the cost of more application-layer responsibility.
