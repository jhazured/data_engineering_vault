# Azure Data Factory Project Patterns

**Tags:** #azure #adf #data-factory #etl #synapse #data-lake #pipeline

## Overview

Azure Data Factory (ADF) is Microsoft's cloud-based ETL/ELT service for data integration and orchestration. This note covers reusable patterns for setting up ADF projects with ADLS Gen2, Azure SQL, Synapse Analytics, and Key Vault — based on a production e-commerce analytics implementation.

See also: [[Microsoft Fabric & Azure Data Services]] for the broader Azure data platform.

---

## Infrastructure Setup (Azure CLI)

### Resource Group

```bash
$resourceGroup = "rg-analytics"
$location = "East US 2"

az login
az account set --subscription $subscriptionId
az group create --name $resourceGroup --location $location
```

### ADLS Gen2 (Data Lake)

```bash
az storage account create \
  --name $storageAccount \
  --resource-group $resourceGroup \
  --location $location \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true

# Create medallion containers
az storage container create --name "raw" --account-name $storageAccount
az storage container create --name "processed" --account-name $storageAccount
az storage container create --name "curated" --account-name $storageAccount
```

### Azure SQL Database (Source)

```bash
az sql server create \
  --name $sqlServer \
  --resource-group $resourceGroup \
  --location $location \
  --admin-user $sqlAdmin \
  --admin-password $sqlPassword

az sql db create \
  --resource-group $resourceGroup \
  --server $sqlServer \
  --name "source-db" \
  --service-objective Basic

# Allow Azure services through firewall
az sql server firewall-rule create \
  --resource-group $resourceGroup \
  --server $sqlServer \
  --name "AllowAzureServices" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### Synapse Analytics (Data Warehouse)

```bash
az synapse workspace create \
  --name $synapseWorkspace \
  --resource-group $resourceGroup \
  --storage-account $storageAccount \
  --file-system "processed" \
  --sql-admin-login-user $synapseAdmin \
  --sql-admin-login-password $synapsePassword \
  --location $location

# Dedicated SQL pool
az synapse sql pool create \
  --name "dwh_analytics" \
  --workspace-name $synapseWorkspace \
  --resource-group $resourceGroup \
  --performance-level DW100c
```

### Key Vault & Data Factory

```bash
az keyvault create \
  --name $keyVault \
  --resource-group $resourceGroup \
  --location $location

az datafactory create \
  --resource-group $resourceGroup \
  --name $dataFactory \
  --location $location
```

---

## Data Warehouse Schema Pattern

### Dimension Tables (Synapse)

Use `DISTRIBUTION = REPLICATE` for small dimension tables — copies to every compute node for fast joins:

```sql
CREATE TABLE dim_customer (
    customer_key INT IDENTITY(1,1) PRIMARY KEY,
    customer_id NVARCHAR(10) NOT NULL,
    first_name NVARCHAR(50),
    last_name NVARCHAR(50),
    email NVARCHAR(100),
    customer_segment NVARCHAR(20),
    -- SCD Type 2 columns
    effective_date DATE NOT NULL,
    expiry_date DATE,
    is_current BIT DEFAULT 1
)
WITH (DISTRIBUTION = REPLICATE);
```

### Fact Tables (Synapse)

Use `DISTRIBUTION = HASH(key)` for large fact tables — distributes rows across compute nodes:

```sql
CREATE TABLE fact_sales (
    sales_key BIGINT IDENTITY(1,1) PRIMARY KEY,
    date_key INT NOT NULL,
    customer_key INT NOT NULL,
    product_key INT NOT NULL,
    order_id NVARCHAR(20) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    created_date DATETIME2 DEFAULT GETDATE()
)
WITH (DISTRIBUTION = HASH(customer_key));
```

See [[Star Schema Implementation Patterns]] for dimensional modelling best practices.

---

## Linked Services Configuration

| Service | Linked Service Name | Authentication |
|---------|-------------------|----------------|
| Azure SQL Database | `ls_azure_sql_source` | SQL Auth (Key Vault) |
| Blob Storage | `ls_azure_blob_storage` | Account Key or Managed Identity |
| ADLS Gen2 | `ls_azure_data_lake` | **Managed Identity (recommended)** |
| Synapse Analytics | `ls_synapse_dedicated_pool` | SQL Auth (Key Vault) |

**Best Practice:** Always use Managed Identity where possible. Store SQL credentials in Key Vault, never hard-code.

---

## Dataset Patterns

### Parameterised CSV Dataset

Use parameters to make datasets reusable across multiple files:

```json
{
    "name": "ds_blob_csv",
    "properties": {
        "linkedServiceName": {
            "referenceName": "ls_azure_blob_storage",
            "type": "LinkedServiceReference"
        },
        "parameters": {
            "fileName": { "type": "string" }
        },
        "type": "DelimitedText",
        "typeProperties": {
            "location": {
                "type": "AzureBlobStorageLocation",
                "fileName": "@dataset().fileName",
                "container": "raw",
                "folderPath": "sales"
            },
            "columnDelimiter": ",",
            "firstRowAsHeader": true
        }
    }
}
```

---

## Pipeline Patterns

### Master Pipeline Pattern

Orchestrate child pipelines with dependencies and parallel execution:

1. **`pl_master_etl`** — master orchestrator
   - Execute Pipeline: `pl_ingest_data` (parallel per source)
   - Execute Pipeline: `pl_transform_data` (after ingestion)
   - Execute Pipeline: `pl_load_warehouse` (after transformation)
   - Notification activity on success/failure

### File Ingestion Pipeline

For processing multiple files (e.g. monthly CSV drops):

1. **Get Metadata** — list CSV files in blob container
2. **ForEach** — iterate over each file:
   - **Copy Data** — CSV to Data Lake raw zone
   - **Data Flow** — transform (type casting, audit columns, filtering)
   - **Copy Data** — processed data to Synapse

### Data Flow Transformations

Common transformation steps in ADF data flows:

| Step | Purpose |
|------|---------|
| Derived Column | Type conversions, add audit columns (`pipeline_run_id`, `ingestion_timestamp`) |
| Filter | Remove invalid or null records |
| Aggregate | Summarise by dimensions |
| Lookup | Enrich with reference data |
| Sink | Write to ADLS processed or Synapse |

---

## Environment Strategy

| Environment | Purpose | Resource Group |
|-------------|---------|----------------|
| Dev | Development and experimentation | `rg-analytics-dev` |
| Test | Integration testing | `rg-analytics-test` |
| Prod | Production workloads | `rg-analytics-prod` |

Use ARM templates or Bicep for consistent deployment across environments. Configure CI/CD with Azure DevOps.

---

## Security Hardening

- **Managed Identity** for all service-to-service authentication
- **Private Endpoints** for network isolation
- **RBAC** with least-privilege access
- **Key Vault** for all secrets and connection strings
- **Audit logging** enabled on all resources

---

## Cost Optimisation

| Strategy | Impact |
|----------|--------|
| Auto-pause Synapse SQL pools | Saves compute cost during idle hours |
| Right-size DIU settings | Match Data Integration Units to workload |
| Storage tiering | Move cold data to Cool/Archive storage |
| Budget alerts | Set up Azure Cost Management alerts |
| Monitor pipeline runs | Identify and eliminate wasteful reruns |

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Connection failures | Firewall rules or missing permissions | Check Azure SQL firewall, verify Managed Identity access |
| Data flow errors | Column mapping or type mismatches | Review schema mappings, check source data types |
| Pipeline failures | Dependency misconfiguration | Check activity dependencies, validate parameter passing |
| Slow performance | Undersized compute or unpartitioned data | Scale DIU, implement [[Data Ingestion Patterns|partitioning]] |

---

**Related:** [[Microsoft Fabric & Azure Data Services]] | [[Star Schema Implementation Patterns]] | [[Data Ingestion Patterns]] | [[ETL Pipeline Templates & Patterns]]
