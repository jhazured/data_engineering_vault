---
tags: [cloud, comparison, aws, azure, gcp, multi-cloud, decision-framework]
---

# Cloud Platform Comparison

> A structured comparison of AWS, Azure, and GCP data services for data engineering. Use this as a decision framework when selecting a cloud provider or evaluating multi-cloud strategies.

See also: [[AWS Data Services for Data Engineering]] | [[Microsoft Fabric & Azure Data Services]] | [[GCP Data Services for Data Engineering]]

---

## 1. Service Mapping Table

### Storage

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Object Storage | S3 | ADLS Gen2 / Blob Storage | Cloud Storage (GCS) |
| Storage Tiers | Standard, Standard-IA, Glacier, Deep Archive | Hot, Cool, Cold, Archive | Standard, Nearline, Coldline, Archive |
| Lifecycle Policies | S3 Lifecycle Rules | Blob Lifecycle Management | Object Lifecycle Management |
| Versioning | S3 Versioning | Blob Versioning | Object Versioning |
| Encryption at Rest | SSE-S3, SSE-KMS, SSE-C | Azure Storage Encryption (Microsoft-managed or CMK) | Google-managed, CMEK, CSEK |
| Hadoop Connector | EMRFS / s3a:// | abfss:// | gcs-connector (gs://) |

### Compute and Processing

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Managed Spark | EMR, Glue | Synapse Spark, HDInsight | Dataproc |
| Serverless Spark | EMR Serverless, Glue | Synapse Spark Pools | Dataproc Serverless |
| Serverless Compute | Lambda | Azure Functions | Cloud Functions |
| Beam Runner | -- (use Flink on EMR/KDA) | -- (use Flink on HDInsight) | Dataflow |
| Workflow Orchestration | Step Functions | Logic Apps | Cloud Workflows |
| Container Orchestration | ECS, EKS | AKS | GKE |

### Data Warehouse

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Primary Warehouse | Redshift | Synapse Analytics / Fabric Warehouse | BigQuery |
| Serverless SQL | Athena | Synapse Serverless / Fabric SQL Endpoint | BigQuery (on-demand) |
| Storage-Compute Separation | Partial (Spectrum for external) | Full (Synapse Serverless, Fabric) | Full (native) |
| Pricing Model | Per-node-hour (provisioned) or per-query (Serverless) | CU-based (Fabric) or DWU-based (Synapse) | Per-TB-scanned (on-demand) or slot-hours (editions) |
| External Tables | Redshift Spectrum (via Glue Catalog) | Fabric Shortcuts / Synapse External Tables | BigQuery External Tables |

### Streaming

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Stream Ingestion | Kinesis Data Streams | Event Hubs | Pub/Sub |
| Managed Delivery | Kinesis Data Firehose | Stream Analytics | Dataflow (streaming) |
| Managed Kafka | MSK (Amazon Managed Streaming for Kafka) | Event Hubs (Kafka protocol) | Managed Service for Apache Kafka |
| Stream Processing | Kinesis Data Analytics (Flink), Lambda | Stream Analytics, Azure Functions | Dataflow (Apache Beam) |
| Real-time Analytics | -- (Kinesis + Lambda + DynamoDB) | Fabric Eventstream | BigQuery Streaming Inserts |

### Orchestration

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Managed Airflow | MWAA | -- (use AKS-hosted Airflow) | Cloud Composer |
| Native Orchestrator | Step Functions | Data Factory Pipelines | Cloud Workflows |
| Event-Driven Triggers | EventBridge | Event Grid | Eventarc |
| Scheduling | EventBridge Scheduler | Data Factory Triggers | Cloud Scheduler |

### ETL and Data Integration

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Managed ETL | Glue | Data Factory + Dataflows Gen2 | Dataflow |
| Data Catalogue | Glue Data Catalog | Purview / Fabric OneLake Catalog | Data Catalog |
| Schema Discovery | Glue Crawlers | Purview Scanning | Data Catalog Auto-Discovery |
| Data Governance | Lake Formation | Purview + Fabric Workspace Security | Dataplex |

### IAM and Security

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Identity Model | Users, Roles, Policies | Service Principals, Managed Identities, RBAC | Service Accounts, Roles |
| Policy Granularity | Fine-grained JSON policies on resource ARNs | Built-in RBAC roles + custom roles | Predefined roles + custom roles |
| Cross-Service Identity | IAM Roles (AssumeRole via STS) | Managed Identity | Workload Identity Federation |
| Secrets | Secrets Manager | Key Vault | Secret Manager |
| Network Isolation | VPC Endpoints, PrivateLink | Private Endpoints, VNet Integration | VPC Service Controls, Private Google Access |

### Monitoring and Observability

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Metrics and Dashboards | CloudWatch | Azure Monitor | Cloud Monitoring |
| Logging | CloudWatch Logs | Log Analytics | Cloud Logging |
| Alerting | CloudWatch Alarms, SNS | Azure Monitor Alerts | Cloud Monitoring Alerting Policies |
| Tracing | X-Ray | Application Insights | Cloud Trace |
| Cost Management | Cost Explorer, Budgets | Cost Management + Advisor | Billing Reports, Budgets |

---

## 2. Decision Framework

### When to Choose AWS

- **Mature ecosystem required.** AWS has the broadest service catalogue and the longest track record in production data platforms.
- **Existing AWS footprint.** If the organisation already runs workloads on EC2/ECS, keeping data pipelines in-region avoids egress costs and latency.
- **Flexible compute models needed.** EMR offers EC2, EKS, and Serverless variants; Glue provides fully managed PySpark; Lambda handles event-driven micro-ETL.
- **Fine-grained IAM.** AWS policy documents allow resource-level, action-level, and condition-level control that is more granular than GCP/Azure out of the box.
- **Kafka-centric streaming.** MSK provides a fully managed Kafka cluster without protocol translation.

### When to Choose Azure

- **Microsoft-centric organisation.** Deep integration with Active Directory, Office 365, Power BI, and Teams.
- **Unified analytics with Fabric.** The T0-T5 layered architecture on OneLake provides a single platform for ingestion, warehousing, transformation, and BI. See [[Azure Fabric Reference Architecture]].
- **Power BI is the primary BI tool.** Direct Lake mode on Fabric delivers near-real-time analytics without data imports.
- **Hybrid cloud requirements.** Azure Arc extends management to on-premises and edge environments.
- **Strong .NET / C# skills** in the engineering team.

### When to Choose GCP

- **BigQuery-centric analytics.** BigQuery's serverless, pay-per-scan model is best-in-class for variable and exploratory workloads with minimal operational overhead.
- **Apache Beam pipelines.** Dataflow is the only fully managed Beam runner, supporting both batch and streaming with the same code.
- **Managed Airflow needed.** Cloud Composer provides a turnkey Airflow environment with GCS-based DAG deployment. See [[GCP Data Services for Data Engineering#6. Cloud Composer]].
- **ML/AI integration.** Vertex AI integrates tightly with BigQuery ML, Dataflow, and GCS.
- **Cost-sensitive variable workloads.** BigQuery on-demand pricing and Dataproc autoscaling suit bursty workloads well.

### When to Consider Multi-Cloud

Legitimate multi-cloud scenarios:

- **Acquisition or merger** -- inherited workloads on a different cloud.
- **Best-of-breed services** -- e.g., BigQuery for analytics with AWS for streaming (Kinesis/MSK).
- **Regulatory requirements** -- data sovereignty mandates that certain data resides in a specific provider's region.
- **Vendor risk mitigation** -- contractual or board-level requirement to avoid single-vendor lock-in.

---

## 3. Pricing Comparison Cheat Sheet

### Object Storage (per GB/month, standard tier, approximate)

| Provider | Storage | PUT (per 10k) | GET (per 10k) | Egress (per GB) |
|----------|---------|--------------|--------------|-----------------|
| AWS S3 | $0.023 | $0.005 | $0.0004 | $0.09 |
| Azure Blob | $0.018 | $0.005 | $0.004 | $0.087 |
| GCS | $0.020 | $0.005 | $0.0004 | $0.12 |

### Data Warehouse (approximate, on-demand)

| Provider | Model | Approximate Cost |
|----------|-------|-----------------|
| AWS Redshift | Per-node-hour (dc2.large) | ~$0.25/hour |
| AWS Athena | Per-TB-scanned | $5.00/TB |
| Azure Fabric | CU-hour (F8) | ~$0.36/hour |
| Azure Synapse | DWU-hour (DW100c) | ~$1.20/hour |
| GCP BigQuery | Per-TB-scanned (on-demand) | $6.25/TB |
| GCP BigQuery | Slot-hour (editions) | ~$0.04/slot-hour |

**Note:** Prices vary by region and are subject to change. Always verify with the provider's pricing calculator.

---

## 4. Terraform Cross-Provider Patterns

Common infrastructure patterns and their Terraform equivalents across providers:

| Pattern | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Storage bucket | `aws_s3_bucket` | `azurerm_storage_account` + `azurerm_storage_container` | `google_storage_bucket` |
| Versioning | `aws_s3_bucket_versioning` | `azurerm_storage_account.blob_properties.versioning_enabled` | `google_storage_bucket.versioning` |
| Lifecycle rules | `aws_s3_bucket_lifecycle_configuration` | `azurerm_storage_management_policy` | `google_storage_bucket.lifecycle_rule` |
| Encryption | `aws_s3_bucket_server_side_encryption_configuration` | `azurerm_storage_account.identity` + CMK | `google_storage_bucket.encryption` |
| Service identity | `aws_iam_role` + `assume_role_policy` | `azurerm_user_assigned_identity` | `google_service_account` |
| IAM binding | `aws_iam_role_policy_attachment` | `azurerm_role_assignment` | `google_project_iam_member` |
| Secrets | `aws_secretsmanager_secret` | `azurerm_key_vault_secret` | `google_secret_manager_secret` |
| Monitoring alert | `aws_cloudwatch_metric_alarm` | `azurerm_monitor_metric_alert` | `google_monitoring_alert_policy` |

See also: [[Terraform for Data Infrastructure]] for detailed IaC patterns.

---

## 5. IAM Philosophy Comparison

### AWS -- Policy Documents

AWS uses fine-grained JSON policy documents attached to users, groups, or roles. Policies specify actions, resources (ARNs), and conditions. This gives maximum flexibility but requires careful management.

```
Principal (Role) --> Policy Document --> Actions + Resource ARNs + Conditions
```

### Azure -- Role-Based Access Control (RBAC)

Azure uses a hierarchical scope model (Management Group > Subscription > Resource Group > Resource) with built-in or custom role definitions. Managed Identities eliminate credential management.

```
Principal (Managed Identity) --> Role Assignment --> Scope (Resource Group or Resource)
```

### GCP -- Predefined Roles on Resources

GCP binds predefined or custom roles to service accounts at the resource level (project, bucket, dataset). Workload Identity Federation replaces key files for GKE and external workloads.

```
Principal (Service Account) --> IAM Binding --> Role + Resource
```

### Key Takeaway

All three providers support least-privilege access. The main difference is the granularity of the policy model:

- **AWS** -- most granular (action-level conditions, resource-level ARNs).
- **Azure** -- most hierarchical (scope inheritance simplifies management).
- **GCP** -- simplest model (predefined roles cover most use cases; fewer moving parts).

---

## 6. Multi-Cloud Anti-Patterns

### Anti-Pattern: Symmetric Deployment

**Symptom:** Running identical workloads on two clouds "for redundancy."

**Why it fails:** Double the operational cost, double the engineering effort, no actual resilience benefit (failover is never tested). Each cloud has different semantics for IAM, networking, and data services, so "identical" is an illusion.

**Better approach:** Choose a primary cloud per workload domain. Use the second cloud only for genuinely distinct workloads or regulatory requirements.

### Anti-Pattern: Lowest-Common-Denominator Architecture

**Symptom:** Avoiding cloud-native services to maintain portability. Using only VMs, PostgreSQL, and custom code.

**Why it fails:** Loses the cost, performance, and operational advantages of managed services (BigQuery, Redshift, Fabric). The "portable" architecture costs more and delivers less.

**Better approach:** Embrace cloud-native services for each platform. Abstract at the data contract level (file formats, schemas, APIs), not at the infrastructure level.

### Anti-Pattern: Cross-Cloud Data Pipelines

**Symptom:** Pipeline reads from S3, transforms on GCP Dataflow, writes to Azure Synapse.

**Why it fails:** Egress costs dominate the bill. Latency is unpredictable. Debugging spans three vendor consoles. IAM becomes a nightmare of cross-cloud credential exchange.

**Better approach:** Keep the full pipeline (ingest, transform, serve) within a single cloud. Exchange data between clouds at well-defined boundaries using object storage transfers or change data capture, not mid-pipeline.

### Anti-Pattern: Multi-Cloud Orchestration

**Symptom:** A single Airflow instance orchestrates jobs across AWS, Azure, and GCP.

**Why it fails:** Credential sprawl (the orchestrator needs privileged access to all three clouds). Network complexity (VPN or public internet between clouds). Blast radius of a compromised orchestrator is tripled.

**Better approach:** Each cloud runs its own orchestrator (MWAA, Data Factory, Cloud Composer). Cross-cloud triggers use event-driven messaging (e.g., write a marker file to a shared bucket, emit an event to a cross-cloud Pub/Sub or EventBridge).

### Anti-Pattern: Ignoring Egress Costs

**Symptom:** Architects design multi-cloud without modelling egress.

**Why it fails:** Cloud egress charges ($0.08-0.12/GB) can dwarf compute and storage costs when pipelines move terabytes between providers daily.

**Better approach:** Model egress costs explicitly during architecture review. Minimise cross-cloud data movement. Use cloud interconnects (AWS Direct Connect, Azure ExpressRoute, GCP Cloud Interconnect) for high-volume transfers.

---

## 7. Migration Considerations

### Data Migration Tooling

| Direction | Tool | Notes |
|-----------|------|-------|
| AWS to GCP | Storage Transfer Service | Scheduled, incremental, server-side |
| GCP to AWS | AWS DataSync | Agent-based or agentless |
| AWS to Azure | Azure Data Box / AzCopy | Data Box for large offline transfers |
| Any to Any | Rclone | Open-source, supports all three providers |

### Schema and Query Translation

| Source | Target | Key Differences |
|--------|--------|----------------|
| Redshift SQL | BigQuery SQL | `DISTKEY`/`SORTKEY` --> partitioning/clustering; `COPY` --> load jobs |
| BigQuery SQL | Synapse SQL | `STRUCT`/`ARRAY` --> JSON columns; `MERGE` syntax differences |
| T-SQL (Fabric) | BigQuery SQL | `HASHBYTES` --> `SHA256()`; `GETDATE()` --> `CURRENT_TIMESTAMP()` |

### What Transfers Cleanly

- **Parquet/Delta files** -- universally readable across all three platforms.
- **SQL logic** -- requires dialect translation but semantics are equivalent.
- **Airflow DAGs** -- portable between MWAA, Cloud Composer, and self-hosted.
- **Terraform modules** -- provider-specific but structurally similar.

### What Does Not Transfer

- **IAM policies** -- fundamentally different models; must be rebuilt.
- **Managed service configurations** -- Redshift WLM, BigQuery reservations, Fabric capacities are not equivalent.
- **Proprietary formats** -- Fabric Dataflows (Power Query M), Glue DynamicFrames, BigQuery ML models.

---

## Cross-References

- [[AWS Data Services for Data Engineering]] -- detailed AWS service patterns
- [[Microsoft Fabric & Azure Data Services]] -- Fabric T0-T5 architecture, Data Factory, Dataflows Gen2
- [[GCP Data Services for Data Engineering]] -- BigQuery, GCS, Dataflow, Cloud Composer
- [[Azure Fabric Reference Architecture]] -- reusable Fabric template
- [[Terraform for Data Infrastructure]] -- IaC patterns across clouds
- [[Data Lake Architecture Patterns]] -- medallion architecture, file formats
