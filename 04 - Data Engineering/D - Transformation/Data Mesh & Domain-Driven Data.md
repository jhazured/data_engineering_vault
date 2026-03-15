tags: #data-mesh #domain-driven #decentralised #data-architecture #data-products #governance #organisational-design

# Data Mesh & Domain-Driven Data

Data Mesh is a sociotechnical approach to managing analytical data at scale, first articulated by Zhamak Dehghani in 2019. It shifts data ownership from centralised teams to the domains that generate and best understand the data, treating analytical data as a product and supporting it with a self-serve platform and federated governance.

This note covers the four principles, implementation patterns, technology mapping, organisational challenges, and when Data Mesh is — and is not — the right fit.

---

## The Four Principles

Dehghani's Data Mesh rests on four interlocking principles. Adopting only one or two typically fails; they reinforce each other.

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA MESH                                │
│                                                                 │
│   ┌──────────────────┐        ┌──────────────────┐             │
│   │  1. Domain        │        │  2. Data as a    │             │
│   │     Ownership     │◄──────►│     Product      │             │
│   └────────┬─────────┘        └────────┬─────────┘             │
│            │                           │                        │
│            ▼                           ▼                        │
│   ┌──────────────────┐        ┌──────────────────┐             │
│   │  3. Self-Serve    │◄──────►│  4. Federated    │             │
│   │     Platform      │        │     Governance   │             │
│   └──────────────────┘        └──────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

| Principle | One-Line Summary |
|-----------|-----------------|
| **Domain Ownership** | Domains own, produce, and serve their own analytical data |
| **Data as a Product** | Analytical data sets are treated with the same rigour as user-facing products |
| **Self-Serve Data Platform** | Infrastructure abstracts away complexity so domain teams can ship data products independently |
| **Federated Computational Governance** | Global interoperability rules are encoded in policy-as-code, enforced automatically, with local autonomy preserved |

---

## Principle 1 — Domain Ownership

### Domains as First-Class Data Producers

In a centralised model, a single data engineering team ingests, transforms, and serves data for the entire organisation. This creates a bottleneck: the team cannot hold deep context for every business domain.

Data Mesh inverts this. Each domain (e.g., Orders, Logistics, Customer, Finance) becomes responsible for:

- **Producing** clean, well-modelled analytical data
- **Publishing** it as a discoverable data product
- **Operating** the pipelines that generate it
- **Supporting** downstream consumers

This aligns with **Conway's Law** — the architecture mirrors the organisational communication structure, which reduces friction rather than fighting it.

### Team Topology

A domain data team typically includes:

| Role | Responsibility |
|------|---------------|
| **Domain Data Engineer** | Builds and operates the domain's data pipelines |
| **Data Product Owner** | Defines SLAs, schema contracts, and consumer needs |
| **Analytics Engineer** | Creates semantic models and metrics for the domain |
| **Domain Engineer** (existing) | Provides source system context and CDC access |

In practice, the data engineer and analytics engineer roles may be combined in smaller domains. The key shift is that these people report into the domain, not a centralised data team.

> **Tip:** Start with domains that already have strong engineering cultures and well-understood data. Expand outward once patterns are proven.

---

## Principle 2 — Data as a Product

A data product is a unit of analytical data that is deliberately designed, built, and maintained for consumption — not a by-product dumped into a lake.

### The Six Qualities of a Data Product

| Quality | What It Means | How to Measure |
|---------|--------------|----------------|
| **Discoverable** | Consumers can find the product without asking the producing team | Registered in a [[Data Cataloguing & Discovery]] tool with documentation |
| **Addressable** | A stable, unique identifier or URI points to the product | Namespace convention: `domain.product.version` |
| **Trustworthy** | Quality is measured, monitored, and guaranteed via SLAs | Freshness, completeness, and accuracy checks via [[Data Validation & Quality Frameworks]] |
| **Self-Describing** | Schema, semantics, lineage, and sample data are embedded in metadata | Machine-readable contracts (see [[Data Contracts & Schema Enforcement]]) |
| **Interoperable** | Products conform to shared standards so they can be joined across domains | Global naming conventions, shared identifier registries, standard date/time formats |
| **Secure** | Access is controlled by policy, not ad-hoc grants | Role-based access, column-level masking, audit logging |

### Data Product Anatomy

```
data-product/
├── interface/
│   ├── schema.yaml           # Contract: columns, types, SLAs
│   ├── sample_data.parquet   # Quick inspection for consumers
│   └── lineage.json          # Upstream dependencies
├── logic/
│   ├── models/               # dbt models or Spark jobs
│   └── tests/                # Quality assertions
├── infrastructure/
│   ├── pipeline.yaml         # Orchestration definition
│   └── access_policy.yaml    # Who can read, join, export
└── docs/
    └── README.md             # Business context, known caveats
```

---

## Principle 3 — Self-Serve Data Platform

### Infrastructure as a Platform

Domain teams should not need to become Kubernetes experts or Terraform specialists to ship a data product. The platform team's job is to provide opinionated, composable building blocks that abstract infrastructure concerns.

### Platform Capabilities

| Capability Layer | What It Provides | Example Tools |
|-----------------|------------------|---------------|
| **Data Product Lifecycle** | Scaffolding, CI/CD, versioning, deprecation | Internal CLI, GitLab CI, GitHub Actions |
| **Storage & Compute** | Provisioned warehouses, lake storage, query engines | Snowflake, [[Databricks & Delta Lake]], BigQuery |
| **Ingestion** | Connectors, CDC, streaming primitives | Fivetran, Debezium, [[Apache Kafka Fundamentals]] |
| **Transformation** | Declarative modelling, testing, documentation | [[Core dbt Fundamentals]], Spark |
| **Cataloguing & Discovery** | Registration, search, lineage visualisation | DataHub, Atlan, Unity Catalog, [[Data Cataloguing & Discovery]] |
| **Governance Automation** | Policy enforcement, access provisioning, tagging | OPA, Immuta, Privacera |
| **Observability** | Freshness checks, anomaly detection, alerting | Monte Carlo, Elementary, [[Pipeline Observability & Monitoring]] |

### Reducing Cognitive Load

The platform succeeds when a domain engineer can:

1. Run a single command to scaffold a new data product
2. Write transformation logic in SQL or Python without worrying about infrastructure
3. Push to a branch and have CI validate schema compatibility, run tests, and preview changes
4. Merge to main and have CD deploy the product with correct access policies

If domains still need to raise tickets to provision storage or configure networking, the platform is incomplete.

---

## Principle 4 — Federated Computational Governance

### Global Policies, Local Autonomy

Governance in a mesh is not a centralised review board. It is a set of machine-enforceable standards that every data product must satisfy, combined with local freedom for domains to model data as they see fit.

### The Governance Stack

```
┌─────────────────────────────────────────────┐
│         Federated Governance Council         │
│   (representatives from each domain + platform) │
├─────────────────────────────────────────────┤
│   Global Standards (encoded as policies)     │
│   • Naming conventions                       │
│   • PII classification rules                 │
│   • SLA tiers (Gold / Silver / Bronze)       │
│   • Interoperability formats (date, currency)│
│   • Mandatory metadata fields                │
├─────────────────────────────────────────────┤
│   Automated Compliance                       │
│   • CI checks: schema validation, naming     │
│   • Runtime checks: freshness, row counts    │
│   • Access audits: who read what, when       │
├─────────────────────────────────────────────┤
│   Local Autonomy                             │
│   • Domain chooses internal modelling style   │
│   • Domain picks transformation tooling       │
│   • Domain sets refresh cadence within SLA    │
└─────────────────────────────────────────────┘
```

### What Gets Centralised vs Decentralised

| Concern | Centralised (Global) | Decentralised (Domain) |
|---------|---------------------|----------------------|
| **Identifier standards** | Shared customer/product ID registries | Domain-specific surrogate keys |
| **PII handling** | Classification taxonomy, masking rules | Which columns to tag in their product |
| **SLA definitions** | Tier definitions (e.g., Gold = < 1 hr latency) | Which tier a product targets |
| **Schema standards** | Required metadata fields | Internal table structures |
| **Tooling** | Approved platform components | Choice of SQL vs Python, modelling approach |
| **Access model** | RBAC framework, audit requirements | Granting access to specific consumers |

---

## Data Mesh vs Centralised Data Teams vs Data Fabric

| Dimension | Centralised Data Team | Data Fabric | Data Mesh |
|-----------|----------------------|-------------|-----------|
| **Ownership** | Central team owns all pipelines | Central team with automation | Domain teams own their data |
| **Architecture** | Monolithic warehouse or lake | Virtualisation + metadata layer | Distributed, domain-aligned products |
| **Governance** | Central approval gates | Automated policy via metadata | Federated: global standards, local execution |
| **Technology Focus** | Single stack (e.g., Snowflake + dbt) | Integration middleware, knowledge graphs | Platform as a product, polyglot |
| **Scaling Model** | Hire more central engineers | Add more connectors/automation | Each domain scales independently |
| **Time to Value** | Fast for first use case, slows as complexity grows | Medium — requires significant metadata investment | Slow to start, scales well with maturity |
| **Best Fit** | Small-to-medium orgs, few domains | Large orgs needing integration across legacy systems | Large orgs with many autonomous domains |
| **Key Risk** | Bottleneck, context loss | Complexity of virtualisation layer | Duplication, inconsistency, skill gaps |

> **Note:** Data Fabric and Data Mesh are not mutually exclusive. A mesh domain might use fabric-style virtualisation internally. The distinction is primarily organisational, not technological.

---

## Implementation Patterns

### Domain-Aligned Data Products

Map data products to bounded contexts from Domain-Driven Design:

| Data Product Type | Description | Example |
|-------------------|------------|---------|
| **Source-Aligned** | Clean, conformed version of a domain's operational data | `orders.order_events.v2` |
| **Aggregate** | Pre-computed metrics or summaries | `finance.monthly_revenue.v1` |
| **Consumer-Aligned** | Purpose-built for a specific downstream use case | `marketing.attribution_model.v1` |

Source-aligned products are the most common starting point. They act as the "public API" of a domain's data.

### Data Product APIs

Every data product exposes a standard interface:

- **Read API** — SQL access (table/view in a shared warehouse), file access (Parquet in object storage), or streaming topic
- **Discovery API** — Metadata endpoint registered in the catalogue
- **Observability API** — Health checks, freshness timestamps, row counts
- **Schema Contract** — Versioned schema definition with compatibility guarantees (see [[Data Contracts & Schema Enforcement]])

### Mesh Topology

```
                    ┌──────────────┐
                    │   Platform   │
                    │   (shared    │
                    │   infra)     │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
  │  Orders    │     │  Customer │     │  Finance  │
  │  Domain    │────►│  Domain   │────►│  Domain   │
  │            │     │           │     │           │
  │ Products:  │     │ Products:  │     │ Products:  │
  │ • events   │     │ • profiles │     │ • revenue  │
  │ • items    │     │ • segments │     │ • invoices │
  └────────────┘     └────────────┘     └────────────┘
        │                                      ▲
        └──────────────────────────────────────┘
              (cross-domain consumption)
```

Arrows represent data consumption, not data flow. The Orders domain publishes `order_events`; Finance consumes it to compute revenue. Each domain owns its own transformation logic.

---

## Technology Mapping

How common data engineering tools map to Data Mesh roles:

| Mesh Concern | Tool | How It Fits |
|-------------|------|-------------|
| **Transformation** | [[Core dbt Fundamentals]] | One dbt project per domain, shared packages for global conventions. dbt Mesh (cross-project references) enables inter-domain dependencies. |
| **Warehouse** | Snowflake | One database per domain, shared via secure views or data shares. Role hierarchy enforces access. See [[Snowflake RBAC & Data Security]]. |
| **Lakehouse** | [[Databricks & Delta Lake]] | Unity Catalog provides three-level namespace (`catalogue.schema.table`) that maps to `domain.product.version`. |
| **Streaming** | [[Apache Kafka Fundamentals]] | One topic namespace per domain. Schema Registry enforces contract compatibility. |
| **Catalogue** | DataHub / Atlan / Unity Catalog | Central discovery layer that indexes all domain products. See [[Data Cataloguing & Discovery]]. |
| **Quality** | Great Expectations / Elementary / Soda | Embedded in each domain's CI pipeline. See [[Data Validation & Quality Frameworks]]. |
| **Orchestration** | [[Apache Airflow & Orchestration Patterns]] | Per-domain DAGs or a shared Airflow with namespace isolation. Cross-domain triggers via dataset-aware scheduling. |
| **Contracts** | Protobuf / JSON Schema / sdf | Schema definitions versioned alongside code. See [[Data Contracts & Schema Enforcement]]. |
| **Platform Provisioning** | Terraform / Pulumi | Platform team maintains modules; domains consume them declaratively. See [[Terraform for Data Infrastructure]]. |

### dbt Mesh — A Practical Starting Point

dbt introduced **cross-project references** (dbt Mesh) in 2023, making it one of the most accessible entry points:

1. Each domain has its own dbt project with its own repository
2. Domains publish **public models** — the data product interface
3. Other domains reference published models via `{{ ref('domain_project', 'model_name') }}`
4. Schema contracts (`contract: true` in YAML) enforce column types and prevent breaking changes
5. dbt Explorer provides cross-project lineage and discovery

This approach works well within a single warehouse (Snowflake, BigQuery, Databricks) and requires minimal platform investment.

---

## Organisational Challenges

### Conway's Law in Practice

> "Any organisation that designs a system will produce a design whose structure is a copy of the organisation's communication structure." — Melvin Conway

Data Mesh explicitly leverages Conway's Law. If your organisation has strong domain boundaries, a mesh architecture aligns naturally. If your organisation is functionally siloed (all data engineers in one team, all analysts in another), adopting Data Mesh requires organisational restructuring first.

### Team Boundaries

| Challenge | Symptom | Mitigation |
|-----------|---------|------------|
| **Unclear domain boundaries** | Nobody agrees which team owns "customer" data | Run an EventStorming or domain discovery workshop before starting |
| **Shared entities** | Customer, Product, Employee appear in many domains | Designate a "source of truth" domain; others consume, not duplicate |
| **Cross-domain joins** | Analysts need data from five domains in one query | Provide a consumption layer (shared schema or BI-specific data products) |
| **Operational overhead** | Each domain must now run pipelines, not just write SQL | Platform must reduce this to near-zero via automation |

### Skill Gaps

Moving to Data Mesh typically requires domain teams to acquire skills they did not previously need:

- **Data modelling** — [[Dimensional Modelling (Kimball)]], [[Data Vault 2.0 Modelling]]
- **Pipeline engineering** — orchestration, idempotency, [[Incremental Loading Strategies]]
- **Quality engineering** — testing, monitoring, [[dbt Testing & Data Quality]]
- **Product thinking** — SLAs, documentation, consumer empathy

The platform team and a central data enablement function (often called a Centre of Excellence) can bridge this gap through training, pair programming, and reusable templates.

### Migration Path from Monolith

A big-bang migration never works. Use a strangler-fig pattern:

1. **Identify one high-value domain** with a willing team and clear data boundaries
2. **Build the first data product** on the self-serve platform, following all governance standards
3. **Prove value** — faster time to insight, fewer tickets to central team, clearer ownership
4. **Document the playbook** — patterns, pitfalls, templates
5. **Expand domain by domain**, refining the platform with each iteration
6. **Sunset monolith pipelines** as domains take ownership

```
Phase 1 (3-6 months)     Phase 2 (6-12 months)     Phase 3 (12-24 months)
─────────────────────    ─────────────────────     ─────────────────────
1-2 pilot domains        3-5 domains migrated      Full mesh operating
Platform MVP             Platform hardened          model
Governance v1            Cross-domain patterns      Monolith retired
                         established
```

---

## When Data Mesh Is NOT the Right Fit

Data Mesh introduces significant organisational and technical complexity. It is overkill — or actively harmful — in several scenarios:

| Scenario | Why Mesh Fails | Better Approach |
|----------|---------------|-----------------|
| **Small organisation** (< 50 engineers) | Not enough people to staff domain data teams | Centralised data team with clear ownership labels |
| **Single domain** | No domain boundaries to distribute across | Standard [[Data Engineering Lifecycle]] with one team |
| **Early-stage company** | Requirements change too fast; premature abstraction | Monolithic warehouse, iterate on schema |
| **Low data maturity** | No existing data culture, no quality baselines | Invest in fundamentals first: ingestion, quality, cataloguing |
| **Regulatory monolith** | All data must be controlled by one team for compliance | Centralised governance with embedded analysts |
| **Homogeneous data** | One source system, one consumption pattern | Simple ELT pipeline; mesh adds no value |

> **Anti-pattern:** Adopting Data Mesh because it is trendy, without the organisational complexity that justifies it. A 20-person startup does not need federated governance.

---

## Real-World Adoption Patterns

### What Works

- **Start with the platform** — build self-serve capabilities before asking domains to take ownership. Without a platform, you are just distributing pain.
- **Domain discovery workshops** — use EventStorming or wardley mapping to identify natural domain boundaries before drawing architecture diagrams.
- **Data product thinking** — train domain teams to think about consumers, SLAs, and contracts. This cultural shift matters more than tooling.
- **Incremental migration** — strangler-fig, not big-bang. Each domain joins the mesh when ready.
- **Measure adoption** — track number of registered data products, consumer satisfaction, time-to-first-query for new consumers.
- **Central enablement team** — a small group that coaches domains, maintains the platform, and evolves governance. This is not a centralised data team; it is an enabling team in the Team Topologies sense.

### Common Anti-Patterns

| Anti-Pattern | Description | Consequence |
|-------------|-------------|-------------|
| **Mesh in name only** | Relabel the central team's outputs as "data products" without changing ownership | No actual decentralisation; same bottleneck with new jargon |
| **Platform neglect** | Push ownership to domains without investing in self-serve tooling | Domains drown in operational toil; quality drops |
| **No governance** | Decentralise everything, including standards | Data silos return; nothing is interoperable |
| **Over-governance** | Central committee must approve every schema change | Worse than the monolith; domains cannot move independently |
| **Copy-paste domains** | Every domain rebuilds the same boilerplate from scratch | Wasted effort; platform should provide golden paths |
| **Ignoring shared entities** | No strategy for cross-cutting data like Customer or Product | Conflicting definitions, reconciliation nightmares |
| **Big-bang migration** | Attempt to move all domains to mesh simultaneously | Overwhelms the platform team; no time to learn and iterate |

### Maturity Model

| Level | Description | Indicators |
|-------|------------|------------|
| **0 — Ad Hoc** | No formal data ownership; data lives in spreadsheets and personal databases | No catalogue, no SLAs, tribal knowledge |
| **1 — Centralised** | One team manages all data pipelines and warehouse | Central backlog, context switching, bottleneck at scale |
| **2 — Emerging Mesh** | 1-3 pilot domains produce data products on a platform MVP | First data products registered, governance v1 in place |
| **3 — Scaling Mesh** | Most domains produce data products; platform is self-serve | Cross-domain consumption is routine; automated compliance |
| **4 — Mature Mesh** | All analytical data is managed as products with federated governance | Domains operate independently; platform evolves via internal open source |

---

## Key Takeaways

1. Data Mesh is an **organisational** architecture first, a **technical** architecture second
2. All four principles must work together — adopting one in isolation fails
3. The **self-serve platform** is the enabler; without it, decentralisation just distributes suffering
4. Start small: one domain, one data product, one platform capability at a time
5. Data Mesh requires **organisational maturity** — clear domain boundaries, engineering culture in domains, and executive sponsorship
6. It is not a replacement for good data engineering fundamentals — you still need solid [[Data Engineering Lifecycle]] practices within each domain

---

## Related Notes

- [[Data Cataloguing & Discovery]] — central to the discoverability quality of data products
- [[Data Contracts & Schema Enforcement]] — the mechanism for interoperability across domains
- [[Core dbt Fundamentals]] — dbt Mesh as a practical implementation pattern
- [[Databricks & Delta Lake]] — Unity Catalog's namespace model maps well to mesh topology
- [[Apache Kafka Fundamentals]] — streaming data products and topic ownership
- [[Data Validation & Quality Frameworks]] — trustworthiness enforcement within domains
- [[Data Engineering Lifecycle]] — the foundational lifecycle that each domain still follows
- [[Dimensional Modelling (Kimball)]] — modelling patterns within domain data products
- [[Data Vault 2.0 Modelling]] — alternative modelling approach suited to mesh contexts
- [[Pipeline Observability & Monitoring]] — observability as a platform capability
