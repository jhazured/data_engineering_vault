# Data Contracts & Schema Enforcement

tags: #data-contracts #schema #data-quality #governance #data-engineering

---

## What Are Data Contracts?

A **data contract** is a formal agreement between a **data producer** and a **data consumer** that defines the structure, semantics, quality guarantees, and operational expectations of a dataset or data interface. It shifts data quality enforcement **left** — closer to the source — rather than relying on downstream consumers to detect and handle issues.

**Why they matter:**

- **Trust** — consumers can build reliable pipelines knowing the shape and quality of incoming data
- **Accountability** — producers take explicit ownership of the data they emit
- **Decoupling** — teams can evolve independently as long as the contract is honoured
- **Break detection** — schema and quality violations are caught before they propagate downstream
- **Data mesh enablement** — contracts are a cornerstone of treating data as a product (see [[Data Cataloguing & Discovery]])

Without contracts, organisations default to a "hope-based" integration model where consumers discover issues only after pipelines fail in production.

---

## Contract Components

A well-defined data contract typically includes five pillars:

### 1. Schema Definition

The structural specification of the data: field names, types, nullability, nesting, and ordering.

```yaml
fields:
  - name: customer_id
    type: string
    required: true
    description: "Unique business identifier for the customer"
  - name: order_total
    type: decimal(12,2)
    required: true
    constraints:
      - minimum: 0.00
  - name: created_at
    type: timestamp
    required: true
    format: "ISO 8601"
```

### 2. Service-Level Agreements (SLAs)

Operational guarantees the producer commits to:

- **Freshness** — data arrives within N minutes/hours of the source event
- **Availability** — uptime percentage of the data interface (e.g., 99.9%)
- **Latency** — maximum time from event occurrence to data availability
- **Volume** — expected row counts or byte ranges per delivery window

### 3. Ownership

- **Producer team** — who is responsible for maintaining the contract
- **Consumer registry** — known downstream dependants
- **Escalation path** — how breaking changes are communicated
- **Approval workflow** — who must sign off on contract changes

### 4. Quality Expectations

Declarative rules the data must satisfy (see [[Data Validation & Quality Frameworks]]):

- Uniqueness constraints (e.g., `customer_id` is unique per batch)
- Referential integrity (e.g., `product_id` exists in the products dataset)
- Freshness thresholds (e.g., no record older than 24 hours)
- Statistical bounds (e.g., `order_total` mean stays within 2 standard deviations of the trailing 30-day average)

### 5. Semantic Metadata

- Field-level descriptions and business glossary references
- Data classification (PII, confidential, public)
- Lineage pointers — where the data originates and where it flows

---

## Schema Enforcement Formats

Three dominant serialisation formats are used for schema enforcement in data pipelines. Each offers different trade-offs between strictness, ecosystem support, and human readability.

| Characteristic | **Protocol Buffers (Protobuf)** | **Apache Avro** | **JSON Schema** |
|---|---|---|---|
| **Serialisation** | Binary | Binary (+ JSON fallback) | Text (JSON) |
| **Schema location** | Compiled into code (`.proto` files) | Embedded in file header or registry | External document or inline |
| **Type system** | Strong, static | Rich (unions, logical types) | Flexible, JSON-native |
| **Schema evolution** | Field numbering, backward/forward safe | Union types, defaults | `additionalProperties`, versioning |
| **Code generation** | Yes (required) | Yes (optional) | Yes (optional) |
| **Human readability** | Low (binary wire format) | Low (binary) / Medium (JSON) | High |
| **Ecosystem** | gRPC, Google Cloud, microservices | [[Apache Kafka Fundamentals]], Hadoop, Spark | REST APIs, config files, OpenAPI |
| **Performance** | Excellent (smallest payload) | Very good | Moderate (text overhead) |
| **Best for** | Service-to-service RPC, high-throughput streams | Event streaming, data lake ingestion | API contracts, configuration validation |

### Protobuf Example

```protobuf
syntax = "proto3";

message CustomerEvent {
  string customer_id = 1;
  string name = 2;
  double order_total = 3;
  google.protobuf.Timestamp created_at = 4;
  // Field 5 reserved for future 'loyalty_tier'
  reserved 5;
}
```

### Avro Example

```json
{
  "type": "record",
  "name": "CustomerEvent",
  "namespace": "com.example.events",
  "fields": [
    {"name": "customer_id", "type": "string"},
    {"name": "name", "type": "string"},
    {"name": "order_total", "type": "double"},
    {"name": "created_at", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "loyalty_tier", "type": ["null", "string"], "default": null}
  ]
}
```

### JSON Schema Example

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["customer_id", "name", "order_total"],
  "properties": {
    "customer_id": {"type": "string", "minLength": 1},
    "name": {"type": "string"},
    "order_total": {"type": "number", "minimum": 0},
    "created_at": {"type": "string", "format": "date-time"}
  },
  "additionalProperties": false
}
```

---

## Schema Evolution Strategies

Schema evolution defines how a schema can change over time without breaking existing producers or consumers.

| Strategy | Definition | Allowed Changes | Forbidden Changes |
|---|---|---|---|
| **Backward compatible** | New schema can read data written by the old schema | Add optional fields, remove fields with defaults | Remove required fields, change field types |
| **Forward compatible** | Old schema can read data written by the new schema | Remove fields, add fields with defaults | Add required fields without defaults |
| **Full compatible** | Both backward and forward compatible simultaneously | Add/remove optional fields with defaults | Any required field change |
| **None** | No compatibility guarantee | Anything | (nothing forbidden — but risky) |

**Practical guidance:**

- **Default to backward compatibility** — this is the safest for most streaming and batch pipelines
- **Use full compatibility** when both producers and consumers may be at different schema versions simultaneously (common in [[Apache Kafka Fundamentals]] consumer groups)
- **Never use "none"** in production — it guarantees pipeline failures during rollouts

### Evolution Checklist

1. Add new fields as **optional with defaults** — safe under all compatibility modes
2. Never rename fields — add a new field and deprecate the old one
3. Never change a field's type — create a new field with the desired type
4. Use a **transition period** when removing fields: mark deprecated, wait for consumers to migrate, then remove

---

## Schema Registry Patterns

A **schema registry** is a centralised service that stores, versions, and validates schemas. It acts as the single source of truth for data contracts in streaming and batch architectures.

### Confluent Schema Registry

The most widely adopted registry, tightly integrated with [[Apache Kafka Fundamentals]]:

- Stores Avro, Protobuf, and JSON Schema
- Subjects follow `<topic>-value` / `<topic>-key` naming convention
- Compatibility checks enforced on schema registration
- REST API for schema CRUD operations

```bash
# Register a new schema version
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{...}"}' \
  http://schema-registry:8081/subjects/customer-events-value/versions

# Check compatibility before registering
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{...}"}' \
  http://schema-registry:8081/compatibility/subjects/customer-events-value/versions/latest
```

### AWS Glue Schema Registry

Native AWS integration for streaming and batch workloads:

- Supports Avro and JSON Schema
- Integrates with Kinesis Data Streams, MSK, and Glue ETL
- Schema versioning with compatibility enforcement
- IAM-based access control
- Auto-discovery of schemas from Glue Crawlers

### Databricks Unity Catalog

Schema governance embedded in the lakehouse platform:

- Enforces schemas on Delta Lake tables as the default behaviour
- Column-level lineage tracking
- Schema evolution controlled via `mergeSchema` and `overwriteSchema` options
- Integrates access controls, data classification, and audit logging in one layer
- Works alongside [[Data Cataloguing & Discovery]] for broader governance

---

## Contract Testing Patterns

### Contract-First Development

Define the contract **before** writing producer or consumer code:

1. **Design** — data producer and consumer teams agree on the contract specification
2. **Publish** — the contract is registered in the schema registry or contract repository
3. **Implement** — producers implement emission logic that conforms to the contract
4. **Validate** — automated tests verify both sides comply
5. **Monitor** — runtime checks ensure ongoing adherence

### Breaking Change Detection

Automated checks that run in CI/CD to prevent incompatible schema changes from reaching production:

```yaml
# Example CI pipeline step
- name: schema-compatibility-check
  run: |
    # Compare PR branch schema against production registry
    schema-tools check-compatibility \
      --schema ./schemas/customer_event.avsc \
      --registry https://schema-registry:8081 \
      --subject customer-events-value \
      --level BACKWARD
```

**What constitutes a breaking change:**

- Removing a required field
- Changing a field's data type
- Renaming a field without aliasing
- Tightening a constraint (e.g., reducing `maxLength`)
- Changing the serialisation format

**Detection strategies:**

- **Schema diff in CI** — compare the proposed schema against the latest registered version
- **Contract tests in the consumer repo** — consumers define expectations; producer CI runs them
- **Shadow validation** — run new schema against a sample of production data before deployment

---

## Implementation Patterns

### Producer-Side Validation

The producer validates data **before** emitting it. This is the strongest guarantee:

```python
from pydantic import BaseModel, Field
from datetime import datetime

class CustomerEvent(BaseModel):
    customer_id: str = Field(..., min_length=1)
    name: str
    order_total: float = Field(..., ge=0)
    created_at: datetime

# Validation happens at construction — invalid data raises immediately
event = CustomerEvent(
    customer_id="CUST_001",
    name="Acme Corp",
    order_total=150.00,
    created_at=datetime.utcnow()
)
```

**Advantages:** issues caught at the source, no bad data enters the pipeline.
**Disadvantages:** adds latency to the producer; requires producer team buy-in.

### Consumer-Side Validation

The consumer validates data **after** receiving it. A defensive pattern when producers cannot be trusted:

```python
import pandera as pa

customer_schema = pa.DataFrameSchema({
    "customer_id": pa.Column(str, nullable=False, unique=True),
    "order_total": pa.Column(float, pa.Check.ge(0)),
    "created_at": pa.Column("datetime64[ns]", nullable=False),
})

# Validate incoming DataFrame — raises SchemaError on violation
validated_df = customer_schema.validate(raw_df)
```

See [[Data Validation & Quality Frameworks]] for deeper coverage of pandera and Great Expectations.

### Contract-As-Code

Store contracts alongside application code in version control. The contract definition drives:

- Schema registration in CI/CD
- Test generation for producers and consumers
- Documentation generation for [[Data Cataloguing & Discovery]]
- Monitoring rule creation

```
repo-root/
├── contracts/
│   ├── customer_events.yaml      # Contract definition
│   ├── order_events.yaml
│   └── schemas/
│       ├── customer_event.avsc   # Avro schema
│       └── order_event.proto     # Protobuf schema
├── src/
│   └── producers/
│       └── customer_producer.py
└── tests/
    └── contract_tests/
        └── test_customer_contract.py
```

---

## Data Contract Specification Examples

### YAML Contract (Open Data Contract Standard Style)

```yaml
dataContract:
  version: "1.0.0"
  name: "customer_events"
  domain: "customer-platform"
  owner:
    team: "customer-engineering"
    contact: "customer-eng@example.com"

  schema:
    type: "avro"
    location: "./schemas/customer_event.avsc"
    compatibilityMode: "BACKWARD"

  quality:
    rules:
      - field: "customer_id"
        rule: "not_null"
      - field: "customer_id"
        rule: "unique"
      - field: "order_total"
        rule: "range"
        min: 0
        max: 1000000
      - field: "created_at"
        rule: "freshness"
        maxAgeMins: 1440

  sla:
    freshness:
      maxDelayMinutes: 60
    availability:
      uptimePercent: 99.5
    volume:
      expectedRowsPerHour:
        min: 1000
        max: 50000

  consumers:
    - team: "analytics"
      usage: "Daily aggregation for dashboards"
    - team: "ml-platform"
      usage: "Feature engineering for churn model"

  lifecycle:
    status: "active"              # draft | active | deprecated | retired
    deprecationDate: null
    migrationGuide: null
```

### JSON Contract (Lightweight API Style)

```json
{
  "contract": "customer_orders",
  "version": "2.1.0",
  "producer": "order-service",
  "schema": {
    "$ref": "./schemas/customer_order.json"
  },
  "quality": {
    "completeness": {"threshold": 0.99},
    "uniqueKey": ["order_id"],
    "freshness": {"maxAgeMinutes": 120}
  },
  "sla": {
    "deliverySchedule": "*/15 * * * *",
    "supportTier": "P2"
  }
}
```

---

## Tooling Landscape

### Soda

Declarative data quality checks defined in YAML, executed against warehouses and lakes:

```yaml
# soda checks YAML
checks for customers:
  - row_count > 0
  - missing_count(customer_id) = 0
  - duplicate_count(customer_id) = 0
  - avg(order_total) between 50 and 500
  - freshness(created_at) < 24h
```

- Supports Snowflake, BigQuery, Redshift, Spark, and many more
- **Soda Contracts** (dedicated feature) — define and enforce data contracts as YAML
- Integrates with orchestrators like Airflow for pipeline-embedded checks

### Great Expectations

Python-native expectation suites for DataFrame and SQL validation (see [[Data Validation & Quality Frameworks]]):

- Expectation suites map directly to contract quality rules
- Checkpoint-based execution with alerting
- Data Docs generate human-readable contract compliance reports

### dbt Contracts

dbt v1.5+ introduced **model contracts** — enforced schema definitions on dbt models (see [[Core dbt Fundamentals]] and [[dbt Testing & Data Quality]]):

```yaml
models:
  - name: dim_customers
    config:
      contract:
        enforced: true
    columns:
      - name: customer_id
        data_type: varchar(50)
        constraints:
          - type: not_null
          - type: primary_key
      - name: customer_name
        data_type: varchar(200)
        constraints:
          - type: not_null
      - name: lifetime_value
        data_type: number(12,2)
```

**Key behaviours:**

- When `contract.enforced: true`, dbt will fail the build if the model output does not match the declared column names, types, and constraints
- Prevents accidental schema drift from `SELECT *` or column reordering
- Constraints are pushed down to the warehouse (e.g., `NOT NULL` in Snowflake DDL)
- Pairs naturally with dbt tests for quality rules beyond structural validation

### Dataform (Google Cloud)

- SQL-based transformation tool with built-in assertions
- Assertions act as lightweight contract checks on table outputs
- Integrated with BigQuery for schema enforcement
- SQLX format supports inline documentation and dependency declarations

---

## Organisational Patterns

### Who Owns The Contract?

| Model | Description | Best For |
|---|---|---|
| **Producer-owned** | The team that produces data defines and maintains the contract | Data mesh architectures, microservices |
| **Consumer-driven** | Consumers specify what they need; producers must meet those specs | API-first organisations, strong consumer teams |
| **Platform-owned** | A central data platform team manages all contracts | Centralised data teams, early-stage contract adoption |
| **Jointly-owned** | Producer and consumer collaborate on the contract definition | Cross-functional data products |

**Recommendation:** Start with **producer-owned** contracts. The producer knows the data best and has the most control over its shape. Consumer-driven contracts work well as a complementary pattern where consumers register their expectations.

### Governance Workflows

**Contract Lifecycle:**

```
Draft → Review → Approved → Active → Deprecated → Retired
```

**Change Management Process:**

1. **Propose** — producer opens a pull request modifying the contract definition
2. **Impact analysis** — automated tooling identifies affected consumers
3. **Review** — consumer teams review and approve/reject the change
4. **Compatibility check** — CI validates schema compatibility mode
5. **Staged rollout** — deploy the new schema version; run shadow validation
6. **Migration window** — consumers have N days/sprints to adapt to non-breaking additions
7. **Enforcement** — after the migration window, the old schema version is deprecated

### Maturity Model

| Level | Characteristics |
|---|---|
| **Level 0 — Ad hoc** | No formal contracts; schema communicated via Slack or wiki pages |
| **Level 1 — Documented** | Schemas defined in files but not enforced at runtime |
| **Level 2 — Validated** | Schema validation in CI/CD; contract tests exist but coverage is partial |
| **Level 3 — Enforced** | Runtime schema enforcement; breaking changes blocked automatically |
| **Level 4 — Governed** | Full lifecycle management; consumer registry; SLA monitoring; automated compliance reporting |

---

## Key Takeaways

- Data contracts formalise the **producer-consumer agreement** and shift quality enforcement upstream
- Choose the serialisation format based on your ecosystem: **Avro** for Kafka-centric streaming, **Protobuf** for gRPC/microservices, **JSON Schema** for REST APIs and configuration
- **Backward compatibility** should be the default evolution strategy — it protects consumers from breaking changes
- Use a **schema registry** as the single source of truth; integrate compatibility checks into CI/CD
- dbt model contracts bring schema enforcement to the transformation layer — combine them with [[dbt Testing & Data Quality]] for full coverage
- Start with **producer-owned, contract-first development** and grow towards a governed maturity model
- Store contracts as code alongside application logic to enable version control, code review, and automated testing
