# A/B Testing Platform

**Candidate #31** — A modern, open-source A/B testing and feature experimentation platform with warehouse-native architecture, advanced statistical methods, and AI-powered capabilities.

## Market Opportunity

The A/B testing software market is valued at **~$1.5–2.5 billion in 2025** and is projected to reach $4–5 billion by 2035 at 11–13% CAGR. The market is dominated by expensive enterprise platforms (Optimizely $63K–$200K+/year) and mid-market commercial tools (Statsig, LaunchDarkly, Clari).

**Key market gaps:**
- No open-source tool offers enterprise-grade experimentation with warehouse-native architecture
- Statsig and LaunchDarkly are cloud-only; enterprises require data sovereignty options
- Current platforms require manual experiment ideation; no tool auto-generates hypotheses from telemetry
- Most tools bundle feature flags with experimentation, but neither depth is market-leading

## What This Platform Solves

This is a **warehouse-native A/B testing engine** that connects directly to your data warehouse (BigQuery, Snowflake, Databricks) to run experiments without a separate event pipeline. It combines:

- **Statistical rigor**: CUPED variance reduction, Sequential testing (SPRT), Bayesian inference, and Sample Ratio Mismatch detection
- **Developer experience**: 24 SDKs covering web, mobile, and server-side languages; REST API for CI/CD integration
- **Self-hostable**: Open-source MIT core allows full data sovereignty and custom deployment
- **Feature flags**: Percentage rollouts, attribute-based targeting, and environment promotion built-in
- **Metrics governance**: Standardised metric registry shared across experiments

## Competitive Differentiation

| Aspect | This Platform | GrowthBook | PostHog | Statsig | Optimizely |
|--------|---------------|-----------|---------|---------|-----------|
| **Open Source** | Yes (MIT) | Yes (MIT core) | Yes (MIT core) | No | No |
| **Warehouse Native** | Yes | Yes | No | No | Yes (new) |
| **CUPED** | Yes | Yes | No | Yes | Yes |
| **Sequential Testing** | Yes | Yes | No | Yes | Yes |
| **Self-hostable** | Yes | Yes | Yes | No | No |
| **Price** | Free | Free / $2K+/mo | Free / $500+/mo | Free tier → $22K+/yr | $63K–$200K+/yr |

## Key Features

### Must-Have (MVP)
- Feature flags with percentage rollouts and targeting rules
- A/B and multivariate experiment engine with frequentist statistics
- Warehouse connectivity (BigQuery, Snowflake, Databricks, Redshift)
- Sample Ratio Mismatch (SRM) detection on every experiment
- SDKs for JavaScript, iOS/Android, and Python
- REST API for programmatic experiment management

### Should-Have (v1.1)
- CUPED variance reduction for improved sensitivity
- Sequential testing (SPRT) for valid early stopping
- Bayesian statistics engine as an alternative
- Standardised metric registry
- Multi-armed bandit / adaptive allocation experiments
- Role-based access control with audit logging

### Nice-to-Have (Backlog)
- AI-assisted hypothesis generation from product telemetry
- Natural language experiment authoring
- Cross-experiment interaction detection
- Global Holdouts for long-run cumulative lift measurement
- MCP server for AI client integration

## Technology Stack

**Backend**: Python/Go, warehouse SDK support  
**Frontend**: React, TypeScript  
**Data**: BigQuery, Snowflake, Databricks native querying  
**Statistics**: SciPy, Bayesian inference library  
**Licensing**: MIT (fully permissive)

## Market Entry Strategy

1. **MVP Launch** (months 1–6): Warehouse-native MVP with CUPED and SRM detection, targeting data-forward companies
2. **Feature Parity** (months 7–12): Sequential testing, Bayesian engine, multi-armed bandits
3. **Enterprise Features** (2027): AI hypothesis generation, cross-experiment analytics, governance layers
4. **Monetization**: Open-source core + managed cloud tier ($500–$2K/mo), enterprise features ($10K+/mo)

## Why This Matters

- **Enterprise adoption bottleneck**: Most enterprises can't afford Optimizely's $80K–$200K/year pricing
- **Data sovereignty requirement**: Statsig and LaunchDarkly are cloud-only; regulated industries need self-hosting
- **AI opportunity**: Hypothesis generation from telemetry and automated SRM root-cause analysis are unserved market gaps
- **Developer experience**: No tool combines warehouse-native querying with comprehensive SDK coverage

## Success Metrics

- **Year 1**: 500+ self-hosted deployments, $100K ARR from managed cloud
- **Year 2**: 5,000+ active experiments across all users, $1M ARR
- **Year 3**: 20% market share among SMB/mid-market (vs. 2% for GrowthBook)
