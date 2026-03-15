
#pyspark #security #governance #encryption #masking #pii #audit #unity-catalog #ranger

## Overview

Patterns for securing data in PySpark workloads: column-level encryption, data masking, row-level filtering, audit logging, PII detection, and integration with governance platforms. These techniques are essential for regulatory compliance (GDPR, HIPAA, PCI-DSS) and enterprise data governance. See also [[PySpark Core Concepts]], [[PySpark Production Engineering]], and [[Databricks & Delta Lake]].

---

## Column-Level Encryption

### AES Encryption With PySpark Functions

```python
from pyspark.sql import functions as F

# Spark 3.3+ provides built-in AES functions
# AES-256 in GCM mode (recommended)

encryption_key = "0123456789abcdef0123456789abcdef"  # 32 hex chars = 128-bit key
# In production, load from secrets manager — never hardcode

# Encrypt sensitive columns
encrypted_df = (
    df
    .withColumn("ssn_encrypted", F.base64(F.aes_encrypt(F.col("ssn"), F.lit(encryption_key))))
    .withColumn("email_encrypted", F.base64(F.aes_encrypt(F.col("email"), F.lit(encryption_key))))
    .drop("ssn", "email")
)

# Decrypt when authorised
decrypted_df = (
    encrypted_df
    .withColumn("ssn", F.aes_decrypt(F.unbase64(F.col("ssn_encrypted")), F.lit(encryption_key)).cast("string"))
    .withColumn("email", F.aes_decrypt(F.unbase64(F.col("email_encrypted")), F.lit(encryption_key)).cast("string"))
)
```

### Key Management Pattern

```python
def get_encryption_key(secret_name, provider="databricks"):
    """Retrieve encryption key from a secrets manager."""
    if provider == "databricks":
        return dbutils.secrets.get(scope="encryption", key=secret_name)
    elif provider == "aws":
        import boto3
        client = boto3.client("secretsmanager")
        response = client.get_secret_value(SecretId=secret_name)
        return response["SecretString"]
    elif provider == "spark_conf":
        return spark.conf.get(f"spark.app.encryption.{secret_name}")
    else:
        raise ValueError(f"Unknown provider: {provider}")

key = get_encryption_key("column_encryption_key")
```

### Envelope Encryption Pattern

```python
import os
import base64
from cryptography.fernet import Fernet

def envelope_encrypt_column(df, column, master_key_name):
    """Envelope encryption: encrypt data key with master key, encrypt data with data key."""
    # Generate a unique data encryption key (DEK) per batch
    dek = Fernet.generate_key()
    fernet = Fernet(dek)

    # Encrypt the DEK with the master key (KEK) — done outside Spark
    master_key = get_encryption_key(master_key_name)
    master_fernet = Fernet(master_key)
    encrypted_dek = master_fernet.encrypt(dek).decode("utf-8")

    # Encrypt data column with DEK using a UDF
    @F.udf("string")
    def encrypt_value(value):
        if value is None:
            return None
        return fernet.encrypt(value.encode("utf-8")).decode("utf-8")

    return (
        df.withColumn(f"{column}_encrypted", encrypt_value(F.col(column)))
          .withColumn("_encrypted_dek", F.lit(encrypted_dek))
          .drop(column)
    )
```

---

## Data Masking

### Hash-Based Masking

```python
from pyspark.sql import functions as F

def hash_mask(df, columns, salt="vault_salt_2025"):
    """One-way hash masking — irreversible, consistent across runs."""
    for col in columns:
        df = df.withColumn(col, F.sha2(F.concat(F.col(col).cast("string"), F.lit(salt)), 256))
    return df

masked_df = hash_mask(df, ["email", "phone_number"])
```

### Redaction Masking

```python
@F.udf("string")
def redact_email(email):
    """Redact email: j***@example.com"""
    if email is None:
        return None
    parts = email.split("@")
    if len(parts) != 2:
        return "***@***"
    local = parts[0]
    return f"{local[0]}***@{parts[1]}"

@F.udf("string")
def redact_phone(phone):
    """Show only last 4 digits: ***-***-1234"""
    if phone is None:
        return None
    digits = "".join(c for c in phone if c.isdigit())
    if len(digits) < 4:
        return "****"
    return f"***-***-{digits[-4:]}"

@F.udf("string")
def redact_card(card_number):
    """PCI-compliant masking: show only last 4 digits."""
    if card_number is None:
        return None
    digits = "".join(c for c in card_number if c.isdigit())
    return f"****-****-****-{digits[-4:]}"

masked_df = (
    df
    .withColumn("email", redact_email("email"))
    .withColumn("phone", redact_phone("phone"))
    .withColumn("card_number", redact_card("card_number"))
)
```

### Generalisation Masking

```python
from pyspark.sql import functions as F

def generalise_age(df, col="age"):
    """Generalise age to bands to prevent re-identification."""
    return df.withColumn(col, (F.floor(F.col(col) / 10) * 10).cast("string"))

def generalise_postcode(df, col="postcode"):
    """Retain only the outward code (first half) of a UK postcode."""
    return df.withColumn(col, F.split(F.col(col), " ")[0])

def generalise_date(df, col="date_of_birth"):
    """Reduce date precision to year-month only."""
    return df.withColumn(col, F.date_format(F.col(col), "yyyy-MM"))
```

### Dynamic Masking Based On Role

```python
def apply_role_based_masking(df, user_role, masking_rules):
    """Apply different masking levels based on user role.

    masking_rules: dict of {role: {column: masking_function}}
    """
    rules = masking_rules.get(user_role, masking_rules.get("default", {}))
    for column, mask_fn in rules.items():
        if column in df.columns:
            df = mask_fn(df, column)
    return df

masking_rules = {
    "analyst": {
        "email": lambda df, c: df.withColumn(c, redact_email(F.col(c))),
        "ssn": lambda df, c: df.withColumn(c, F.lit("***-**-****")),
        "salary": lambda df, c: df.withColumn(c, (F.floor(F.col(c) / 10000) * 10000)),
    },
    "data_engineer": {
        "ssn": lambda df, c: df.withColumn(c, F.sha2(F.col(c), 256)),
    },
    "admin": {},  # no masking
    "default": {
        "email": lambda df, c: df.withColumn(c, F.lit("***")),
        "ssn": lambda df, c: df.withColumn(c, F.lit("***")),
        "salary": lambda df, c: df.drop(c),
    },
}
```

---

## Row-Level Filtering

### View-Based Row-Level Security

```python
def create_filtered_view(df, user_context, filter_rules):
    """Apply row-level security based on user attributes."""
    filtered = df
    for rule in filter_rules:
        if rule["applies_to"](user_context):
            filtered = filtered.filter(rule["predicate"])
    return filtered

filter_rules = [
    {
        "applies_to": lambda ctx: ctx["role"] == "regional_manager",
        "predicate": F.col("region") == F.lit("EMEA"),
    },
    {
        "applies_to": lambda ctx: ctx["department"] is not None,
        "predicate": F.col("department") == F.lit("Finance"),
    },
]

# Usage
user_ctx = {"role": "regional_manager", "department": "Finance", "user_id": "u123"}
secured_df = create_filtered_view(df, user_ctx, filter_rules)
```

### Policy-Driven Filtering

```python
def apply_data_policy(df, policy_table_path, user_id):
    """Join against a policy table to enforce row-level access."""
    policies = spark.read.parquet(policy_table_path).filter(F.col("user_id") == user_id)

    # Policy table schema: user_id, allowed_regions, allowed_departments
    allowed = policies.select(
        F.explode("allowed_regions").alias("allowed_region")
    ).distinct()

    return df.join(
        allowed,
        df["region"] == allowed["allowed_region"],
        "inner"
    ).drop("allowed_region")
```

---

## Audit Logging

### Query-Level Audit Trail

```python
from datetime import datetime
from pyspark.sql import functions as F

class AuditLogger:
    """Log data access events for compliance and forensics."""

    def __init__(self, spark, audit_table_path):
        self.spark = spark
        self.audit_table_path = audit_table_path

    def log_access(self, user_id, dataset, operation, columns_accessed,
                   row_count, filters_applied=None):
        """Write an audit record for a data access event."""
        record = {
            "event_id": str(uuid.uuid4()),
            "timestamp": datetime.utcnow().isoformat(),
            "user_id": user_id,
            "dataset": dataset,
            "operation": operation,
            "columns_accessed": columns_accessed,
            "row_count": row_count,
            "filters_applied": str(filters_applied) if filters_applied else None,
            "spark_app_id": self.spark.sparkContext.applicationId,
        }
        audit_df = self.spark.createDataFrame([record])
        audit_df.write.mode("append").parquet(self.audit_table_path)

    def log_read(self, user_id, dataset, df):
        """Convenience wrapper for read operations."""
        columns = df.columns
        row_count = df.count()
        self.log_access(user_id, dataset, "READ", columns, row_count)

import uuid

audit = AuditLogger(spark, "/mnt/audit/access_logs/")

# Log before returning data
result_df = spark.read.parquet("/mnt/data/customers/")
audit.log_read("user_abc", "customers", result_df)
```

### Delta Lake Audit With History

```python
# Delta Lake provides built-in audit via table history
# See [[Databricks & Delta Lake]] for details

history_df = spark.sql("DESCRIBE HISTORY customers_table")
history_df.select(
    "version", "timestamp", "userId", "operation",
    "operationParameters", "operationMetrics"
).show(truncate=False)
```

---

## PII Detection Patterns

### Regex-Based PII Scanner

```python
import re
from pyspark.sql import functions as F
from pyspark.sql.types import ArrayType, StringType

PII_PATTERNS = {
    "email": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
    "uk_nino": r"[A-Z]{2}\d{6}[A-D]",
    "us_ssn": r"\d{3}-\d{2}-\d{4}",
    "credit_card": r"\b(?:\d[ -]*?){13,16}\b",
    "uk_postcode": r"[A-Z]{1,2}\d[A-Z\d]?\s*\d[A-Z]{2}",
    "phone_intl": r"\+\d{1,3}[\s-]?\d{4,14}",
}

def scan_column_for_pii(df, column):
    """Scan a single column for PII patterns, return matched types."""
    @F.udf(ArrayType(StringType()))
    def detect_pii(value):
        if value is None:
            return []
        found = []
        for pii_type, pattern in PII_PATTERNS.items():
            if re.search(pattern, str(value)):
                found.append(pii_type)
        return found

    return df.withColumn(f"{column}_pii_types", detect_pii(F.col(column)))

def scan_dataframe_for_pii(df, sample_fraction=0.01):
    """Scan all string columns in a DataFrame for PII."""
    string_cols = [f.name for f in df.schema.fields if str(f.dataType) == "StringType()"]
    sample_df = df.sample(fraction=sample_fraction)
    results = {}

    for col in string_cols:
        scanned = scan_column_for_pii(sample_df, col)
        pii_counts = (
            scanned
            .select(F.explode(f"{col}_pii_types").alias("pii_type"))
            .groupBy("pii_type")
            .count()
            .collect()
        )
        if pii_counts:
            results[col] = {row["pii_type"]: row["count"] for row in pii_counts}

    return results

# Usage
pii_report = scan_dataframe_for_pii(df, sample_fraction=0.05)
for col, types in pii_report.items():
    print(f"Column '{col}': {types}")
```

---

## Governance Platform Integration

### Unity Catalog (Databricks)

```python
# Unity Catalog provides three-level namespace: catalog.schema.table
# Access control is managed via SQL GRANT statements

# Grant table-level access
spark.sql("GRANT SELECT ON TABLE main.finance.transactions TO `data_analysts`")

# Grant column-level access (column masking)
spark.sql("""
    ALTER TABLE main.finance.transactions
    ALTER COLUMN ssn SET MASK mask_ssn
""")

# Row-level filtering via row filter functions
spark.sql("""
    ALTER TABLE main.finance.transactions
    SET ROW FILTER region_filter ON (region)
""")

# Tag columns for governance
spark.sql("""
    ALTER TABLE main.finance.transactions
    ALTER COLUMN email SET TAGS ('pii' = 'true', 'sensitivity' = 'high')
""")

# Query lineage (Unity Catalog tracks automatically)
# View in Catalog Explorer UI or via REST API
```

### Apache Ranger Integration

```python
# Ranger policies are typically configured via the Ranger Admin UI or REST API
# PySpark reads/writes are enforced by the Ranger plugin on the Spark/Hive server

# Example: Ranger policy JSON (applied via REST API)
ranger_policy = {
    "policyName": "finance_table_access",
    "resourceName": "finance.transactions",
    "policyItems": [
        {
            "users": ["data_analyst_group"],
            "accesses": [{"type": "select", "isAllowed": True}],
            "conditions": [
                {"type": "row-filter", "values": ["region = 'EMEA'"]}
            ]
        }
    ],
    "columnMaskPolicyItems": [
        {
            "users": ["data_analyst_group"],
            "dataMaskInfo": {"dataMaskType": "MASK_HASH"},
            "accesses": [{"type": "select", "isAllowed": True}],
            "resources": {"column": {"values": ["ssn", "email"]}}
        }
    ]
}
```

### AWS Lake Formation Integration

```python
# Lake Formation manages permissions at the Glue Catalog level
# PySpark on EMR or Glue respects Lake Formation policies when configured

# Register a data lake location
# (done via AWS CLI or console, not PySpark)
# aws lakeformation register-resource --resource-arn arn:aws:s3:::my-data-lake

# Grant fine-grained access via boto3
import boto3

lf_client = boto3.client("lakeformation")

# Column-level permission
lf_client.grant_permissions(
    Principal={"DataLakePrincipalIdentifier": "arn:aws:iam::123456789012:role/AnalystRole"},
    Resource={
        "TableWithColumns": {
            "DatabaseName": "finance_db",
            "Name": "transactions",
            "ColumnNames": ["transaction_id", "amount", "date"],
            # Excluded columns (ssn, email) are not accessible
        }
    },
    Permissions=["SELECT"]
)

# Data filters for row-level security
lf_client.create_data_cells_filter(
    TableData={
        "DatabaseName": "finance_db",
        "TableName": "transactions",
        "Name": "emea_only_filter",
        "RowFilter": {"FilterExpression": "region = 'EMEA'"},
        "ColumnWildcard": {}
    }
)
```

---

## Secure Credential Management

### Spark Configuration

```python
# Pass secrets via Spark conf (set at cluster launch, not in code)
# spark-submit --conf spark.app.db.password=secret123

password = spark.conf.get("spark.app.db.password")
```

### Databricks Secrets

```python
# Databricks secrets are scoped and access-controlled
# Create scope and secret via Databricks CLI:
#   databricks secrets create-scope --scope my-scope
#   databricks secrets put --scope my-scope --key db-password

password = dbutils.secrets.get(scope="my-scope", key="db-password")

# Secrets are redacted in notebook output and logs
# Display shows [REDACTED] instead of the actual value
```

### AWS Secrets Manager

```python
import boto3
import json

def get_secret(secret_name, region="eu-west-1"):
    """Retrieve a secret from AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

creds = get_secret("prod/database/credentials")
df = (
    spark.read.format("jdbc")
    .option("url", creds["jdbc_url"])
    .option("user", creds["username"])
    .option("password", creds["password"])
    .option("dbtable", "public.transactions")
    .load()
)
```

### Credential Caching Pattern

```python
from functools import lru_cache

@lru_cache(maxsize=32)
def get_cached_secret(secret_name):
    """Cache secrets to avoid repeated API calls within a single job."""
    return get_secret(secret_name)

# Invalidate cache when rotating credentials
get_cached_secret.cache_clear()
```

---

## Network Isolation

### VPC And Private Endpoints

```text
Production Spark clusters should run within a VPC with:

1. Private subnets only — no public IP addresses on worker nodes
2. VPC endpoints for AWS services (S3, Glue, Secrets Manager, KMS)
   - Avoids traffic traversing the public internet
3. Security groups restricting inbound/outbound traffic
   - Workers: allow inter-node communication on Spark ports (7077, 4040, etc.)
   - Driver: allow access from orchestration layer only
4. NAT Gateway only if external egress is required (e.g. fetching packages)
5. PrivateLink for Databricks workspace connectivity
```

### Spark SSL/TLS Configuration

```properties
# Enable encryption in transit between Spark nodes
spark.ssl.enabled                 true
spark.ssl.keyStore                /path/to/keystore.jks
spark.ssl.keyStorePassword        ${KEY_STORE_PASSWORD}
spark.ssl.trustStore              /path/to/truststore.jks
spark.ssl.trustStorePassword      ${TRUST_STORE_PASSWORD}
spark.ssl.protocol                TLSv1.3

# Enable encryption for RPC and block transfer
spark.network.crypto.enabled      true
spark.network.crypto.keyLength    256
spark.authenticate                true
spark.authenticate.secret         ${SPARK_AUTH_SECRET}
```

### Data At Rest Encryption

```python
# S3 server-side encryption (SSE-KMS)
spark.conf.set("spark.hadoop.fs.s3a.server-side-encryption-algorithm", "SSE-KMS")
spark.conf.set("spark.hadoop.fs.s3a.server-side-encryption.key", "arn:aws:kms:eu-west-1:123456789012:key/my-key-id")

# DBFS encryption (Databricks manages this transparently)
# Configure via workspace settings: customer-managed keys (CMK)

# Delta Lake — encrypted storage is handled by the underlying filesystem
# No additional configuration needed at the Spark/Delta level
```

---

## Related Notes

- [[PySpark Core Concepts]] — SparkSession, configuration fundamentals
- [[PySpark Production Engineering]] — production deployment patterns
- [[PySpark Performance Optimization]] — performance considerations for security overhead
- [[Databricks & Delta Lake]] — Unity Catalog deep dive
- [[Databricks Modern Patterns (2025)]] — latest governance features
- [[PySpark Cloud Integration Patterns]] — cloud-specific security configurations
