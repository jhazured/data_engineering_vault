---
tags:
  - security
  - governance
  - compliance
  - privacy
  - gdpr
  - hipaa
  - pci-dss
  - soc2
  - data-classification
  - encryption
---

# Data Compliance & Privacy Frameworks

This note covers the major regulatory and industry compliance frameworks that data engineers encounter when building and maintaining data platforms. Understanding these frameworks is essential for designing pipelines that handle sensitive data lawfully and securely.

Related notes: [[Snowflake RBAC & Data Security]], [[Cross-Platform IAM & Data Governance]], [[Data Cataloguing & Discovery]], [[Trust Stores & Certificate Management]]

---

## GDPR (General Data Protection Regulation)

The EU's General Data Protection Regulation (2018) governs the processing of personal data for individuals within the European Economic Area. It applies to any organisation that processes EU residents' data, regardless of where the organisation is based.

### Key Principles

| Principle | Description |
|-----------|-------------|
| **Lawfulness, Fairness & Transparency** | Data must be processed lawfully with a valid legal basis (consent, contract, legitimate interest, etc.) and in a transparent manner |
| **Purpose Limitation** | Data must be collected for specified, explicit, and legitimate purposes and not further processed in a manner incompatible with those purposes |
| **Data Minimisation** | Only data that is adequate, relevant, and limited to what is necessary for the stated purpose should be collected |
| **Accuracy** | Personal data must be kept accurate and up to date; inaccurate data must be erased or rectified without delay |
| **Storage Limitation** | Data must not be kept in identifiable form for longer than necessary for the stated purpose |
| **Integrity & Confidentiality** | Data must be processed with appropriate security measures, including protection against unauthorised access, loss, or destruction |
| **Accountability** | The data controller must demonstrate compliance with all principles |

### Data Subject Rights

Data subjects (individuals whose data is processed) have the following rights under GDPR:

1. **Right of Access (Art. 15)** -- individuals can request a copy of their personal data and information about how it is processed
2. **Right to Rectification (Art. 16)** -- individuals can request correction of inaccurate personal data
3. **Right to Erasure / Right to Be Forgotten (Art. 17)** -- individuals can request deletion of their personal data under certain conditions
4. **Right to Restriction of Processing (Art. 18)** -- individuals can request that processing be limited
5. **Right to Data Portability (Art. 20)** -- individuals can receive their data in a structured, machine-readable format
6. **Right to Object (Art. 21)** -- individuals can object to processing based on legitimate interests or direct marketing
7. **Rights Related to Automated Decision-Making (Art. 22)** -- individuals can request human intervention for decisions made solely by automated means

### Data Protection Officer (DPO)

A DPO must be appointed when:

- The organisation is a public authority
- Core activities involve large-scale systematic monitoring of individuals
- Core activities involve large-scale processing of special category data (health, biometric, racial/ethnic origin, etc.)

The DPO reports directly to the highest level of management, operates independently, and acts as the point of contact for supervisory authorities.

### Breach Notification

GDPR mandates a **72-hour notification window** for personal data breaches:

1. **To the supervisory authority** -- within 72 hours of becoming aware of the breach (unless the breach is unlikely to result in a risk to individuals' rights)
2. **To affected data subjects** -- without undue delay when the breach is likely to result in a high risk to their rights and freedoms

Data engineers must ensure that breach detection mechanisms (audit logs, anomaly detection, access monitoring) are built into pipeline infrastructure.

### Implications For Data Pipelines

| Area | Requirement |
|------|-------------|
| **Pseudonymisation** | Replace identifying fields with artificial identifiers; store the mapping separately with restricted access |
| **Encryption** | Encrypt personal data at rest and in transit; use envelope encryption with managed key services |
| **Audit Trails** | Log all access to personal data, including who accessed it, when, and for what purpose |
| **Data Lineage** | Maintain clear lineage showing where personal data flows through the platform -- see [[Data Cataloguing & Discovery]] |
| **Retention Policies** | Implement automated deletion or anonymisation after the defined retention period expires |
| **Cross-Border Transfers** | Ensure data transferred outside the EEA has adequate safeguards (Standard Contractual Clauses, adequacy decisions) |

---

## SOC 2 (System and Organisation Controls 2)

SOC 2 is an auditing framework developed by the American Institute of Certified Public Accountants (AICPA). It evaluates an organisation's controls relevant to security, availability, processing integrity, confidentiality, and privacy.

### Trust Service Criteria

| Criterion | Description | Data Engineering Relevance |
|-----------|-------------|---------------------------|
| **Security** | Protection against unauthorised access (logical and physical) | Access controls, network segmentation, encryption, vulnerability management |
| **Availability** | System is available for operation and use as committed | SLAs, disaster recovery, failover, monitoring and alerting |
| **Processing Integrity** | System processing is complete, valid, accurate, timely, and authorised | Data validation, reconciliation, idempotent pipelines, error handling |
| **Confidentiality** | Information designated as confidential is protected as committed | Data classification, encryption, access restrictions, secure disposal |
| **Privacy** | Personal information is collected, used, retained, disclosed, and disposed of in conformity with commitments | Consent management, data subject rights, privacy notices |

### Type I vs Type II

| Aspect | Type I | Type II |
|--------|--------|---------|
| **Scope** | Design of controls at a point in time | Design and operating effectiveness over a period (typically 6-12 months) |
| **Evidence** | Controls are suitably designed | Controls are suitably designed **and** operating effectively |
| **Value** | Useful as a starting point; faster to obtain | More rigorous; preferred by enterprise customers and partners |
| **Audit Duration** | Weeks | Months of observation |

### Implications For Data Engineering

- **Access Controls** -- enforce least-privilege access to all data stores and pipelines; use role-based access control (RBAC) with regular access reviews -- see [[Snowflake RBAC & Data Security]]
- **Change Management** -- all pipeline changes must go through version-controlled CI/CD with approval gates; no ad-hoc production changes
- **Monitoring & Alerting** -- implement comprehensive monitoring for pipeline failures, data quality anomalies, and unauthorised access attempts
- **Incident Response** -- document and maintain runbooks for data incidents; log all incidents and their resolution
- **Vendor Management** -- assess third-party tools and services (Fivetran, dbt Cloud, etc.) for their own SOC 2 compliance status

---

## HIPAA (Health Insurance Portability and Accountability Act)

HIPAA governs the use and disclosure of Protected Health Information (PHI) in the United States. It applies to covered entities (healthcare providers, health plans, healthcare clearinghouses) and their business associates.

### Protected Health Information (PHI)

PHI is any individually identifiable health information that is created, received, maintained, or transmitted by a covered entity. This includes:

- Patient names, addresses, dates of birth, Social Security numbers
- Medical record numbers, health plan beneficiary numbers
- Diagnoses, treatment information, lab results
- Billing and payment information linked to an individual

**Electronic PHI (ePHI)** is PHI stored or transmitted electronically -- the primary concern for data engineers.

### Technical Safeguards

| Safeguard | Requirement | Implementation |
|-----------|-------------|----------------|
| **Access Control** | Unique user identification, emergency access procedures, automatic logoff, encryption/decryption | RBAC, MFA, session timeouts, column-level encryption |
| **Audit Controls** | Hardware, software, and procedural mechanisms to record and examine access to ePHI | Comprehensive access logging, log retention (minimum 6 years), regular log review |
| **Integrity Controls** | Policies and procedures to protect ePHI from improper alteration or destruction | Checksums, digital signatures, immutable audit logs |
| **Transmission Security** | Technical security measures to guard against unauthorised access to ePHI during transmission | TLS 1.2+, VPN tunnels, encrypted file transfers -- see [[Trust Stores & Certificate Management]] |

### Business Associate Agreements (BAAs)

Any third party that creates, receives, maintains, or transmits PHI on behalf of a covered entity must sign a BAA. This includes cloud providers (AWS, Azure, GCP all offer BAAs), SaaS tools, and consulting firms.

Data engineers must verify that **every component in the data pipeline** that touches PHI is covered by a BAA, including:

- Cloud storage (S3, ADLS, GCS)
- Data warehouses (Snowflake, Databricks, BigQuery)
- Orchestration tools (Airflow, dbt Cloud)
- Monitoring and logging platforms

### De-Identification Methods

HIPAA defines two methods for de-identifying health data, after which the data is no longer considered PHI:

**Safe Harbor Method (Section 164.514(b)(2)):**

Remove all 18 specified identifiers:

1. Names
2. Geographic data smaller than a state
3. Dates (except year) related to an individual
4. Phone numbers
5. Fax numbers
6. Email addresses
7. Social Security numbers
8. Medical record numbers
9. Health plan beneficiary numbers
10. Account numbers
11. Certificate/licence numbers
12. Vehicle identifiers and serial numbers
13. Device identifiers and serial numbers
14. Web URLs
15. IP addresses
16. Biometric identifiers
17. Full-face photographs and comparable images
18. Any other unique identifying number, characteristic, or code

**Expert Determination Method (Section 164.514(b)(1)):**

A qualified statistical expert determines that the risk of identifying an individual from the data is very small. This method is more flexible but requires documented expert analysis.

---

## PCI-DSS (Payment Card Industry Data Security Standard)

PCI-DSS applies to any organisation that stores, processes, or transmits cardholder data. It is maintained by the PCI Security Standards Council (founded by Visa, Mastercard, American Express, Discover, and JCB).

### Cardholder Data Environment (CDE)

The CDE encompasses all people, processes, and technologies that store, process, or transmit cardholder data or sensitive authentication data. Minimising the CDE through network segmentation and tokenisation reduces compliance scope.

| Data Element | Storage Permitted | Protection Required |
|-------------|-------------------|---------------------|
| Primary Account Number (PAN) | Yes | Must be rendered unreadable (encryption, hashing, truncation, or tokenisation) |
| Cardholder Name | Yes | Must be protected if stored with PAN |
| Service Code | Yes | Must be protected if stored with PAN |
| Expiry Date | Yes | Must be protected if stored with PAN |
| Full Magnetic Stripe / CVV / PIN | **No** | Must never be stored after authorisation |

### The 12 Requirements (Overview)

| Category | Requirement |
|----------|-------------|
| **Build & Maintain a Secure Network** | 1. Install and maintain network security controls |
| | 2. Apply secure configurations to all system components |
| **Protect Account Data** | 3. Protect stored account data |
| | 4. Protect cardholder data with strong cryptography during transmission |
| **Maintain a Vulnerability Management Programme** | 5. Protect all systems and networks from malicious software |
| | 6. Develop and maintain secure systems and software |
| **Implement Strong Access Control** | 7. Restrict access to system components and cardholder data by business need to know |
| | 8. Identify users and authenticate access to system components |
| | 9. Restrict physical access to cardholder data |
| **Monitor & Test Networks** | 10. Log and monitor all access to system components and cardholder data |
| | 11. Test security of systems and networks regularly |
| **Maintain an Information Security Policy** | 12. Support information security with organisational policies and programmes |

### Tokenisation

Tokenisation replaces cardholder data with a non-sensitive surrogate value (token) that has no exploitable meaning. Unlike encryption, tokenisation does not use a mathematical relationship between the original data and the token.

```
Original PAN:  4532 1234 5678 9012
Token:         tok_a8f3c2e1b4d7
```

Tokenisation effectively removes systems from PCI-DSS scope because they no longer store, process, or transmit actual cardholder data.

### Network Segmentation

Isolate the CDE from the rest of the network to reduce PCI-DSS scope:

- Place cardholder data stores in dedicated VPCs/subnets with strict firewall rules
- Use jump hosts or bastion servers for administrative access
- Ensure data warehouses that receive tokenised (not raw) card data sit outside the CDE

### Implications For Data Warehouses

- **Never store raw PANs** in analytical data stores; use tokenised or truncated values
- **Separate environments** -- ensure CDE-scoped systems are isolated from general analytical workloads
- **Access logging** -- all queries against tables containing cardholder data must be logged and monitored
- **Key management** -- encryption keys must be managed separately from the data they protect; rotate keys according to policy

---

## Data Classification

A data classification framework assigns sensitivity levels to data assets, determining the security controls and handling requirements for each tier.

### Classification Tiers

| Tier | Label | Description | Examples |
|------|-------|-------------|----------|
| **Tier 1** | Public | Data intended for public disclosure; no business impact if exposed | Marketing materials, published reports, public APIs |
| **Tier 2** | Internal | Data for internal use only; minor impact if disclosed | Internal dashboards, aggregated metrics, org charts |
| **Tier 3** | Confidential | Sensitive business data; significant impact if disclosed | Financial reports, customer lists, contracts, trade secrets |
| **Tier 4** | Restricted | Highly sensitive data; severe impact if disclosed; subject to regulatory requirements | PII, PHI, cardholder data, credentials, encryption keys |

### Handling Requirements By Tier

| Control | Public | Internal | Confidential | Restricted |
|---------|--------|----------|--------------|------------|
| **Encryption at Rest** | Optional | Recommended | Required | Required (AES-256 or equivalent) |
| **Encryption in Transit** | Recommended | Required | Required | Required (TLS 1.2+) |
| **Access Control** | Open | Role-based | Role-based with approval | Named individuals with MFA |
| **Audit Logging** | Not required | Recommended | Required | Required with real-time alerting |
| **Data Masking** | Not required | Not required | Required in non-production | Required everywhere except authorised production |
| **Retention Policy** | Business discretion | Defined | Defined and enforced | Defined, enforced, and audited |
| **Disposal Method** | Standard deletion | Standard deletion | Secure deletion | Cryptographic erasure or certified destruction |
| **Backup Requirements** | Business discretion | Standard | Encrypted backups | Encrypted backups with restricted access |

Data classification should be applied as metadata tags on tables, columns, and datasets within the data catalogue -- see [[Data Cataloguing & Discovery]].

---

## Implementation Patterns For Data Engineers

### Column-Level Encryption And Masking

Apply encryption or masking at the column level to protect sensitive fields whilst keeping non-sensitive columns accessible:

```sql
-- Snowflake: Dynamic Data Masking policy
CREATE OR REPLACE MASKING POLICY pii_mask AS (val STRING)
  RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('ANALYST', 'ENGINEER') THEN '***MASKED***'
    WHEN CURRENT_ROLE() IN ('DATA_STEWARD', 'COMPLIANCE') THEN val
    ELSE '***MASKED***'
  END;

ALTER TABLE customers MODIFY COLUMN email
  SET MASKING POLICY pii_mask;
```

```sql
-- Column-level encryption using a key stored in a secrets manager
INSERT INTO customers (id, name, email_encrypted)
VALUES (
  1,
  'Jane Smith',
  ENCRYPT(email_value, $encryption_key)  -- platform-specific function
);
```

**Key considerations:**

- Masking is preferable for analytical workloads where the raw value is not needed
- Encryption is necessary when the raw value must be recoverable by authorised processes
- Combine both approaches: encrypt at rest, mask at query time for non-privileged roles

### Data Retention And Deletion Pipelines

Implement automated retention enforcement to comply with storage limitation requirements:

```python
# Pseudocode: Retention enforcement pipeline
def enforce_retention(table: str, retention_days: int):
    """
    Soft-delete or hard-delete records that have exceeded
    their retention period.
    """
    cutoff_date = current_date() - timedelta(days=retention_days)

    # Option 1: Soft delete (mark as expired)
    execute(f"""
        UPDATE {table}
        SET is_deleted = TRUE,
            deleted_at = CURRENT_TIMESTAMP()
        WHERE created_at < '{cutoff_date}'
          AND is_deleted = FALSE
    """)

    # Option 2: Hard delete (permanent removal)
    execute(f"""
        DELETE FROM {table}
        WHERE created_at < '{cutoff_date}'
    """)

    # Option 3: Anonymise (replace PII with null/placeholder)
    execute(f"""
        UPDATE {table}
        SET email = NULL,
            name = 'ANONYMISED',
            phone = NULL
        WHERE created_at < '{cutoff_date}'
    """)

    log_retention_action(table, cutoff_date, rows_affected)
```

Retention policies should be defined per data classification tier and documented in the data catalogue.

### Audit Logging For Access Tracking

Every access to sensitive data should be logged with sufficient detail for forensic analysis:

| Field | Description |
|-------|-------------|
| `timestamp` | When the access occurred (UTC) |
| `user_id` | Who accessed the data (authenticated identity) |
| `role` | The role used for access |
| `action` | Read, write, delete, export |
| `table_name` | The table or dataset accessed |
| `columns_accessed` | Specific columns queried (where trackable) |
| `row_count` | Number of rows returned or affected |
| `query_id` | Reference to the full query text |
| `source_ip` | IP address of the client |

Audit logs themselves are Tier 4 (Restricted) data and must be stored in an immutable, tamper-proof location with their own access controls.

### Right To Be Forgotten Implementation

Three approaches for handling erasure requests, each with trade-offs:

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **Soft Delete** | Set a `deleted` flag; exclude from queries via views/policies | Reversible; maintains referential integrity; simple to implement | Data still exists; does not satisfy strict GDPR interpretation; increases storage |
| **Hard Delete** | Physically remove records from all tables and backups | Fully compliant; reduces storage | Irreversible; breaks referential integrity; complex across denormalised models; backup purging is difficult |
| **Anonymisation** | Replace identifying fields with irreversible placeholders | Preserves aggregate analytics; satisfies GDPR; maintains referential integrity | Irreversible; must ensure true anonymisation (no re-identification risk) |

**Recommended pattern:** Anonymisation for analytical data stores, hard delete for operational/transactional stores.

Implementation steps:

1. **Identify all tables** containing the data subject's personal data (requires robust data lineage)
2. **Cascade through dimensions and facts** -- ensure the request propagates to all downstream tables
3. **Handle slowly changing dimensions** -- anonymise historical records in SCD Type 2 tables
4. **Purge from backups** within a defined timeframe (or accept and document the risk)
5. **Log the erasure** -- maintain an audit record that the request was fulfilled (without storing the deleted data)
6. **Verify completion** -- run a validation query to confirm no residual personal data remains

### Environment Data Controls

Prevent PII and other sensitive data from appearing in non-production environments:

| Control | Implementation |
|---------|----------------|
| **Synthetic data generation** | Use tools like Faker or Gretel to generate realistic but artificial test data |
| **Data subsetting** | Extract a representative subset with all sensitive columns masked or removed |
| **Dynamic masking in lower environments** | Apply masking policies that are always active in dev/test (no privileged bypass) |
| **Automated scanning** | Run PII detection scans (AWS Macie, Azure Purview) against dev/test environments on a schedule |
| **Access restrictions** | Production data stores should be inaccessible from development networks |

```sql
-- Example: Create a masked view for dev/test consumption
CREATE OR REPLACE VIEW dev_customers AS
SELECT
    id,
    MD5(email) AS email_hash,
    CONCAT(LEFT(name, 1), '****') AS masked_name,
    date_of_birth,  -- consider masking if classification requires it
    region,
    signup_date
FROM prod_customers;
```

---

## Platform-Specific Compliance

### Snowflake

See also: [[Snowflake RBAC & Data Security]]

**Dynamic Data Masking:**

Snowflake supports masking policies that dynamically mask column values based on the querying user's role. Policies are defined once and applied to columns across multiple tables.

```sql
-- Tag-based masking: apply policies via object tags
ALTER TAG pii_tag SET MASKING POLICY pii_full_mask;

-- Apply the tag to columns
ALTER TABLE customers MODIFY COLUMN email SET TAG pii_tag = 'email';
ALTER TABLE customers MODIFY COLUMN phone SET TAG pii_tag = 'phone';
```

**Tag-Based Policies:**

Snowflake's object tagging system enables governance at scale:

- Tags can be applied to databases, schemas, tables, columns, and warehouses
- Tags propagate through lineage (tag inheritance)
- Combine with masking policies and row access policies for automated governance

**Access History (ACCOUNT_USAGE.ACCESS_HISTORY):**

```sql
-- Query access history for compliance auditing
SELECT
    query_id,
    user_name,
    direct_objects_accessed,
    base_objects_accessed,
    query_start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE query_start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND ARRAY_CONTAINS('CUSTOMERS'::VARIANT,
      TRANSFORM(base_objects_accessed, x -> x:objectName))
ORDER BY query_start_time DESC;
```

This view provides column-level access tracking without additional instrumentation.

### Databricks

**Unity Catalog Column Masking:**

Unity Catalog supports column-level masking functions applied declaratively:

```sql
-- Create a masking function
CREATE FUNCTION mask_email(email STRING)
  RETURNS STRING
  RETURN CASE
    WHEN IS_MEMBER('compliance_team') THEN email
    ELSE CONCAT(LEFT(email, 2), '***@***.***')
  END;

-- Apply to a column
ALTER TABLE customers ALTER COLUMN email
  SET MASK mask_email;
```

**Row Filters:**

```sql
-- Row-level security via row filters
CREATE FUNCTION region_filter(region STRING)
  RETURNS BOOLEAN
  RETURN (IS_MEMBER('global_team') OR region = CURRENT_USER_ATTRIBUTE('region'));

ALTER TABLE sales
  SET ROW FILTER region_filter ON (region);
```

Unity Catalog also provides:

- **Data lineage** at the column level across notebooks, jobs, and SQL queries
- **Audit logging** via the system tables (`system.access.audit`)
- **Information schema** for discovering tagged and classified objects

### AWS

| Service | Compliance Function |
|---------|---------------------|
| **KMS (Key Management Service)** | Centralised key management for encryption at rest; supports automatic key rotation; integrates with S3, RDS, Redshift, DynamoDB |
| **Macie** | Automated PII and sensitive data discovery across S3 buckets; uses machine learning to classify data; generates findings for remediation |
| **GuardDuty** | Threat detection service that monitors for malicious activity and unauthorised behaviour; analyses CloudTrail, VPC Flow Logs, and DNS logs |
| **CloudTrail** | API-level audit logging for all AWS service calls; essential for SOC 2 and HIPAA compliance |
| **Config** | Tracks resource configuration changes; enables compliance rules that automatically flag non-compliant resources |
| **Lake Formation** | Fine-grained access control for data lakes; column-level and row-level security for data stored in S3 and queried via Athena, Redshift Spectrum, or Glue |

### Azure

| Service | Compliance Function |
|---------|---------------------|
| **Purview** | Unified data governance service; automated data discovery, classification, and lineage; sensitivity labels integrated with Microsoft 365 |
| **Information Protection** | Sensitivity labels applied to documents, emails, and data assets; integrates with Purview for automated classification |
| **Key Vault** | Centralised secrets and key management; HSM-backed keys; integrates with Azure SQL, Storage, and Databricks |
| **Defender for Cloud** | Security posture management and threat detection across Azure resources |
| **Policy** | Azure Policy enables organisation-wide compliance rules (e.g., enforce encryption on all storage accounts, restrict regions for data residency) |
| **Monitor / Log Analytics** | Centralised logging and alerting; diagnostic settings capture access logs from Azure SQL, ADLS, and Synapse |

See also: [[Cross-Platform IAM & Data Governance]] for detailed IAM patterns across cloud providers.

---

## Compliance Checklist For New Data Pipeline Projects

Use this checklist when designing and implementing any new data pipeline that handles sensitive or regulated data.

### Data Discovery And Classification

- [ ] Identify all data sources and their data classification tiers
- [ ] Catalogue all PII, PHI, and cardholder data fields in the pipeline
- [ ] Document the legal basis for processing personal data (GDPR)
- [ ] Confirm data processing agreements / BAAs are in place with all third-party processors
- [ ] Tag all sensitive columns in the data catalogue with appropriate classification labels

### Access Controls

- [ ] Implement RBAC with least-privilege access for all pipeline components
- [ ] Enable MFA for all human access to production data stores
- [ ] Use service accounts with scoped permissions for automated pipeline processes
- [ ] Conduct and document initial access review; schedule recurring reviews (quarterly)
- [ ] Ensure no shared credentials or embedded secrets in code

### Encryption And Data Protection

- [ ] Enable encryption at rest for all data stores (AES-256 or equivalent)
- [ ] Enable encryption in transit (TLS 1.2+ for all connections)
- [ ] Implement column-level encryption or masking for Tier 4 (Restricted) data
- [ ] Store encryption keys in a managed key service (KMS, Key Vault) with rotation policies
- [ ] Verify that backup and disaster recovery stores are also encrypted

### Audit And Monitoring

- [ ] Enable access logging on all data stores (query logs, access history)
- [ ] Configure alerting for anomalous access patterns (unusual query volumes, after-hours access, bulk exports)
- [ ] Ensure audit logs are stored in an immutable, tamper-proof location
- [ ] Define log retention periods that meet regulatory requirements (HIPAA: 6 years, SOC 2: per policy)
- [ ] Test breach notification procedures and document response runbooks

### Data Lifecycle Management

- [ ] Define retention periods for each dataset based on regulatory and business requirements
- [ ] Implement automated retention enforcement (deletion or anonymisation pipelines)
- [ ] Document the process for handling data subject access requests (DSARs)
- [ ] Document and test the right-to-be-forgotten workflow end to end
- [ ] Ensure non-production environments contain no real PII (synthetic data or masked copies only)

### Pipeline Design

- [ ] Ensure pipeline is idempotent and produces consistent, auditable results
- [ ] Implement data quality checks that validate sensitive field handling (no accidental PII leakage)
- [ ] Apply network segmentation to isolate sensitive data processing from general workloads
- [ ] Version control all pipeline code and configuration with approval-gated deployments
- [ ] Document data flow diagrams showing where sensitive data enters, is processed, and is stored

### Ongoing Compliance

- [ ] Schedule periodic compliance reviews (quarterly for SOC 2, annually for PCI-DSS, ongoing for GDPR)
- [ ] Run automated PII scanning tools against all environments on a defined schedule
- [ ] Maintain an up-to-date data processing register (GDPR Article 30)
- [ ] Review and update data classification labels as data usage evolves
- [ ] Conduct annual security awareness training for all team members with data access

---

*This note provides a framework-level overview. For organisation-specific policies, consult your Data Protection Officer, Legal team, and Information Security team. Regulatory requirements evolve -- verify current obligations against the latest published standards.*
