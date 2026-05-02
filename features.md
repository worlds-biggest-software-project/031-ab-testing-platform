# A/B Testing Platform — Feature & Functionality Survey

> Candidate #31 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| GrowthBook | Open Source / Commercial | MIT (core); GrowthBook Enterprise Licence (premium features) | https://www.growthbook.io |
| PostHog | Open Source / Commercial | MIT (self-hosted); commercial cloud | https://posthog.com |
| Statsig | Commercial (OpenAI-owned) | Proprietary SaaS; usage-based pricing | https://www.statsig.com |
| LaunchDarkly | Commercial | Proprietary SaaS; per-seat pricing | https://launchdarkly.com |
| Optimizely | Commercial | Proprietary SaaS; annual enterprise contracts | https://www.optimizely.com |
| VWO (Visual Website Optimizer) | Commercial | Proprietary SaaS; traffic-based pricing | https://vwo.com |
| Amplitude Experiment | Commercial | Proprietary SaaS; bundled with Amplitude analytics | https://amplitude.com |
| AB Tasty | Commercial | Proprietary SaaS; custom enterprise pricing | https://www.abtasty.com |
| Kameleoon | Commercial | Proprietary SaaS; custom enterprise pricing | https://www.kameleoon.com |
| Convert | Commercial | Proprietary SaaS; traffic-based pricing | https://www.convert.com |

---

## Feature Analysis by Solution

### GrowthBook

**Core features**
- Warehouse-native architecture: queries data directly from BigQuery, Snowflake, Databricks, Redshift, and 8 other sources without a separate event pipeline
- 24 SDKs covering React, Python, Android, iOS, Go, Ruby, PHP, and more
- Frequentist and Bayesian statistics engines selectable per experiment
- CUPED variance reduction (covariate adjustment using pre-experiment data)
- Sequential testing (SPRT) for valid early stopping
- Sample Ratio Mismatch (SRM) detection built into every experiment
- Multi-armed bandit / adaptive allocation experiments
- Feature flags with percentage rollouts, targeting rules, and override lists
- Metrics governance: standardised metric registry shared across experiments

**Differentiating features**
- True open-source MIT core — self-hostable with full feature access, not a "lite" version
- Post-stratification variance reduction (beyond CUPED, using post-experiment covariates)
- Bandit experiments natively integrated alongside standard A/B tests
- Transparent SQL: every statistical calculation is exposed as inspectable SQL

**UX patterns**
- Single-page experiment creation wizard covering hypothesis, metrics, targeting, and traffic allocation
- Results page displays confidence intervals, p-values, and Bayesian posteriors side-by-side
- Modular design allows starting simple (feature flags only) and adding metrics/stats later
- GitHub-style metric registry for centralised governance

**Integration points**
- 11+ data warehouse connectors (BigQuery, Snowflake, Databricks, Redshift, ClickHouse, PostgreSQL, MySQL, Presto, Athena, Mixpanel, Google Analytics)
- REST API and webhooks for CI/CD pipeline integration
- Slack notifications on experiment results
- OpenFeature (CNCF) standard support for SDK portability

**Known gaps**
- Cloud-hosted managed offering has a smaller ecosystem compared to Optimizely or LaunchDarkly
- UI polish is less refined than commercial-only competitors
- No built-in session replay or heatmap integration
- Multi-experiment interaction detection (experiment collision) not automated

**Licence / IP notes**
- MIT licence for the core product; several `/enterprise` directories carry the GrowthBook Enterprise Licence (source-available, not OSI-approved)
- No known patent encumbrances; CUPED technique is from Microsoft Research (2013) and is unpatented industry-standard practice

---

### PostHog

**Core features**
- All-in-one platform: product analytics, session replay, feature flags, A/B experiments, surveys, and CDP in a single deployment
- Feature flags with percentage rollouts, user/group targeting, cohort targeting, and bootstrapping to eliminate page-load flicker
- Experiments built on top of feature flags with automatic variant exposure tracking
- Local evaluation SDK reduces flag-check latency from ~500 ms to under 50 ms
- Results filterable by any funnel, trend, path, or session replay directly from experiment view
- Free tier: 1 million analytics events, 5,000 session recordings, and 1 million flag requests per month
- Self-host on own infrastructure for full data sovereignty

**Differentiating features**
- Seamless path from "rolling out a feature" to "running an experiment" — flip a flag into an experiment without code changes
- Session replays filterable by experiment variant, enabling qualitative diagnosis of quantitative results
- LLM-powered chat interface (2025–2026) allowing natural language queries across analytics data
- AI session insights automatically surfacing user friction points

**UX patterns**
- Single unified interface for analytics, flags, and experiments reduces context switching
- Event auto-capture reduces need for manual instrumentation
- Cohort builder allows defining experiment audiences from existing analytics segments

**Integration points**
- CDPs, data warehouses (Snowflake, BigQuery) via pipeline destinations
- 52KB+ JavaScript SDK with auto-capture; smaller SDKs for mobile and backend
- REST API and webhook support
- Reverse proxy deployment option for first-party data collection

**Known gaps**
- A/B testing is not the primary focus; statistical rigour (SRM checks, CUPED) less emphasised than GrowthBook or Statsig
- Heavy self-hosted stack (ClickHouse, Kafka, Redis) requiring significant infrastructure management
- 52KB+ SDK size is a meaningful page-weight concern for performance-sensitive applications
- Privacy posture weaker than dedicated privacy-first tools; EU hosting requires explicit configuration

**Licence / IP notes**
- MIT licence for self-hosted community edition
- No known patent encumbrances

---

### Statsig

**Core features**
- Unified platform: feature gates, dynamic config, experiments, layers, and product analytics in one system
- CUPED variance reduction (Statsig claims 30–50% sensitivity improvement, enabling smaller sample sizes or shorter run times)
- CURE (Controlled-experiment Using Regression Estimates): extended CUPED allowing arbitrary covariate data for regression adjustment
- Sequential Probability Ratio Test (SPRT) for valid early stopping without inflating false-positive rates
- Inline Power Analysis (launched February 2026): run and view power analysis directly on the experiment setup page with recommended duration
- AI Copilot features (2025): AI-powered assistance for experiment setup, interpretation, and flagging
- Knowledge Graph (2026): connects codebase, feature gates, experiments, users, and metrics for richer cross-cutting context
- Hyperparameter optimisation: systematic testing of model hyperparameter combinations to find best-performing variants

**Differentiating features**
- Knowledge Graph linking code, flags, experiments, and metrics is unique among mainstream platforms
- Inline Power Analysis embedded in experiment setup (not a separate tool) reduces pre-experiment friction
- CURE (extended CUPED with arbitrary covariates) is more flexible than standard CUPED implementations
- Acquired by OpenAI in 2024/2025 — platform benefits from AI investment but raises strategic independence concerns

**UX patterns**
- Guided experiment setup with inline power analysis and recommended duration
- AI Copilot overlays context and recommendations throughout the experiment lifecycle
- Unified metrics dashboard across experiments and product analytics

**Integration points**
- Usage-based event ingestion (SDKs for web, mobile, server-side)
- Data warehouse export (BigQuery, Snowflake, Redshift)
- Slack and PagerDuty alerts
- REST API for programmatic flag and experiment management

**Known gaps**
- Cloud-only: no self-hosting option, which is a blocker for regulated industries and data-sovereign deployments
- Cost escalates significantly at high event volumes
- OpenAI acquisition raises concerns about long-term product independence and pricing trajectory
- No warehouse-native query path: requires ingesting events into Statsig's own pipeline

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- No known patent encumbrances; SPRT and CUPED are published academic techniques

---

### LaunchDarkly

**Core features**
- Industry-leading feature flag infrastructure: real-time flag evaluation at sub-millisecond latency via a streaming connection
- Four platform pillars as of 2025–2026: Release Management, Observability & Monitoring, Analytics & Experimentation, and AI Configs
- A/B and multivariate experiments measuring feature impact on business metrics
- AI Configs: stores and manages LLM prompt configurations, models, and parameters as feature flags — a new category enabling A/B testing of AI model variants
- Targeting by user attributes, segments, cohorts, or percentage rollouts
- Audit log and change history for every flag modification
- OpenFeature (CNCF standard) SDK support for portability

**Differentiating features**
- AI Configs represent a novel use case: treating LLM prompts and model selections as feature flags, enabling safe rollout and experimentation of AI features
- Streaming SDK architecture provides near-instant flag propagation (sub-second) globally
- Recognised as a G2 Leader in Feature Management (Winter 2025) with the largest enterprise customer base in the category

**UX patterns**
- Flag creation wizard with context-aware targeting rule builder
- Release pipeline view for progressive delivery across environments
- AI Configs editor with model/prompt comparison capability

**Integration points**
- 100+ SDK integrations (web, mobile, server-side, edge)
- OpenFeature standard support
- Integrations with Datadog, New Relic, Splunk, Segment, Amplitude, Jira, Slack
- Terraform provider for infrastructure-as-code flag management

**Known gaps**
- No self-hosting option; cloud-only architecture is a hard constraint
- Experimentation is secondary to flags: statistical engine less advanced than GrowthBook or Statsig (no CUPED, no sequential testing as of research date)
- Cost scales steeply with seat count; can become expensive for large engineering organisations
- Warehouse-native querying not available; relies on LaunchDarkly's own event pipeline

**Licence / IP notes**
- Proprietary SaaS; no open-source core
- No known patent encumbrances

---

### Optimizely

**Core features**
- Full-stack experimentation (Feature Experimentation) and web experimentation (visual editor + WYSIWYG) as separate but integrated products
- Sequential testing, CUPED variance reduction, and ratio metrics for custom KPIs
- Global Holdouts (2026): designate a user holdback group untouched by any experiment or feature rollout, enabling clean long-run causal measurement
- Warehouse-Native Experimentation Analytics (GA as of 2025): query experiment data directly from the warehouse without pushing events to Optimizely
- Granular roles and permissions for experiment governance in large organisations
- AI Variation Development Agent: AI-assisted creation and modification of web experiment variants while preserving brand consistency

**Differentiating features**
- Global Holdouts feature is rare among competitors and enables rigorous measurement of cumulative experiment impact
- MCP server integration (2026): connects Optimizely Experimentation data to Claude, Cursor, and other AI clients via the Model Context Protocol standard
- Opal AI agent for pre-launch experiment review: reviews experiment configuration and flags issues before launch
- Integrated DXP (Digital Experience Platform) spanning CMS, CDP, and experimentation — relevant for enterprise content teams

**UX patterns**
- Separate UIs for web experimentation (marketer-facing) and feature experimentation (developer-facing); both accessible from unified Optimizely account
- Experiment Review agent provides AI-powered pre-launch checklist
- Reporting Metric Impact Report dashboard aggregates lift data across the full experiment portfolio

**Integration points**
- REST API and SDKs for all major languages
- Warehouse-Native Analytics connecting to Snowflake, BigQuery, Databricks
- MCP server for AI client integration (Claude, Cursor, etc.)
- Integrations with Salesforce, Contentful, and major CDPs

**Known gaps**
- Very high cost ($63K–$200K+/year) excludes all but enterprise buyers
- Fragmented UX across acquired products (web experimentation and feature experimentation are distinct legacy codebases)
- Complex onboarding; time-to-first-experiment typically measured in weeks not days
- No open-source option or free tier

**Licence / IP notes**
- Proprietary SaaS
- No known patent encumbrances; company has filed patents on specific DXP optimisation techniques but none directly relevant to A/B testing mechanics

---

### VWO (Visual Website Optimizer)

**Core features**
- No-code WYSIWYG visual editor for creating test variants without engineering involvement
- A/B, multivariate (MVT), and split URL testing
- Heatmaps, session recordings, and scroll maps for qualitative diagnosis
- Funnel analysis and goal tracking via point-and-click tag builder
- Segment Explorer for rapid behavioural pattern analysis
- Personalisation campaigns with AI-driven targeting
- Full-stack SDK testing for mobile apps across all technologies
- VWO Copilot (2025): AI assistant that suggests what to test, generates variations, and surfaces hidden segments

**Differentiating features**
- All-in-one CRO (Conversion Rate Optimisation) suite unifying testing, heatmaps, session replay, and personalisation in a single platform
- VWO Copilot's predictive analytics for ROI tracking represents an unusually integrated AI layer for a CRO-focused tool
- Particularly strong for non-developer (marketing/CRO) teams due to the no-code editor

**UX patterns**
- Marketing-first UX: visual editor and heatmaps foregrounded; statistical configuration backgrounded
- Unified workspace (2025 redesign) integrating insights, testing, and personalisation layers
- No limit on variations, account users, metrics, session recordings, or integrations (plan-dependent)

**Integration points**
- JavaScript snippet for web; SDKs for mobile and server-side
- Integrations with Google Analytics, Mixpanel, HubSpot, Salesforce, and major tag managers
- REST API for programmatic experiment management

**Known gaps**
- Statistical methodology less rigorous than warehouse-native tools: no native CUPED or Sequential testing as of research date
- Pricing is opaque and traffic-based, making costs difficult to predict
- Less suited to engineering/product use cases than GrowthBook or Statsig; primarily a marketer tool
- No open-source option; vendor lock-in with proprietary event pipeline

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- No known patent encumbrances

---

### Amplitude Experiment

**Core features**
- Experimentation layer tightly integrated with Amplitude's product analytics
- Sequential testing and CUPED for advanced statistical rigour
- Feature flags for progressive delivery and experiment targeting
- Web experimentation (visual editor for non-developers)
- Direct segment creation from Amplitude analytics cohorts for experiment targeting
- Results viewable within the Amplitude analytics dashboards for unified reporting

**Differentiating features**
- Deepest integration with user-level analytics cohorts for targeting and analysis among all commercial tools
- Bidirectional data flow: experiment results enrich analytics; analytics cohorts populate experiment segments
- Feature flags, web experimentation, and feature experimentation available as a unified offering

**UX patterns**
- Experiment setup flows from Amplitude's existing analytics interface, reducing context switching for analytics-first teams
- Funnel and retention analysis on experiment results available natively without data export

**Integration points**
- Deep Amplitude Analytics integration (events, cohorts, dashboards)
- Kameleoon event streaming integration announced 2025
- Standard SDK integrations for web and mobile

**Known gaps**
- Requires Amplitude ecosystem buy-in; not suitable as a standalone experimentation tool
- Expensive for dedicated experimentation use when bundled with analytics suite
- Smaller community and third-party integration ecosystem than LaunchDarkly or GrowthBook

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- No known patent encumbrances

---

### AB Tasty

**Core features**
- Visual editor for no-code variant creation targeting marketers and CRO teams
- A/B, split, and multivariate testing
- AI-assisted hypothesis and recommendation engine
- Personalisation campaigns with audience segmentation
- Progressive delivery and feature rollout support
- Server-side and client-side testing options

**Differentiating features**
- Strong personalisation layer natively integrated with A/B testing
- AI recommendation engine suggests optimisation ideas based on behavioural data
- Purpose-built for enterprise marketing and e-commerce use cases

**UX patterns**
- Marketing-led UX with campaign-style experiment management
- AI recommendations surfaced in the dashboard as actionable items

**Integration points**
- Tag manager and CMS integrations
- CDP connectors (Salesforce, Segment)
- REST API for programmatic access

**Known gaps**
- Limited utility for engineering and product teams needing feature flag granularity
- Opaque pricing (custom enterprise-only); no self-service or self-hosting
- Statistical engine less documented/transparent than GrowthBook or Statsig

**Licence / IP notes**
- Proprietary SaaS
- No known patent encumbrances

---

### Kameleoon

**Core features**
- AI-powered experimentation with visitor conversion probability scoring using machine learning
- Full-stack A/B testing (client-side, server-side, mobile)
- Personalisation engine with ML-driven targeting
- Feature flagging and progressive delivery
- Strong GDPR/EU privacy compliance architecture
- Event streaming integration with Amplitude (announced 2025)

**Differentiating features**
- Conversion probability score: ML model predicts each visitor's likelihood to convert, enabling AI-driven traffic allocation to the winning variant
- Among the strongest EU privacy compliance postures of any commercial experimentation platform
- Native integration with Amplitude for event streaming and goal measurement (2025)

**UX patterns**
- Enterprise-facing UI with AI recommendations embedded in campaign management
- Segment builder combining behavioural, contextual, and ML-derived signals

**Integration points**
- Amplitude event streaming (2025)
- CMS, CDN, and e-commerce platform integrations
- REST API and server-side SDKs

**Known gaps**
- Enterprise-only pricing; no self-service or mid-market tiers
- Limited transparency around AI model methodology
- Smaller developer community than GrowthBook or PostHog

**Licence / IP notes**
- Proprietary SaaS
- No known patent encumbrances

---

### Convert

**Core features**
- A/B, multivariate, and split URL testing
- GDPR/CCPA-compliant by design: no free user profiling or third-party data sharing
- Visual editor for no-code variant creation
- Audience segmentation with custom goal tracking
- Agency-friendly features: multi-client project management
- Mid-market positioning with transparent traffic-based pricing ($199–$699/mo)

**Differentiating features**
- Explicit privacy stance: does not profile free users or share data with third parties — a differentiator over ad-funded competitors
- Mid-market price point with full feature set avoids the enterprise-only gap
- Well-regarded for agency use cases with multi-client project management

**UX patterns**
- Straightforward dashboard focused on experiment results without analytics overlay
- Transparent pricing calculator on the website

**Integration points**
- Google Analytics, Mixpanel, HubSpot, Segment integrations
- WordPress, Shopify, and CMS plugins
- REST API

**Known gaps**
- Smaller feature set than Optimizely or VWO
- Statistical engine less sophisticated: no published CUPED or sequential testing support
- Limited engineering/product team use cases

**Licence / IP notes**
- Proprietary SaaS
- No known patent encumbrances

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- A/B and multivariate test support with a visual editor for marketers and an SDK path for engineers
- Feature flags with percentage rollouts, targeting rules, and environment promotion
- Statistical significance reporting (frequentist p-values at minimum); confidence intervals
- Sample Ratio Mismatch (SRM) detection — its absence is a significant quality gap
- SDK availability for web JavaScript, at least one mobile platform (iOS/Android), and at least one server-side language
- Role-based access control and audit logging for experiment governance
- Integration with at least one major analytics tool or data warehouse

### Differentiating Features
- Warehouse-native architecture enabling experiments on existing data without a separate event pipeline
- CUPED variance reduction for improved experiment sensitivity (30–50% efficiency gain)
- Sequential testing (SPRT or always-valid inference) enabling valid early stopping
- Multi-armed bandit / adaptive allocation for automated traffic optimisation
- Global Holdouts for long-run cumulative lift measurement
- Knowledge Graph or cross-experiment causal model linking experiments to outcomes
- AI Configs for experimenting on LLM prompt/model parameters as feature flags

### Underserved Areas / Opportunities
- Cross-experiment interaction detection: no platform automatically detects or manages collisions between concurrent experiments affecting the same users
- AI-native hypothesis generation from product telemetry: all platforms require manual experiment ideation
- Natural language experiment authoring: no platform allows plain-language experiment configuration end-to-end
- Automated SRM root-cause analysis: SRM detection is available but root cause (bot traffic, SDK misconfiguration, etc.) is not automatically diagnosed
- Self-hostable platform with enterprise-grade stats engine: GrowthBook comes closest but lacks some enterprise UX polish
- Compounding organisational learning: no tool builds a persistent causal model across the experiment corpus to surface which product areas yield consistent lift

### AI-Augmentation Candidates
- Hypothesis generation from funnel and retention telemetry
- Automated pre-launch experiment configuration review (Optimizely's Opal is an early example)
- SRM anomaly root-cause analysis
- Natural language experiment authoring (metrics, targeting, guardrails from a plain-language description)
- Cross-experiment interaction effect detection
- Post-experiment narrative report generation for non-statistical stakeholders

---

## Legal & IP Summary

All ten tools in this survey are either MIT-licensed open-source (GrowthBook core, PostHog) or proprietary commercial SaaS products. The MIT licence is permissive and imposes no constraints on building derivative or competing products. The GrowthBook Enterprise Licence covers only specific premium directories and is not OSI-approved; the core product remains MIT. No patent-encumbered techniques were identified: CUPED, SPRT, Bayesian A/B testing, and Sample Ratio Mismatch checks are all published academic techniques in the public domain. LaunchDarkly and PostHog both support the OpenFeature CNCF standard, which is Apache 2.0 licensed. Optimizely has filed patents on specific DXP personalisation techniques unrelated to core A/B mechanics. Any new platform should conduct a freedom-to-operate review before implementing ML-based adaptive traffic allocation, as this area is more active in patent filings from large technology companies.

---

## Recommended Feature Scope

**Must-have (MVP)**:
- Feature flags with percentage rollouts, attribute-based targeting, and multi-environment promotion
- A/B and multivariate experiment engine with frequentist statistics (p-values, confidence intervals)
- Sample Ratio Mismatch (SRM) detection on every experiment
- Warehouse-native data source connectivity (BigQuery, Snowflake, at least two others)
- SDKs for JavaScript (web), at least one mobile platform, and one server-side language
- REST API for CI/CD pipeline integration

**Should-have (v1.1)**:
- CUPED variance reduction for improved experiment sensitivity
- Sequential testing (SPRT or always-valid inference) for valid early stopping
- Bayesian statistics engine as an alternative to frequentist
- Standardised metric registry shared across experiments
- Multi-armed bandit / adaptive allocation experiments
- Role-based access control with audit logging

**Nice-to-have (backlog)**:
- AI-assisted hypothesis generation from product telemetry
- Natural language experiment authoring (LLM-powered configuration from plain-language description)
- Cross-experiment interaction detection and collision management
- Global Holdouts for long-run cumulative lift measurement
- MCP server for AI client integration (Claude, Cursor, etc.)
