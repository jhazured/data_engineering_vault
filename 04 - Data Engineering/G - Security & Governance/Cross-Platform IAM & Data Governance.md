---
tags:
  - security
  - iam
  - governance
  - gcp
  - azure
  - aws
  - data-lineage
  - rls
---

# Cross-Platform IAM & Data Governance

This note consolidates identity and access management patterns, row-level security implementations, data lineage with business impact scoring, and data retention policies across GCP, Azure/Fabric, and AWS. It serves as a cross-platform reference for designing secure, governed data platforms.

---

## GCP IAM Patterns

Source: [[gcp_infra_terraform]] Terraform IAM module.

### Service Account Design

GCP uses service accounts as the primary identity for non-human workloads. The Terraform module pattern creates a single service account per logical service and conditionally attaches IAM roles based on what the service needs:

```hcl
resource "google_service_account" "service_account" {
  account_id   = var.service_account_name
  display_name = var.display_name
  description  = var.description
  project      = var.project_id
  labels       = var.labels
}
```

Key design principles:
- **One service account per service** -- avoids shared credentials and simplifies audit trails
- **Conditional role attachment** -- each role binding uses a `count` parameter tied to a feature flag (e.g., `var.enable_bigquery`), so roles are only granted when the service actually needs them
- **Labels for governance** -- service accounts carry labels for cost allocation and ownership tracking

### Role Binding Patterns

The module uses `google_project_iam_member` (additive) rather than `google_project_iam_binding` (authoritative). This is the safer pattern for shared projects because it does not remove other members from the same role.

| Service | Reader Role | Writer/Admin Role | Feature Flag |
|---------|-------------|-------------------|--------------|
| Artifact Registry | `artifactregistry.reader` | `artifactregistry.writer` | `artifact_registry_repo != null` |
| Cloud Storage | `storage.objectViewer` | `storage.objectAdmin` | `storage_bucket != null` |
| Secret Manager | `secretmanager.secretAccessor` | `secretmanager.admin` | `enable_secret_manager` |
| Compute Engine | -- | `compute.instanceAdmin`, `compute.networkUser` | `enable_compute_admin` |
| BigQuery | -- | `bigquery.dataEditor`, `bigquery.jobUser` | `enable_bigquery` |
| Logging | -- | `logging.logWriter` | `enable_logging` |
| Monitoring | -- | `monitoring.metricWriter` | `enable_monitoring` |

Custom roles are supported via a `for_each` over `var.custom_roles`, allowing project-specific permissions without modifying the module.

### Workload Identity Federation

For GKE workloads, the module supports [[Workload Identity]], which eliminates service account keys entirely:

```hcl
resource "google_service_account_iam_binding" "workload_identity" {
  count              = var.enable_workload_identity ? 1 : 0
  service_account_id = google_service_account.service_account.name
  role               = "roles/iam.workloadIdentityUser"
  members            = var.workload_identity_members
}
```

Workload Identity maps Kubernetes service accounts to GCP service accounts, meaning pods authenticate as the GCP service account without needing a JSON key file. This is the recommended approach for any GKE-based data pipeline.

### Service Account Keys -- When and When Not To

The module supports key creation via `google_service_account_key`, but this should be a last resort:

| Approach | Security | Use Case |
|----------|----------|----------|
| Workload Identity | Best -- no keys | GKE pods |
| Attached service account | Good -- no keys | Compute Engine, Cloud Functions |
| Impersonation | Good -- short-lived tokens | Cross-project access, CI/CD |
| Service account key | Worst -- static credential | External systems with no GCP SDK |

### GCP IAM Security Gaps Observed

1. **No IAM Conditions** -- the Terraform module does not use `google_project_iam_member` conditions (e.g., restricting access by time, IP, or resource attribute). Adding `condition` blocks would enable zero-trust patterns.
2. **Project-level bindings only** -- all roles are bound at the project level. For multi-project estates, consider organisation-level policies or folder-level bindings.
3. **No custom role definitions** -- the module accepts custom role names but does not define them. Custom roles with minimal permissions are preferable to predefined roles that grant more than needed.
4. **Secret Manager admin is broad** -- granting `secretmanager.admin` allows creating and deleting secrets, not just reading them. Consider splitting into accessor (read) and versioner (write new versions) roles.

---

## Azure / Microsoft Fabric RLS Patterns

Source: [[fabric-aged-care-lakehouse]] RLS rules and [[data-engineering-wiki]] Fabric security patterns.

### Row-Level Security in Fabric Warehouse

Fabric implements RLS using security functions and security policies, following the T-SQL pattern:

```sql
-- Step 1: Create a security function
CREATE FUNCTION t5.fn_security_filter_department()
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT dept_id
    FROM t0.user_department_access
    WHERE user_name = USER_NAME();

-- Step 2: Create a security policy
CREATE SECURITY POLICY t5.policy_department_access
ADD FILTER PREDICATE t5.fn_security_filter_department()
ON t5.vw_payroll_detail
WITH (STATE = ON);
```

### RLS Architecture Placement

In the T0-T5 architecture, RLS should be applied at specific layers:

| Layer | Security Role | RLS Applied? |
|-------|--------------|:---:|
| T0 -- Control | Stores user-to-department mappings | No |
| T2 -- Historical | Raw historical data | No |
| T3 -- Transformation | Processing layer | No |
| T5 -- Presentation | Views consumed by users | Yes |
| Semantic Model | Power BI / DAX layer | Yes |

The principle: **apply RLS at the consumption layer, not the storage layer**. This keeps the underlying data accessible for engineering and transformation work whilst restricting what end users see.

### Dynamic RLS Patterns

**Manager hierarchy** -- a user sees their own department plus any departments they manage:

```sql
CREATE FUNCTION t0.fn_user_department_hierarchy()
RETURNS TABLE
AS
RETURN
    SELECT dept_id FROM t0.user_department_access
    WHERE user_name = USER_NAME()
    UNION
    SELECT dept_id FROM t2.dim_department d
    INNER JOIN t2.dim_employee e ON d.dept_id = e.department_id
    WHERE e.employee_id IN (
        SELECT manager_id FROM t2.dim_employee
        WHERE employee_id = (
            SELECT employee_id FROM t0.user_employee_mapping
            WHERE user_name = USER_NAME()
        )
    );
```

**Time-based access** -- restrict access to business hours for certain access levels:

```sql
CREATE FUNCTION t5.fn_time_based_access()
RETURNS TABLE
AS
RETURN
    SELECT dept_id
    FROM t0.user_department_access
    WHERE user_name = USER_NAME()
    AND (access_level = 'Full'
         OR (access_level = 'BusinessHours'
             AND DATEPART(HOUR, GETDATE()) BETWEEN 8 AND 18));
```

### RLS in Fabric Lakehouse (Delta/Spark)

For lakehouse tables (as opposed to warehouse tables), RLS uses Delta row filters:

```sql
-- Create or replace a row filter on a Delta table
CREATE OR REPLACE ROW FILTER region_filter ON silver_clients
  USING (region = current_setting('app.region'));
```

The session parameter `app.region` is set via the connection string or Fabric workspace parameter. This pattern is used in the aged-care lakehouse for regional manager access to client data.

### Semantic Model RLS (Power BI)

At the Power BI semantic model layer, RLS uses DAX expressions:

```dax
FILTER(
    fact_payroll_FINAL,
    fact_payroll_FINAL[dept_key] IN
        CALCULATETABLE(
            VALUES(t0_security[department_id]),
            t0_security[user_name] = USERNAME()
        )
)
```

### Column-Level Security

For sensitive columns (salary, date of birth, national identifiers), use view-based column filtering:

```sql
CREATE VIEW t5.vw_employee_restricted AS
SELECT
    employee_key, employee_id, first_name, last_name, email,
    CASE
        WHEN IS_MEMBER('hr_role') THEN annual_salary
        ELSE NULL
    END AS annual_salary
FROM t3.dim_employee_FINAL;
```

### Fabric Authentication Patterns

| Method | Use Case | Security Level |
|--------|----------|---------------|
| Managed Identity | Fabric service-to-service | Best -- no credentials |
| Service Principal | Automation, Data Factory | Good -- rotate secrets |
| User credentials | Interactive analysis | Acceptable -- MFA required |
| Connection string secrets | Legacy integrations | Poor -- avoid if possible |

---

## Data Lineage with Business Impact Scoring

Source: [[logistics-analytics-platform]] advanced data lineage module.

### Lineage Table Design

The governance schema tracks lineage as source-target pairs with four quantitative impact dimensions:

```sql
CREATE TABLE GOVERNANCE.ADVANCED_DATA_LINEAGE (
    LINEAGE_ID              VARCHAR(50) DEFAULT UUID_STRING(),
    SOURCE_TABLE            VARCHAR(255) NOT NULL,
    TARGET_TABLE            VARCHAR(255) NOT NULL,
    TRANSFORMATION_TYPE     VARCHAR(100) NOT NULL,
    BUSINESS_IMPACT_SCORE   FLOAT NOT NULL,
    DATA_QUALITY_SCORE      FLOAT NOT NULL,
    PERFORMANCE_IMPACT      FLOAT NOT NULL,
    COST_IMPACT             FLOAT NOT NULL,
    CRITICALITY_LEVEL       VARCHAR(20) NOT NULL,
    BUSINESS_OWNER          VARCHAR(255),
    TECHNICAL_OWNER         VARCHAR(255),
    LAST_UPDATED            TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    LINEAGE_METADATA        VARIANT
);
```

### Impact Scoring Functions

**Business impact** -- weighted combination of usage frequency, downstream dependency count, and criticality classification:

```sql
CASE
    WHEN business_criticality = 'CRITICAL' THEN 1.0
    WHEN business_criticality = 'HIGH'     THEN 0.8
    WHEN business_criticality = 'MEDIUM'   THEN 0.6
    WHEN business_criticality = 'LOW'      THEN 0.4
    ELSE 0.2
END *
(LEAST(usage_frequency / 100.0, 1.0) * 0.4 +
 LEAST(downstream_dependencies / 10.0, 1.0) * 0.6)
```

**Data quality** -- composite of four dimensions:

| Dimension | Weight | Source |
|-----------|--------|--------|
| Completeness | 30% | Null ratio calculation |
| Accuracy | 30% | SLA pass/fail from monitoring |
| Consistency | 20% | Referential integrity test results |
| Timeliness | 20% | Minutes since last sync |

**Overall impact** -- the composite score used for prioritisation:

```
overall = business_impact * 0.4
        + data_quality    * 0.3
        + performance     * 0.2
        + cost            * 0.1
```

### Criticality and Risk Classification

| Overall Impact Score | Criticality | Business Priority |
|---------------------|-------------|-------------------|
| >= 0.8 | CRITICAL | P0 |
| >= 0.6 | HIGH | P1 |
| >= 0.4 | MEDIUM | P2 |
| < 0.4 | LOW | P3 |

Risk level is derived from quality and performance thresholds:
- **HIGH_RISK** -- data quality score below 0.8
- **MEDIUM_RISK** -- performance impact above 0.7 or cost impact above 0.8
- **LOW_RISK** -- all scores within acceptable ranges

### Lineage Metadata (VARIANT Column)

Each lineage record carries structured metadata in a Snowflake VARIANT column:
- `transformation_rules` -- array of applied transformations (e.g., `data_type_conversion`, `null_handling`)
- `business_rules` -- array of business logic applied (e.g., `profit_margin_calculation`)
- `dependencies` -- array of upstream table names
- `sla_requirements` -- freshness and quality thresholds (e.g., `freshness_hours: 2, quality_threshold: 0.95`)

### Automated Lineage Alerts

A stored procedure checks for high-risk lineage issues and inserts alerts into the monitoring schema:

```sql
-- Fires when any lineage entry has HIGH_RISK and quality < 0.8
CALL GOVERNANCE.SP_check_lineage_impact_alerts();
```

This should be scheduled as a Snowflake Task running on a regular cadence (e.g., every 4 hours).

---

## Data Retention and Deletion Policies

General best practices for data retention across platforms.

### Retention Tier Model

| Tier | Retention Period | Storage Class | Use Case |
|------|-----------------|---------------|----------|
| Hot | 0--90 days | Standard / Premium | Active queries, dashboards |
| Warm | 90 days -- 1 year | Infrequent Access / Cool | Ad-hoc analysis, audit |
| Cold | 1--7 years | Archive / Glacier | Regulatory compliance |
| Tombstone | Post-retention | Deleted + audit record | GDPR/privacy compliance |

### Platform-Specific Retention Mechanisms

**Snowflake:**
- `DATA_RETENTION_TIME_IN_DAYS` on tables (Time Travel) -- default 1 day, up to 90 days on Enterprise edition
- `FAIL-SAFE` -- 7 additional days of non-queryable recovery (not configurable)
- Use `COPY INTO @stage` for long-term archival to cloud storage

**GCP (BigQuery / Cloud Storage):**
- BigQuery table expiration: `expiration_timestamp` on tables
- Cloud Storage lifecycle rules: transition to Nearline/Coldline/Archive after N days
- Object versioning for audit trails

**Azure (Fabric / ADLS):**
- ADLS Gen2 lifecycle management policies (hot -> cool -> archive)
- Fabric lakehouse: Delta table `VACUUM` to remove old files (default 7-day retention)
- Soft delete on storage accounts for recovery

### Deletion Policy Patterns

1. **Logical deletion** -- set a `deleted_at` timestamp; filter in views. Preserves audit trail.
2. **Physical deletion with tombstone** -- delete the record, insert a tombstone row in an audit table with the deletion reason and timestamp.
3. **Right to erasure (GDPR Article 17)** -- requires physical deletion of PII across all tiers (raw, staging, marts, backups). Use lineage metadata to identify all downstream copies.
4. **Cascade deletion** -- when a source record is deleted, propagate deletion to all downstream tables identified in the lineage graph.

### Retention Policy Checklist

- [ ] Define retention periods per data classification tier
- [ ] Automate tier transitions (hot to warm to cold)
- [ ] Implement deletion procedures with audit logging
- [ ] Test recovery from each storage tier
- [ ] Document regulatory requirements per jurisdiction
- [ ] Map lineage for all PII columns to support right-to-erasure requests

---

## Cross-Platform IAM Comparison

### AWS IAM vs Azure RBAC vs GCP IAM

| Aspect | AWS IAM | Azure RBAC | GCP IAM |
|--------|---------|------------|---------|
| **Identity model** | Users, groups, roles | Users, groups, service principals, managed identities | Users, groups, service accounts |
| **Policy attachment** | Policies attached to users/groups/roles (JSON documents) | Role assignments at scope (subscription, resource group, resource) | Role bindings at scope (organisation, folder, project, resource) |
| **Policy language** | JSON with Effect/Action/Resource/Condition | Built-in role definitions (JSON); custom roles supported | Predefined roles + custom roles; IAM Conditions for context-aware access |
| **Least privilege tool** | IAM Access Analyzer, unused permissions alerts | Entra ID access reviews, PIM (Privileged Identity Management) | IAM Recommender, Policy Analyser |
| **Service identity** | IAM Roles (assumed by services), instance profiles | Managed Identity (system-assigned or user-assigned) | Service accounts + Workload Identity Federation |
| **Key management** | Access keys (long-lived), STS temporary credentials | Client secrets, certificates, managed identity (keyless) | Service account keys (discouraged), Workload Identity (keyless) |
| **Cross-account/project** | AssumeRole across accounts | Lighthouse, cross-tenant access | Cross-project IAM bindings, Workload Identity pools |
| **Conditional access** | IAM Conditions (IP, MFA, time, tags) | Conditional Access policies (Entra ID) | IAM Conditions (IP, time, resource attributes) |
| **Audit** | CloudTrail | Azure Activity Log, Entra ID sign-in logs | Cloud Audit Logs |
| **Data platform roles** | Lake Formation permissions, Redshift roles | Fabric workspace roles, Synapse RBAC | BigQuery IAM roles, dataset-level access |

### Choosing the Right Pattern

| Scenario | Recommended Approach |
|----------|---------------------|
| Single cloud, all services | Use native IAM with predefined roles |
| Multi-cloud data platform | Federate identities; use Workload Identity (GCP), Managed Identity (Azure), IAM Roles (AWS) |
| CI/CD pipeline access | Short-lived tokens via OIDC federation (all three clouds support this) |
| Cross-team data sharing | Snowflake shares, BigQuery authorised views, Fabric workspace sharing |
| Regulatory compliance (SOX, HIPAA) | Enable audit logging on all platforms; use IAM Conditions to restrict access windows |

### Service Identity Best Practices (All Platforms)

1. **Prefer keyless authentication** -- Workload Identity (GCP), Managed Identity (Azure), IAM Roles with STS (AWS)
2. **One identity per service** -- never share service accounts across unrelated workloads
3. **Rotate credentials** -- if keys are unavoidable, automate rotation on a 90-day cycle
4. **Scope to minimum resources** -- bind roles at the narrowest possible scope (resource > project > folder > organisation)
5. **Audit continuously** -- enable and review access logs; alert on anomalous patterns
6. **Use conditions** -- restrict by source IP, time window, or resource attribute wherever possible

---

## Row-Level Security Across Platforms

### Pattern Comparison

| Platform | RLS Mechanism | Predicate Location | Identity Function |
|----------|--------------|-------------------|-------------------|
| Snowflake | Row Access Policies | Policy on table/view | `CURRENT_ROLE()`, `CURRENT_USER()` |
| Fabric Warehouse | Security Policy + Filter Predicate | T5 presentation views | `USER_NAME()` |
| Fabric Lakehouse | Delta Row Filter | Table properties | `current_setting('app.region')` |
| Power BI | DAX role filter | Semantic model | `USERNAME()`, `USERPRINCIPALNAME()` |
| BigQuery | Authorised views + row-level access policies | View or policy | `SESSION_USER()` |

### Design Principles for Cross-Platform RLS

1. **Centralise access mappings** -- maintain a single source of truth for user-to-scope mappings (e.g., user-to-region, user-to-department). Store in a control table (T0) or identity provider.
2. **Apply at the consumption layer** -- filter data at the presentation/serving layer, not at ingestion or transformation.
3. **Test with impersonation** -- use `EXECUTE AS USER` (Fabric), `EXECUTE AS CALLER` (Snowflake), or test role switching to verify predicates.
4. **Document predicates** -- maintain a registry of all active RLS policies, their target tables, and the business rules they enforce.
5. **Performance test** -- RLS predicates add overhead to every query. Benchmark with realistic data volumes and user counts.

---

## Security Gap Summary

| Gap | Platform | Severity | Recommendation |
|-----|----------|----------|----------------|
| No IAM Conditions on GCP bindings | GCP | Medium | Add `condition` blocks for time/IP restrictions |
| Project-level bindings only | GCP | Medium | Use folder/org-level for shared services |
| dbt roles overly broad in Snowflake | Snowflake | High | Restrict to specific schemas per environment |
| No dynamic data masking in logistics platform | Snowflake | High | Apply `CREATE MASKING POLICY` for PII_HIGH columns |
| No RLS in logistics platform | Snowflake | Medium | Add row access policies if multi-tenant |
| Lakehouse RLS not yet implemented | Azure/Fabric | Medium | Implement Delta row filters for regional access |
| No automated credential rotation | All | Medium | Automate 90-day rotation or migrate to keyless |
| Lineage alerts not scheduled | Snowflake | Low | Schedule `SP_check_lineage_impact_alerts` as a Task |

---

## Data Masking Patterns

Data masking replaces sensitive values with realistic but non-identifiable substitutes. It is a key control for protecting PII, PHI, and other regulated data both in production queries and in non-production environments.

### Static vs Dynamic Masking

| Aspect | Static Masking | Dynamic Masking |
|--------|---------------|-----------------|
| **When applied** | At data copy/export time | At query time |
| **Original data** | Permanently altered in the copy | Unchanged in storage |
| **Use case** | Non-prod environments, test data, analytics sandboxes | Production queries where different roles see different views |
| **Performance** | No query-time overhead | Slight overhead per query (policy evaluation) |
| **Reversibility** | Irreversible (one-way) | Role-dependent — privileged roles see original values |

### Masking Techniques

| Technique | Description | Example | When to Use |
|-----------|-------------|---------|-------------|
| **Hash** | Deterministic one-way hash (SHA-256) | `john@acme.com` → `a3f2b8c...` | Pseudonymisation; preserves join capability |
| **Redact** | Replace with fixed string or partial mask | `john@acme.com` → `****@****.com` | Display-level masking for dashboards |
| **Tokenise** | Replace with random token; store mapping in vault | `john@acme.com` → `tok_9x2k4` | Reversible masking; supports re-identification by authorised parties |
| **Generalise** | Reduce precision | `1987-03-15` → `1987-01-01` (year only) | k-anonymity; statistical analysis |
| **Noise injection** | Add random offset to numeric values | `salary: 85000` → `salary: 83412` | Differential privacy; aggregate accuracy preserved |
| **Shuffle** | Randomly reassign values within a column | Salaries redistributed across rows | Preserves distribution; breaks individual linkage |
| **Nullify** | Replace with NULL | `SSN: 123-45-6789` → `NULL` | Simplest approach; column becomes unusable |

### Platform Implementations

#### Snowflake Masking Policies

Snowflake applies dynamic masking at query time via `CREATE MASKING POLICY`:

```sql
-- Create a masking policy for email addresses
CREATE OR REPLACE MASKING POLICY mask_email AS (val STRING)
RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('DATA_ENGINEER', 'ANALYST_FULL')
            THEN val
        WHEN CURRENT_ROLE() = 'ANALYST_RESTRICTED'
            THEN REGEXP_REPLACE(val, '.+@', '****@')
        ELSE '****'
    END;

-- Apply to a column
ALTER TABLE customers MODIFY COLUMN email
    SET MASKING POLICY mask_email;

-- Conditional masking based on data classification tag
CREATE OR REPLACE MASKING POLICY mask_by_tag AS (val STRING)
RETURNS STRING ->
    CASE
        WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('pii_classification') = 'PUBLIC'
            THEN val
        ELSE SHA2(val)
    END;
```

#### Databricks Column Masks

Databricks Unity Catalog supports column masks on Delta tables:

```sql
-- Create a masking function
CREATE FUNCTION mask_ssn(ssn STRING)
RETURNS STRING
RETURN
    CASE
        WHEN IS_MEMBER('pii_viewers') THEN ssn
        ELSE CONCAT('***-**-', RIGHT(ssn, 4))
    END;

-- Apply mask to a column
ALTER TABLE customers
    ALTER COLUMN ssn SET MASK mask_ssn;
```

#### BigQuery Column-Level Security

BigQuery uses policy tags from Data Catalog for column-level access control:

```sql
-- Policy tags are applied via Data Catalog taxonomy (Terraform or Console)
-- Column-level access is enforced by IAM binding on the policy tag

-- Terraform example
resource "google_bigquery_table" "customers" {
  schema = jsonencode([
    {
      name = "email"
      type = "STRING"
      policyTags = {
        names = ["projects/my-project/locations/eu/taxonomies/123/policyTags/456"]
      }
    }
  ])
}

-- Only users with Fine-Grained Reader role on the policy tag can see the column
-- Others receive an access denied error (not masked — fully blocked)
```

### Masking in Non-Production Environments

Non-prod environments require static masking to prevent PII leakage into development, testing, and staging systems.

**Pattern: ETL-based static masking pipeline**

```python
import hashlib

MASKING_RULES = {
    "email":       lambda v: hashlib.sha256(v.encode()).hexdigest()[:16] + "@masked.local",
    "phone":       lambda v: "+00-000-" + v[-4:] if v else None,
    "first_name":  lambda v: "User_" + hashlib.md5(v.encode()).hexdigest()[:6],
    "last_name":   lambda v: "Surname_" + hashlib.md5(v.encode()).hexdigest()[:6],
    "date_of_birth": lambda v: v[:4] + "-01-01" if v else None,  # Generalise to year
    "salary":      lambda v: round(v * (0.9 + (hash(str(v)) % 20) / 100), 2),  # Noise
}

def mask_dataframe(df, rules=MASKING_RULES):
    """Apply static masking rules to a pandas DataFrame."""
    masked = df.copy()
    for col, mask_fn in rules.items():
        if col in masked.columns:
            masked[col] = masked[col].apply(lambda v: mask_fn(v) if v else v)
    return masked
```

**Non-prod masking checklist:**

- [ ] Mask all PII columns before copying to lower environments
- [ ] Maintain referential integrity (use deterministic hashing for join keys)
- [ ] Validate masked data still passes schema and type checks
- [ ] Automate masking as part of the environment refresh pipeline
- [ ] Never store masking reversal keys in the same environment as masked data
- [ ] Audit and log all masking operations for compliance evidence

---

## Related Notes

- [[Snowflake RBAC & Data Security]] -- detailed Snowflake role hierarchy, data classification, and access auditing
- [[Terraform Infrastructure as Code]] -- GCP Terraform module patterns including IAM
- [[Microsoft Fabric Architecture]] -- T0-T5 architecture and Fabric security patterns
- [[Data Contracts & Schema Evolution]] -- governance patterns for schema changes
- [[Data Quality & Testing]] -- quality scoring dimensions referenced in lineage impact scoring
