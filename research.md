# A/B Testing Platform

> Candidate #31 · Researched: 2026-05-01

## Existing Products and Software Packages

| Tool | Type | Description | Pricing | Strengths / Weaknesses |
|------|------|-------------|---------|------------------------|
| **Optimizely** | Commercial | Full-stack experimentation and digital experience platform (acquired by Episerver/private equity in 2020); includes web, feature, and full-stack experimentation | ~$63K–$200K+/year (custom); no free tier; annual contract required | Strengths: enterprise feature depth, sequential inference, multi-metric governance, CMS integration. Weaknesses: extremely high cost, complex onboarding, fragmented UX across acquired products |
| **LaunchDarkly** | Commercial | Feature flag and experimentation platform; cloud-only; widely used for progressive delivery | Starter ~$8.33/seat/mo; Pro ~$16.67/seat/mo; Enterprise custom | Strengths: mature feature flagging, large ecosystem, reliable SDKs. Weaknesses: no self-hosting, experimentation is secondary to flags, cost scales steeply with teams |
| **VWO (Visual Website Optimizer)** | Commercial | Conversion optimisation platform with heatmaps, session replay, and A/B testing | Starts ~$267/month for 10K tracked users; custom enterprise | Strengths: non-developer friendly, all-in-one CRO suite. Weaknesses: pricing opaque, statistical methodology less rigorous than warehouse-native tools |
| **GrowthBook** | Open Source / Commercial | Warehouse-native A/B testing and feature flags; Bayesian + frequentist stats engine; self-hostable MIT-licensed core | Free self-hosted; Cloud plans start ~$0 free tier → paid seats | Strengths: true open source, CUPED, Sequential, Bayesian, Bandits, SRM checks, warehouse-native. Weaknesses: smaller managed cloud ecosystem, less polished UI than enterprise competitors |
| **Statsig** | Commercial (now owned by OpenAI) | Unified experimentation, feature flags, and product analytics platform; cloud-only | Usage-based on metered events; free tier available; enterprise custom | Strengths: CUPED variance reduction (30–50% sensitivity gain), automated heterogeneous effect detection, fast SDK. Weaknesses: cloud-only, cost escalates at scale, now under OpenAI ownership raising strategic concerns |
| **PostHog** | Open Source / Commercial | All-in-one product analytics + A/B testing + feature flags + session replay; self-hostable | Free up to 1M events/month; usage-based beyond that | Strengths: breadth of features, open source, active community. Weaknesses: complex self-hosting stack (ClickHouse, Kafka, Redis), 52KB+ SDK, A/B testing is not the primary focus |
| **Amplitude Experiment** | Commercial | Experimentation layer built on Amplitude's analytics; sequential testing, CUPED | Custom pricing bundled with Amplitude analytics | Strengths: tight integration with user analytics. Weaknesses: requires Amplitude ecosystem buy-in, expensive for dedicated experimentation |
| **AB Tasty** | Commercial | Marketing-focused A/B testing and personalisation; AI-assisted recommendations | Custom pricing; typically $40K+/year for enterprise | Strengths: strong personalisation, no-code visual editor. Weaknesses: limited for engineering/product use cases, opaque pricing |
| **Kameleoon** | Commercial | AI-powered experimentation with visitor conversion probability scoring | Custom enterprise pricing | Strengths: AI-driven targeting, strong EU privacy compliance. Weaknesses: enterprise only, limited open data access |
| **Convert** | Commercial | Privacy-focused A/B testing with GDPR compliance; mid-market positioning | ~$199–$699/month depending on traffic | Strengths: solid GDPR stance, no free users profiling, good for agencies. Weaknesses: smaller feature set vs Optimizely, less sophisticated stats engine |

## Relevant Industry Standards or Protocols

- **ISO/IEC 27001** — Information security management; relevant for storing experiment assignment data and user segmentation safely.
- **GDPR / CCPA** — Experimentation platforms must handle user consent for variant assignment and tracking; significant compliance surface, especially for web-layer A/B testing tools.
- **CUPED (Controlled-experiment Using Pre-Experiment Data)** — Not a formal standard but a widely adopted variance reduction technique originating from Microsoft Research (2013); now the de facto requirement for enterprise-grade platforms.
- **Sequential Probability Ratio Test (SPRT)** — Statistical method enabling early stopping without inflating false positive rates; implemented by Optimizely, Statsig, and GrowthBook.
- **Sample Ratio Mismatch (SRM) checks** — Industry-standard diagnostic check for experiment integrity; absence from a platform is a significant quality gap.
- **OpenFeature (CNCF)** — Emerging open standard for feature flag evaluation APIs, supported by LaunchDarkly and others; relevant for portability between experimentation backends.
- **W3C Privacy Principles** — W3C working group on privacy-preserving measurement affects how client-side experiment assignment data may be collected without third-party cookies.

## Available Research Materials

1. Kohavi, R., Tang, D., & Xu, Y. (2020). *Trustworthy Online Controlled Experiments: A Practical Guide to A/B Testing.* Cambridge University Press. https://www.cambridge.org/core/books/trustworthy-online-controlled-experiments/D97B26382EB0EB2DC2019A7A7B518F59 — Peer-reviewed book; authoritative industry reference.

2. Deng, A., Xu, Y., Kohavi, R., & Walker, T. (2013). *Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data.* WSDM 2013. https://www.exp-platform.com/Documents/2013-02-CUPED-ImprovingSensitivityOfControlledExperiments.pdf — Original CUPED paper (peer-reviewed conference).

3. Johari, R., Koomen, P., Pekelis, L., & Walsh, D. (2017). *Peeking at A/B Tests: Why It Matters, and What to Do About It.* KDD 2017. https://dl.acm.org/doi/10.1145/3097983.3097992 — Sequential testing foundations; peer-reviewed.

4. Howard, S. R., et al. (2021). *Time-uniform, nonparametric, nonasymptotic confidence sequences.* Annals of Statistics. https://projecteuclid.org/journals/annals-of-statistics/volume-49/issue-2/Time-uniform-nonparametric-nonasymptotic-confidence-sequences/10.1214/20-AOS1991.full — Theoretical foundations for always-valid inference; peer-reviewed.

5. GrowthBook Engineering Blog (2023). *Bayesian vs Frequentist A/B Testing: A Practical Guide.* https://blog.growthbook.io — Practitioner-focused; preprint-equivalent industry writing.

6. Journal of Electronic Business & Digital Economics (2022). *A comparative study of frequentist vs Bayesian A/B testing in the detection of E-commerce fraud.* Emerald Publishing. https://www.emerald.com/jebde/article/1/1-2/3/225916/ — Peer-reviewed.

7. Lukas, M. et al. (2024). *A/B and Multivariant Testing Using Bayesian Algorithm.* IJRESM. https://journal.ijresm.com/index.php/ijresm/article/download/2755/2747/3592 — Peer-reviewed journal article.

## Market Research

**Market Size:** The A/B testing software market was valued at ~USD 1.50 billion in 2025, projected to reach USD 4.82 billion by 2036 at an 11.2% CAGR (Future Market Insights, 2025). A parallel estimate from Business Research Insights places the 2025 figure at USD 840 million growing to USD 2.49 billion by 2035 at 11.5% CAGR. The variance reflects differing scope (pure A/B tools vs. broader experimentation suites).

**Pricing Landscape:**

| Tier | Representative Tools | Approx. Cost |
|------|---------------------|--------------|
| Free / Open Source | GrowthBook (self-hosted), PostHog (free tier) | $0 |
| Startup / SMB | Statsig (free tier), Convert ($199+/mo), VWO ($267+/mo) | $0–$500/mo |
| Mid-market | LaunchDarkly Pro, AB Tasty | $500–$5K/mo |
| Enterprise | Optimizely, Kameleoon, Amplitude Experiment | $40K–$200K+/year |

**Key Buyer Personas:**
- Product managers and growth teams at B2C/B2B SaaS companies running feature rollouts
- Engineering teams needing feature flags tightly coupled to experiment data
- CRO/marketing teams at e-commerce companies running web-layer tests
- Data science teams at large enterprises managing complex multi-metric experiments

**Notable Acquisitions / Funding:**
- Optimizely acquired by Episerver (2020) for ~$600M; subsequently rebranded and expanded via further acquisitions into a multi-product DXP suite exceeding $400M ARR.
- Statsig acquired by OpenAI in 2024/2025, raising concerns among some users about independence.
- Fivetran + dbt Labs merger (October 2025, all-stock, ~$600M combined ARR) is indirectly relevant as data warehouse-native experimentation depends on the ingestion/transformation stack.
- 78% of enterprises use at least one experimentation tool as of 2024; 52% run more than 10 experiments/month (up from 29% five years prior).

## AI-Native Opportunity

- **Automated hypothesis generation:** Existing tools require humans to manually propose experiment ideas. An AI-native platform could analyze product telemetry, user session data, and funnel metrics to proactively generate ranked hypotheses — something no current tool offers end-to-end.
- **Adaptive experiment design:** Current platforms require pre-specified sample sizes, metrics, and stopping rules. AI could dynamically adjust traffic allocation using multi-armed bandit or Bayesian optimization strategies, reducing time-to-decision by 30–50% beyond what CUPED alone achieves.
- **Natural language experiment authoring:** Engineers and PMs still write targeting rules, metric definitions, and segmentation queries manually. LLMs could allow teams to describe an experiment in plain language and have the platform configure feature flags, metrics, guardrails, and rollout schedules automatically.
- **Automated diagnostic surfacing:** SRM checks, novelty effects, and interaction effects between concurrent experiments are poorly surfaced in most tools. An AI layer could continuously monitor running experiments and proactively alert teams to validity threats — a significant gap even in enterprise platforms.
- **Cross-experiment learning:** All current platforms treat experiments as isolated events. An AI-native tool could build a causal model across the experiment corpus, identifying which product areas yield consistent lifts and which are saturated — creating compounding organizational learning that scales beyond individual test results.
