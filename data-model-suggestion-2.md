# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: A/B Testing Platform · Created: 2026-05-11

## Philosophy

This model treats the immutable event log as the single source of truth. Every state change — creating an experiment, modifying traffic allocation, starting/stopping a test, updating a metric definition — is recorded as a domain event in an append-only event store. Read-optimised materialised views (projections) are built from these events to serve the UI, API, and SDK payload generation.

Event sourcing is particularly well-suited to experimentation platforms because the domain is inherently temporal: experiments move through lifecycle stages, traffic allocations shift over time, and statistical results accumulate incrementally. The full audit trail is not an afterthought bolted onto CRUD operations — it is the primary data store. This means regulatory compliance (who changed what, when, and why) comes for free, and temporal queries ("what was the experiment configuration on March 15th?") are answered by replaying events rather than querying snapshot tables.

Real-world systems using this pattern include financial trading platforms (which must reconstruct exact state at any past moment), healthcare record systems (where change history is a legal requirement), and Microsoft's own experimentation platform (which logs every configuration change and exposure as events). The CQRS (Command Query Responsibility Segregation) pattern separates the write path (commands that produce events) from the read path (projections optimised for queries), enabling each to scale independently.

**Best for:** Teams that prioritise complete audit trails, temporal debugging ("why did this experiment behave differently last Tuesday?"), and the ability to replay/reprocess historical experiment data with new statistical methods.

**Trade-offs:**
- (+) Complete, immutable audit trail with zero additional effort — every change is an event
- (+) Temporal queries are first-class: reconstruct any entity's state at any past timestamp
- (+) Event replay enables reprocessing historical experiments with updated statistical engines
- (+) Write path and read path scale independently; event store is append-only (fast writes)
- (+) Natural fit for experiment lifecycle (draft → running → paused → completed)
- (-) Higher implementation complexity; requires event handlers, projections, and eventual consistency handling
- (-) Read model must be kept in sync with event store; eventual consistency can surprise developers
- (-) Debugging requires understanding both the event store and the current projection state
- (-) Event schema evolution (versioning events over time) requires careful planning
- (-) Fewer tables but more moving parts (event processors, projection rebuilders, snapshot stores)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFeature (CNCF) | Evaluation context model informs the event payload structure for `FlagEvaluated` and `ExperimentExposure` events |
| CloudEvents (CNCF) | Event envelope format follows the CloudEvents v1.0 specification (`id`, `source`, `type`, `time`, `data`) |
| OpenTelemetry | Exposure and evaluation events correlate with OTel trace IDs for end-to-end observability |
| JSON Schema (draft 2020-12) | Event `data` payloads are validated against versioned JSON schemas |
| ISO 8601 | All timestamps in events use ISO 8601 with timezone (stored as `TIMESTAMPTZ`) |

---

## Event Store (Core)

The event store is the single source of truth. All other tables are materialised projections that can be rebuilt from events.

```sql
-- Core event store: append-only, immutable
CREATE TABLE domain_event (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,              -- aggregate root ID (experiment, flag, metric, etc.)
    stream_type VARCHAR(50) NOT NULL,     -- 'Experiment', 'FeatureFlag', 'Metric', 'Segment', 'Project'
    event_type VARCHAR(100) NOT NULL,     -- e.g., 'ExperimentCreated', 'VariationAdded', 'TrafficAllocated'
    event_version INTEGER NOT NULL DEFAULT 1,  -- schema version for this event type
    sequence_number BIGINT NOT NULL,      -- monotonically increasing within a stream
    data JSONB NOT NULL,                  -- event payload (see event catalog below)
    metadata JSONB NOT NULL DEFAULT '{}', -- actor, IP, user agent, correlation ID
    -- metadata example: {"actor_id": "uuid", "ip": "1.2.3.4", "correlation_id": "uuid", "source": "api"}
    organization_id UUID NOT NULL,
    project_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Optimistic concurrency: unique constraint on (stream_id, sequence_number) prevents
-- two concurrent commands from appending to the same stream position.

CREATE INDEX idx_event_stream ON domain_event(stream_id, sequence_number);
CREATE INDEX idx_event_type ON domain_event(event_type, created_at);
CREATE INDEX idx_event_org ON domain_event(organization_id, created_at DESC);
CREATE INDEX idx_event_project ON domain_event(project_id, created_at DESC);
CREATE INDEX idx_event_created ON domain_event(created_at);

-- Periodic snapshots to avoid replaying all events for long-lived aggregates
CREATE TABLE aggregate_snapshot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,
    stream_type VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,      -- snapshot is valid up to this sequence number
    state JSONB NOT NULL,                 -- serialised aggregate state at this point
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshot_stream ON aggregate_snapshot(stream_id, sequence_number DESC);
```

### Event Catalog

Below are the key domain events. Each event's `data` field contains a JSON payload specific to the event type.

```
┌─────────────────────────────────────┬───────────────────────────────────────────────────┐
│ Event Type                          │ Key Data Fields                                   │
├─────────────────────────────────────┼───────────────────────────────────────────────────┤
│ ProjectCreated                      │ name, slug, settings                              │
│ ProjectUpdated                      │ changes: {field: {old, new}}                      │
│ EnvironmentCreated                  │ name, slug, sort_order                            │
│                                     │                                                   │
│ FeatureFlagCreated                  │ key, value_type, default_value, description        │
│ FeatureFlagUpdated                  │ changes: {field: {old, new}}                      │
│ FlagEnabledInEnvironment            │ environment_id, enabled                           │
│ FlagTargetingRuleAdded              │ rule: {conditions, variations, sort_order}         │
│ FlagTargetingRuleUpdated            │ rule_index, changes                               │
│ FlagTargetingRuleRemoved            │ rule_index                                        │
│ FlagOverrideSet                     │ context_key, variation_value                      │
│ FlagOverrideRemoved                 │ context_key                                       │
│ FeatureFlagArchived                 │ reason                                            │
│                                     │                                                   │
│ ExperimentCreated                   │ name, slug, hypothesis, type, stats_engine         │
│ ExperimentUpdated                   │ changes: {field: {old, new}}                      │
│ VariationAdded                      │ key, name, value, weight, is_control               │
│ VariationUpdated                    │ variation_key, changes                             │
│ VariationRemoved                    │ variation_key                                      │
│ TrafficAllocated                    │ allocations: [{variation_key, weight}]             │
│ MetricAttached                      │ metric_id, role (primary|secondary|guardrail)      │
│ MetricDetached                      │ metric_id                                         │
│ ExperimentStarted                   │ started_at, environment_ids                       │
│ ExperimentPaused                    │ reason                                            │
│ ExperimentResumed                   │ -                                                  │
│ ExperimentStopped                   │ winner_variation_key, reason                      │
│ ExperimentArchived                  │ reason                                            │
│ ExperimentLinkedToFlag              │ feature_flag_id                                   │
│ ExperimentAddedToExclusionGroup     │ exclusion_group_id, range_start, range_end         │
│                                     │                                                   │
│ MetricCreated                       │ name, slug, type, aggregation, fact_table_id       │
│ MetricUpdated                       │ changes                                           │
│ MetricCovariateAdded                │ covariate_metric_id, lookback_days                │
│ MetricArchived                      │ reason                                            │
│                                     │                                                   │
│ SegmentCreated                      │ name, slug, rules                                 │
│ SegmentUpdated                      │ changes                                           │
│ SegmentArchived                     │ reason                                            │
│                                     │                                                   │
│ DataSourceConnected                 │ name, type, settings (credentials excluded)        │
│ FactTableDefined                    │ name, sql_query, user_id_column, timestamp_column  │
│ AssignmentQueryDefined              │ name, sql_query, id_type                          │
│                                     │                                                   │
│ ResultsComputed                     │ snapshot_id, total_users, srm_p_value, results[]   │
│ SRMDetected                         │ snapshot_id, expected_ratios, observed_ratios      │
│ BanditWeightsUpdated                │ allocations: [{variation_key, new_weight, reward}] │
│                                     │                                                   │
│ SDKConnectionCreated                │ key_type, api_key (hashed)                        │
│ SDKPayloadPublished                 │ version, checksum                                 │
│                                     │                                                   │
│ UserInvited                         │ email, role                                       │
│ UserRoleChanged                     │ user_id, old_role, new_role                        │
│ WebhookConfigured                   │ url, events[]                                     │
└─────────────────────────────────────┴───────────────────────────────────────────────────┘
```

---

## Read Model Projections

These tables are materialised views rebuilt from events. They are the *only* tables queried by the API and UI. They can be dropped and rebuilt at any time.

### Experiment Projection

```sql
CREATE TABLE v_experiment (
    id UUID PRIMARY KEY,                  -- same as stream_id
    project_id UUID NOT NULL,
    name VARCHAR(500) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    hypothesis TEXT,
    type VARCHAR(30) NOT NULL DEFAULT 'ab',
    status VARCHAR(30) NOT NULL DEFAULT 'draft',
    feature_flag_id UUID,
    assignment_query_id UUID,
    hash_attribute VARCHAR(100) NOT NULL DEFAULT 'id',
    stats_engine VARCHAR(20) NOT NULL DEFAULT 'frequentist',
    significance_level NUMERIC(5,4) NOT NULL DEFAULT 0.0500,
    power NUMERIC(5,4) NOT NULL DEFAULT 0.8000,
    sequential_testing_enabled BOOLEAN NOT NULL DEFAULT false,
    targeting_segment_id UUID,
    exclusion_group_id UUID,
    owner_id UUID,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    last_event_sequence BIGINT NOT NULL,  -- for optimistic concurrency on reads
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE v_experiment_variation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES v_experiment(id) ON DELETE CASCADE,
    key VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    value TEXT,
    weight INTEGER NOT NULL DEFAULT 0,
    is_control BOOLEAN NOT NULL DEFAULT false,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE v_experiment_metric (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id UUID NOT NULL REFERENCES v_experiment(id) ON DELETE CASCADE,
    metric_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'secondary'
);

CREATE INDEX idx_v_experiment_project ON v_experiment(project_id);
CREATE INDEX idx_v_experiment_status ON v_experiment(project_id, status);
```

### Feature Flag Projection

```sql
CREATE TABLE v_feature_flag (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    key VARCHAR(255) NOT NULL,
    description TEXT,
    value_type VARCHAR(20) NOT NULL DEFAULT 'boolean',
    default_value TEXT NOT NULL DEFAULT 'false',
    tags TEXT[] DEFAULT '{}',
    owner_id UUID,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE v_flag_environment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    feature_flag_id UUID NOT NULL REFERENCES v_feature_flag(id) ON DELETE CASCADE,
    environment_id UUID NOT NULL,
    enabled BOOLEAN NOT NULL DEFAULT false,
    -- Full targeting configuration materialised as JSONB for fast SDK payload generation
    targeting_rules JSONB NOT NULL DEFAULT '[]',
    -- targeting_rules example:
    -- [
    --   {
    --     "conditions": [{"attribute": "country", "operator": "$in", "value": ["US","CA"]}],
    --     "variations": [{"value": "true", "weight": 8000}, {"value": "false", "weight": 2000}]
    --   }
    -- ]
    overrides JSONB NOT NULL DEFAULT '{}',
    -- overrides example: {"user-123": "true", "user-456": "false"}
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_flag_project ON v_feature_flag(project_id);
CREATE INDEX idx_v_flag_key ON v_feature_flag(project_id, key);
```

### Metric Projection

```sql
CREATE TABLE v_metric (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50) NOT NULL,
    fact_table_id UUID,
    aggregation VARCHAR(50) NOT NULL DEFAULT 'sum',
    value_column VARCHAR(255),
    denominator_metric_id UUID,
    sql_filter TEXT,
    cap_value NUMERIC,
    conversion_window_hours INTEGER DEFAULT 72,
    delay_hours INTEGER DEFAULT 0,
    is_inverse BOOLEAN NOT NULL DEFAULT false,
    min_sample_size INTEGER DEFAULT 100,
    tags TEXT[] DEFAULT '{}',
    owner_id UUID,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    covariates JSONB NOT NULL DEFAULT '[]',
    -- covariates example: [{"metric_id": "uuid", "lookback_days": 7}]
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_metric_project ON v_metric(project_id);
CREATE INDEX idx_v_metric_slug ON v_metric(project_id, slug);
```

### Segment Projection

```sql
CREATE TABLE v_segment (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID,
    -- Rules stored as JSONB since they're read as a unit for evaluation
    rules JSONB NOT NULL DEFAULT '[]',
    -- rules example:
    -- [
    --   {"conditions": [{"attribute": "plan", "operator": "$eq", "value": "enterprise"}]},
    --   {"conditions": [{"attribute": "employees", "operator": "$gte", "value": 500}]}
    -- ]
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_segment_project ON v_segment(project_id);
```

### Results Projection

```sql
CREATE TABLE v_experiment_snapshot (
    id UUID PRIMARY KEY,
    experiment_id UUID NOT NULL,
    run_at TIMESTAMPTZ NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'success',
    error_message TEXT,
    total_users INTEGER,
    date_range_start TIMESTAMPTZ,
    date_range_end TIMESTAMPTZ,
    srm_p_value NUMERIC(10,8),
    srm_detected BOOLEAN NOT NULL DEFAULT false,
    -- Full results embedded as JSONB for fast single-query retrieval
    results JSONB NOT NULL DEFAULT '[]',
    -- results example:
    -- [
    --   {
    --     "variation_key": "control",
    --     "metrics": [
    --       {
    --         "metric_id": "uuid", "users": 5000, "mean": 0.12, "cr": 0.12,
    --         "ci_lower": 0.10, "ci_upper": 0.14, "p_value": null, "is_control": true
    --       }
    --     ]
    --   },
    --   {
    --     "variation_key": "variation-1",
    --     "metrics": [
    --       {
    --         "metric_id": "uuid", "users": 5100, "mean": 0.15, "cr": 0.15,
    --         "ci_lower": 0.13, "ci_upper": 0.17, "p_value": 0.001, "uplift_percent": 25.0,
    --         "is_statistically_significant": true
    --       }
    --     ]
    --   }
    -- ]
    created_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_snapshot_experiment ON v_experiment_snapshot(experiment_id, run_at DESC);
```

---

## Organisation & Infrastructure Projections

```sql
CREATE TABLE v_organization (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    settings JSONB NOT NULL DEFAULT '{}',
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE v_project (
    id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    description TEXT,
    settings JSONB NOT NULL DEFAULT '{}',
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    UNIQUE (organization_id, slug)
);

CREATE TABLE v_environment (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    UNIQUE (project_id, slug)
);

CREATE TABLE v_data_source (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    connection_params_encrypted BYTEA NOT NULL,
    settings JSONB NOT NULL DEFAULT '{}',
    last_event_sequence BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE v_sdk_connection (
    id UUID PRIMARY KEY,
    project_id UUID NOT NULL,
    environment_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    key_type VARCHAR(20) NOT NULL DEFAULT 'client',
    api_key VARCHAR(255) NOT NULL UNIQUE,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

-- SDK payload is a projection materialised from flag + experiment events
CREATE TABLE v_sdk_payload (
    sdk_connection_id UUID PRIMARY KEY,
    payload JSONB NOT NULL,
    version BIGINT NOT NULL DEFAULT 1,
    generated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## User & RBAC (Non-Event-Sourced)

User authentication is CRUD-based, not event-sourced. User identity is an operational concern, not a domain aggregate. However, role changes ARE recorded as events for audit purposes.

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

CREATE TABLE v_user_membership (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL,
    role_name VARCHAR(100) NOT NULL,
    permissions JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, organization_id)
);
```

---

## Projection Tracking

```sql
-- Tracks the last processed event for each projection, enabling incremental rebuilds
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,  -- 'v_experiment', 'v_feature_flag', etc.
    last_processed_event_id UUID NOT NULL,
    last_processed_sequence BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_rebuilding BOOLEAN NOT NULL DEFAULT false,
    error_message TEXT
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE projection_dead_letter (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL,
    projection_name VARCHAR(100) NOT NULL,
    error_message TEXT NOT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Reconstruct experiment state at a specific timestamp

```sql
-- Replay all events for experiment X up to a given timestamp
SELECT
    e.event_type,
    e.data,
    e.metadata,
    e.created_at
FROM domain_event e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.stream_type = 'Experiment'
  AND e.created_at <= '2026-03-15 14:30:00+00'
ORDER BY e.sequence_number ASC;
```

### Find all changes to traffic allocation for an experiment

```sql
SELECT
    e.data->>'allocations' AS allocations,
    e.metadata->>'actor_id' AS changed_by,
    e.created_at
FROM domain_event e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.event_type = 'TrafficAllocated'
ORDER BY e.sequence_number ASC;
```

### Audit trail: all actions by a specific user in the last 7 days

```sql
SELECT
    e.stream_type,
    e.event_type,
    e.stream_id,
    e.data,
    e.created_at
FROM domain_event e
WHERE e.metadata->>'actor_id' = 'user-uuid-here'
  AND e.created_at >= now() - interval '7 days'
ORDER BY e.created_at DESC;
```

### Rebuild a projection from scratch

```sql
-- 1. Mark projection as rebuilding
UPDATE projection_checkpoint
SET is_rebuilding = true
WHERE projection_name = 'v_experiment';

-- 2. Truncate the projection table
TRUNCATE v_experiment CASCADE;

-- 3. Replay all experiment events (application code processes each event)
SELECT * FROM domain_event
WHERE stream_type = 'Experiment'
ORDER BY stream_id, sequence_number;

-- 4. Update checkpoint
UPDATE projection_checkpoint
SET is_rebuilding = false,
    last_processed_event_id = (SELECT id FROM domain_event ORDER BY created_at DESC LIMIT 1),
    last_processed_at = now()
WHERE projection_name = 'v_experiment';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (core) | 2 | domain_event, aggregate_snapshot |
| Experiment Projections | 3 | v_experiment, v_experiment_variation, v_experiment_metric |
| Feature Flag Projections | 2 | v_feature_flag, v_flag_environment |
| Metric Projections | 1 | v_metric |
| Segment Projections | 1 | v_segment |
| Results Projections | 1 | v_experiment_snapshot (results embedded as JSONB) |
| Organisation Projections | 5 | v_organization, v_project, v_environment, v_data_source, v_sdk_connection |
| SDK | 1 | v_sdk_payload |
| User & RBAC | 2 | app_user, v_user_membership |
| Infrastructure | 2 | projection_checkpoint, projection_dead_letter |
| **Total** | **~20** | Plus the event store which replaces the need for audit_log |

---

## Key Design Decisions

1. **Single event store table** — All domain events across all aggregate types go into one `domain_event` table, partitioned by `stream_type`. This simplifies event processing infrastructure and enables cross-aggregate queries (e.g., "what happened in this project today?"). An alternative (one table per aggregate type) was rejected because it complicates global ordering and cross-cutting queries.

2. **CloudEvents-compatible envelope** — The event structure (id, source/stream, type, time, data) aligns with the CNCF CloudEvents specification, making it possible to publish events to external systems (Kafka, webhooks) without transformation.

3. **Projections use `v_` prefix** — All read-model tables are prefixed with `v_` to make it immediately obvious that they are derived, disposable views. Any `v_` table can be dropped and rebuilt from the event store.

4. **JSONB in projections, not in events** — Event `data` payloads use JSONB for flexibility, but the key fields are also indexed. Projections use JSONB selectively (targeting rules, results) where the data is read as a unit and doesn't need relational joins.

5. **Results embedded in snapshot projection** — Unlike the normalized model (Suggestion 1) which has a separate `experiment_result` row per variation per metric, this model embeds all results as JSONB inside `v_experiment_snapshot`. This is because results are always read as a complete set and never joined with other tables.

6. **User auth is NOT event-sourced** — Authentication/session state is operational, not domain-critical. Event-sourcing user passwords or sessions adds complexity without audit value. Role changes ARE events because they affect experiment governance.

7. **Optimistic concurrency via sequence numbers** — The `UNIQUE (stream_id, sequence_number)` constraint on `domain_event` provides optimistic concurrency control. Two concurrent commands that try to append to the same stream at the same position will fail, requiring a retry with the latest state.

8. **Aggregate snapshots for performance** — Long-lived aggregates (experiments with hundreds of events) would be slow to rebuild from events on every read. Periodic snapshots in `aggregate_snapshot` allow loading state from a recent snapshot and replaying only subsequent events.

9. **Projection checkpointing** — The `projection_checkpoint` table tracks which events each projection has processed. This enables incremental updates (only process new events) and reliable recovery after failures. The dead letter queue captures events that fail processing without blocking the pipeline.

10. **Fewer tables, more events** — This model has ~20 tables vs. ~36 in the normalized model. The complexity lives in the event processing pipeline rather than in the schema. Adding a new feature (e.g., AI hypothesis generation) means adding new event types, not new tables — the schema evolves through event versioning rather than DDL migrations.
