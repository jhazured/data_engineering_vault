# Terraform for Data Infrastructure

Infrastructure as Code (IaC) for provisioning and managing cloud data platform resources declaratively.

## Core Concepts

### HCL (HashiCorp Configuration Language)

```hcl
# Provider — which cloud/service to manage
provider "snowflake" {
  account  = var.snowflake_account
  username = var.snowflake_user
  password = var.snowflake_password
  role     = "SYSADMIN"
}

# Resource — an infrastructure object to create/manage
resource "snowflake_database" "analytics" {
  name    = "PROD_T3_INTEGRATION"
  comment = "Integration layer — dimensions and facts"
}

# Variable — parameterised input
variable "snowflake_account" {
  type        = string
  description = "Snowflake account identifier"
  sensitive   = false
}

# Output — expose values after apply
output "database_name" {
  value = snowflake_database.analytics.name
}
```

### State

Terraform tracks what it manages in a **state file** (`terraform.tfstate`):

- Maps HCL resources to real infrastructure objects
- Enables plan/diff — "what will change?"
- **Remote state** (recommended): Store in S3, GCS, Azure Blob, or Terraform Cloud — never commit to git
- **State locking**: Prevents concurrent modifications (DynamoDB for S3 backend, native for Terraform Cloud)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "data-platform/terraform.tfstate"
    region         = "ap-southeast-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

## Workflow

```bash
terraform init      # Download providers, initialise backend
terraform plan      # Preview changes (dry run)
terraform apply     # Execute changes (with confirmation)
terraform destroy   # Tear down all managed resources
```

**Always run `plan` before `apply`** — review changes before they execute.

## Data Platform Resources

### Snowflake

```hcl
provider "snowflake" {
  account = var.sf_account
  role    = "SYSADMIN"
}

# Databases (one per tier)
resource "snowflake_database" "t1" {
  name = "${var.environment}_T1_TRANSIENT_STAGING"
}

resource "snowflake_database" "t2" {
  name = "${var.environment}_T2_PERSISTENT_STAGING"
}

# Warehouses
resource "snowflake_warehouse" "transform" {
  name              = "TRN_${var.environment}_CENTRAL_WH"
  warehouse_size    = "X-SMALL"
  auto_suspend      = var.environment == "PROD" ? 120 : 60
  auto_resume       = true
  initially_suspended = true
}

# Roles
resource "snowflake_role" "engineer" {
  name = "ENGINEER"
}

resource "snowflake_role_grants" "engineer_wh" {
  role_name = snowflake_role.engineer.name
  roles     = ["SYSADMIN"]
}

# Grants
resource "snowflake_database_grant" "engineer_t2" {
  database_name = snowflake_database.t2.name
  privilege     = "ALL PRIVILEGES"
  roles         = [snowflake_role.engineer.name]
}
```

### AWS (S3 + Glue + Redshift)

```hcl
provider "aws" {
  region = "ap-southeast-2"
}

# S3 data lake bucket
resource "aws_s3_bucket" "data_lake" {
  bucket = "company-data-lake-${var.environment}"
}

resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration { status = "Enabled" }
}

# Glue catalog database
resource "aws_glue_catalog_database" "analytics" {
  name = "analytics_${var.environment}"
}

# IAM role for Glue jobs
resource "aws_iam_role" "glue_role" {
  name = "glue-etl-role-${var.environment}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "glue.amazonaws.com" }
    }]
  })
}
```

### GCP (BigQuery + GCS)

```hcl
provider "google" {
  project = var.project_id
  region  = "australia-southeast1"
}

resource "google_bigquery_dataset" "analytics" {
  dataset_id = "analytics_${var.environment}"
  location   = "australia-southeast1"
}

resource "google_storage_bucket" "data_lake" {
  name     = "${var.project_id}-data-lake"
  location = "AUSTRALIA-SOUTHEAST1"
  versioning { enabled = true }
}
```

## Environment Management

### Workspaces

Separate state per environment using workspaces:

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select dev
terraform apply -var="environment=DEV"
```

### Variable Files

```hcl
# environments/dev.tfvars
environment       = "DEV"
warehouse_size    = "X-SMALL"
auto_suspend_secs = 60

# environments/prod.tfvars
environment       = "PROD"
warehouse_size    = "SMALL"
auto_suspend_secs = 120
```

```bash
terraform apply -var-file="environments/dev.tfvars"
```

## Modules

Reusable, composable infrastructure components:

```hcl
# modules/snowflake-tier/main.tf
variable "environment" { type = string }
variable "tier_name"   { type = string }
variable "schema_name" { type = string }

resource "snowflake_database" "db" {
  name = "${var.environment}_${var.tier_name}"
}

resource "snowflake_schema" "schema" {
  database = snowflake_database.db.name
  name     = var.schema_name
}

# Root main.tf — call the module for each tier
module "t1" {
  source      = "./modules/snowflake-tier"
  environment = var.environment
  tier_name   = "T1_TRANSIENT_STAGING"
  schema_name = "LOGISTICS"
}

module "t2" {
  source      = "./modules/snowflake-tier"
  environment = var.environment
  tier_name   = "T2_PERSISTENT_STAGING"
  schema_name = "LOGISTICS"
}
```

## Best Practices

| Practice | Why |
|----------|-----|
| **Remote state with locking** | Prevents concurrent modifications, enables team collaboration |
| **Never commit `.tfstate`** | Contains sensitive data (passwords, keys) |
| **Use variables for everything environment-specific** | Same code, different vars per environment |
| **Run `plan` before `apply`** | Review changes before executing |
| **Use modules for repeated patterns** | DRY — databases, warehouses, roles follow the same pattern |
| **Pin provider versions** | Prevents breaking changes: `version = "~> 0.87"` |
| **Tag all resources** | Cost attribution, ownership tracking |
| **Use `prevent_destroy`** | Protect production databases from accidental deletion |

```hcl
resource "snowflake_database" "prod_t3" {
  name = "PROD_T3_INTEGRATION"
  lifecycle {
    prevent_destroy = true  # terraform destroy will fail on this resource
  }
}
```

## Terraform vs SQL Scripts

| Aspect | Terraform | SQL Scripts (deploy_all.sh) |
|--------|-----------|---------------------------|
| State tracking | Automatic (tfstate) | Manual (.deployment_status) |
| Drift detection | `terraform plan` shows diff | Must query SHOW DATABASES manually |
| Idempotency | Built-in | Must use IF NOT EXISTS |
| Rollback | `terraform destroy` or revert | Manual DROP statements |
| Multi-cloud | Single tool, multiple providers | Cloud-specific SQL |
| Learning curve | Higher | Lower (just SQL) |
| Best for | Multi-cloud, team collaboration, complex infra | Single-cloud, small teams, quick setup |

---

## GCP Module Design Patterns

Every module follows `main.tf` / `variables.tf` / `outputs.tf`. The root module injects shared config via `locals` and passes `labels` to every child module:

```hcl
locals {
  common_labels = {
    project = var.project_id, environment = var.environment,
    managed_by = "terraform", purpose = "data-migration"
  }
  network_name = "${var.environment}-datamigration-vpc"
  bucket_name  = "${var.project_id}-${var.environment}-datamigration-bucket"
}
module "network" {
  source = "./modules/network"
  project_id = var.project_id
  network_name = local.network_name
  labels = local.common_labels
}
```

**Conditional creation** -- toggle resources with `count` on booleans or null checks (`count = var.storage_bucket != null ? 1 : 0`). **Dynamic blocks** -- iterate over variable-length inputs (the compute module uses this for `attached_disk`, the containers module for Docker build args):

---

## GCP Networking Module

Custom VPC (`auto_create_subnetworks = false`) with secondary IP ranges for GKE, private Google access, and three firewall rules scoped by `target_tags`:

```hcl
resource "google_compute_subnetwork" "subnet" {
  name          = var.subnet_name
  ip_cidr_range = var.cidr_range
  network       = google_compute_network.network.id
  private_ip_google_access = var.enable_private_google_access
  secondary_ip_range { range_name = "pods",     ip_cidr_range = "10.1.0.0/16" }
  secondary_ip_range { range_name = "services", ip_cidr_range = "10.2.0.0/20" }
}
```

Firewall rules: SSH (`target_tags = ["ssh-allowed"]`), internal (all TCP/UDP/ICMP within subnet CIDR), HTTP/HTTPS (`target_tags = ["web-server"]`, ports 80/443/8080/8443). Cloud Router + NAT gateway give private instances outbound internet (`nat_ip_allocate_option = "AUTO_ONLY"`, `source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"`) with error-only logging. Static NAT IP conditionally created via `count`.

---

## Compute and VM Management

Shielded VMs (secure boot, vTPM, integrity monitoring) harden instances at firmware level. Preemptible scheduling cuts cost up to 80% for non-prod workloads:

```hcl
shielded_instance_config {
  enable_secure_boot = true, enable_vtpm = true, enable_integrity_monitoring = true
}
scheduling {
  preemptible = var.preemptible
  automatic_restart = var.automatic_restart
}
```

Startup scripts use `templatefile()` to inject runtime config (project ID, bucket, registry URL) into a bash script that installs Docker, gcloud CLI, and Python dependencies:

```hcl
startup_script = templatefile("${path.module}/scripts/startup.sh", {
  project_id  = var.project_id
  bucket_name = module.storage.bucket_name
  registry_url = module.artifact_registry.repository_url
})
```

TCP health checks are conditionally created. Additional disks use `for_each` with a dynamic `attached_disk` block so disk count is fully variable-driven.

---

## Storage Module

### GCS with Lifecycle Policies

Two lifecycle rules handle automatic cleanup: delete objects after N days, and transition older objects to a cheaper storage class:

```hcl
lifecycle_rule {
  condition { age = var.lifecycle_delete_age }
  action    { type = "Delete" }
}
lifecycle_rule {
  condition { age = var.lifecycle_transition_age }
  action {
    type          = "SetStorageClass"
    storage_class = var.lifecycle_storage_class
  }
}
```

### Versioning, Encryption, and Access Control

Buckets use KMS encryption, uniform bucket-level access (no per-object ACLs), and enforced public access prevention:

```hcl
versioning { enabled = var.bucket_versioning }
encryption { default_kms_key_name = var.kms_key_name }
uniform_bucket_level_access = true
public_access_prevention    = "enforced"
```

### Pub/Sub Notifications for Event-Driven Workflows

When `enable_notifications = true`, the module creates a Pub/Sub topic and binds the GCS service agent as publisher, enabling downstream triggers on `OBJECT_FINALIZE`, `OBJECT_DELETE`, and `OBJECT_METADATA_UPDATE` events.

---

## IAM and Service Accounts

Each role is gated by a boolean flag so only needed permissions are granted. The same `count` pattern covers artifact registry, storage, secret manager, compute, logging, monitoring, and BigQuery:

```hcl
resource "google_project_iam_member" "secret_manager_accessor" {
  count   = var.enable_secret_manager ? 1 : 0
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.service_account.email}"
}
```

Custom roles use `for_each = toset(var.custom_roles)`. **Workload Identity** binds K8s service accounts to GCP service accounts (`roles/iam.workloadIdentityUser`) so pods authenticate without exported keys. **Service account keys** are off by default -- only create for external systems that cannot use Workload Identity or metadata auth.

---

## Secrets Management

### Secret Manager with Automatic Replication

Secrets are created from a map variable using `for_each`, with automatic replication across regions:

```hcl
resource "google_secret_manager_secret" "secrets" {
  for_each  = var.secrets
  secret_id = each.key
  replication { automatic = true }
  labels = merge(var.labels, { secret_name = each.key })
}
```

Version data is stored via `google_secret_manager_secret_version`. The `lifecycle { ignore_changes = [secret_data] }` block prevents Terraform from overwriting secrets that are rotated externally.

### IAM Access Bindings

Accessor and admin roles are bound per-secret so access can be scoped individually rather than project-wide. Pub/Sub notifications for rotation alerting follow the same conditional pattern as the storage module.

---

## Monitoring Module

Dashboards use `dashboard_json` with a mosaic layout -- tiles for CPU, memory, disk, and network traffic, each querying `compute.googleapis.com` metrics with 60s alignment. Four alert policies are conditionally created (high CPU, high memory, high disk, instance down) using `condition_threshold` with 300s duration windows (60s for instance-down), auto-closing after 1800s. Notification channels wire to email when `notification_email` is set. HTTP uptime checks poll the health endpoint at 60s intervals with 10s timeout.

---

## Multi-Environment Management

Each environment gets its own directory under `envs/` with a `main.tf` referencing shared modules via relative paths (`source = "../../modules/network"`). Per-environment tfvars control sizing and cost:

```hcl
# envs/dev/terraform.tfvars.example
environment  = "dev"
machine_type = "e2-small"
budget_amount = 100
enable_backup = false
```

State is isolated via the `prefix` field in the GCS backend (`bucket = "gcp-datamigration-terraform-state"`, `prefix = "terraform/state"`). Directory-based isolation is preferred over workspaces when environments differ in resource composition, since each env can have its own `main.tf` with different module calls.

---

## Variable Validation

Terraform `validation` blocks catch bad input at `plan` time. Five validation strategies used across the project:

```hcl
# Regex -- project ID format
variable "project_id" {
  type = string
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{4,28}[a-z0-9]$", var.project_id))
    error_message = "Must be 6-30 chars, start with letter, lowercase/numbers/hyphens only."
  }
}
# Enum -- contains() for fixed sets
variable "environment" {
  validation { condition = contains(["dev", "test", "uat", "prod"], var.environment) }
}
# CIDR -- cidrhost() validates IPv4 blocks
variable "cidr_range" {
  validation { condition = can(cidrhost(var.cidr_range, 0)) }
}
# Email -- regex with optional empty
variable "notification_email" {
  validation {
    condition = var.notification_email == "" || can(regex(
      "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.notification_email))
  }
}
# List -- alltrue() with comprehension for budget thresholds
variable "budget_alert_thresholds" {
  validation {
    condition = alltrue([for t in var.budget_alert_thresholds : t > 0 && t <= 100])
  }
}
```

The `region` variable uses `contains()` with an explicit allowlist of 20+ valid GCP regions. `storage_class` validates against `["STANDARD", "NEARLINE", "COLDLINE", "ARCHIVE"]`.
