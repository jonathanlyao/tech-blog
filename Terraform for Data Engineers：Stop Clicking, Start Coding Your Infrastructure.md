# Terraform for Data Engineers: Stop Clicking, Start Coding Your Infrastructure

> **Every resource you create manually is a resource you'll have to recreate manually. Terraform ends that cycle.**

---

## Table of Contents

1. [What Is Terraform?](#1-what-is-terraform)
2. [Why Data Engineers Should Care](#2-why-data-engineers-should-care)
3. [Core Concepts You Need to Know](#3-core-concepts-you-need-to-know)
4. [The Problem: Manual Infrastructure in Real Projects](#4-the-problem-manual-infrastructure-in-real-projects)
5. [Project 3 — Codifying the BTS Flights Pipeline Infrastructure](#5-project-3--codifying-the-bts-flights-pipeline-infrastructure)
6. [Project 4 — Codifying the Kafka + Spark Streaming Infrastructure](#6-project-4--codifying-the-kafka--spark-streaming-infrastructure)
7. [The Full Terraform Workflow](#7-the-full-terraform-workflow)
8. [Version Control and Reproducibility](#8-version-control-and-reproducibility)
9. [One-Command Teardown and Rebuild](#9-one-command-teardown-and-rebuild)
10. [What Terraform Does Not Replace](#10-what-terraform-does-not-replace)
11. [Summary and Next Steps](#11-summary-and-next-steps)

---

## 1. What Is Terraform?

Terraform is an open-source **Infrastructure as Code (IaC)** tool developed by HashiCorp. It allows you to define cloud infrastructure — servers, storage buckets, databases, networks, permissions — in human-readable configuration files, and then create, update, or destroy that infrastructure through the command line.

Instead of logging into the AWS Console and clicking through menus to create an S3 bucket, you write this:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "olist-data-lake-leeyao-oregon"
}
```

Run `terraform apply`, and Terraform creates the bucket. Run `terraform destroy`, and it's gone. The entire state of your infrastructure lives in code that can be committed to Git, reviewed, versioned, and shared.

Terraform is **cloud-agnostic**: the same workflow applies whether you're provisioning resources on AWS, Snowflake, GCP, Azure, or Kafka. It communicates with each provider through a plugin system called **providers**.

---

## 2. Why Data Engineers Should Care

Data engineering projects involve a surprisingly large number of infrastructure resources. For a single pipeline project, you might manually create:

- S3 buckets (with specific configurations: versioning, lifecycle rules, region)
- Snowflake databases, schemas, warehouses, roles, and users
- IAM roles and policies for cross-service access
- Docker networks and volumes for local development
- Kafka topics and configurations
- Airflow connections and environment variables

Every one of these was created by hand. And that means:

- **It cannot be reproduced** on a new machine or in a new environment without doing it all again
- **It cannot be reviewed** — there is no record of what was created, when, or why
- **It cannot be rolled back** — if a configuration change breaks something, reverting is manual
- **It will be forgotten** — six months later, you won't remember exactly how things were set up

Terraform solves all four problems simultaneously.

---

## 3. Core Concepts You Need to Know

### Provider

A provider is a plugin that lets Terraform communicate with a specific platform. Each provider exposes a set of **resources** you can manage.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.87"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

provider "snowflake" {
  account = var.snowflake_account
  username = var.snowflake_username
  password = var.snowflake_password
  role     = "SYSADMIN"
}
```

### Resource

A resource is a single infrastructure object — an S3 bucket, a Snowflake database, a Kafka topic. Resources are the fundamental building blocks of a Terraform configuration.

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "olist-data-lake-leeyao-oregon"
}
```

### Variable

Variables make configurations reusable and keep secrets out of your code.

```hcl
variable "snowflake_account" {
  description = "Snowflake account identifier"
  type        = string
  sensitive   = true
}
```

### State

Terraform maintains a **state file** (`terraform.tfstate`) that maps your configuration to the real resources that exist in the cloud. This is how Terraform knows what already exists and what needs to be created, updated, or deleted.

### Plan and Apply

```bash
terraform plan    # shows what Terraform will do — no changes made
terraform apply   # executes the changes after confirmation
terraform destroy # removes all resources defined in the configuration
```

---

## 4. The Problem: Manual Infrastructure in Real Projects

Let me be specific about what "manual infrastructure" actually looks like in practice, using my own project experience.

### Project 3 — US DOT BTS Flights Pipeline

To build this pipeline, I manually:

1. Logged into the AWS Console → S3 → created `olist-data-lake-leeyao-oregon` in `us-west-2`
2. Configured bucket versioning manually in the console
3. Logged into Snowflake → created database `BTS_FLIGHTS_DB`, schema `RAW`, warehouse `COMPUTE_WH`
4. Created Snowflake roles and granted privileges through a series of SQL statements run in the worksheet
5. Created an IAM user with S3 read access and manually downloaded the access key

**The problem**: None of this is documented in code. If I need to rebuild this environment — for a job interview demo, a new team member, or after accidentally deleting a resource — I have to remember every step and redo it manually.

### Project 4 — CMS Medicare Streaming Pipeline

On top of the above, this project adds:

1. Docker Compose file for Kafka + Zookeeper (partially codified, but network and volume setup is implicit)
2. Additional Snowflake objects for the streaming schema
3. Spark configuration that assumes specific Snowflake warehouse sizes

The infrastructure spans three platforms (AWS, Snowflake, Docker) and was assembled piece by piece without a single source of truth.

---

## 5. Project 3 — Codifying the BTS Flights Pipeline Infrastructure

Here is how the entire Project 3 infrastructure would be defined in Terraform.

### Directory Structure

```
project3-terraform/
├── main.tf           # provider configuration
├── variables.tf      # input variable declarations
├── terraform.tfvars  # actual variable values (never commit this to Git)
├── s3.tf             # AWS S3 resources
├── iam.tf            # AWS IAM roles and policies
├── snowflake.tf      # Snowflake resources
└── outputs.tf        # output values (bucket name, warehouse name, etc.)
```

### main.tf — Provider Configuration

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.87"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

provider "snowflake" {
  account  = var.snowflake_account
  username = var.snowflake_username
  password = var.snowflake_password
  role     = "SYSADMIN"
}
```

### variables.tf

```hcl
variable "aws_region" {
  description = "AWS region for all resources"
  type        = string
  default     = "us-west-2"
}

variable "s3_bucket_name" {
  description = "Name of the S3 data lake bucket"
  type        = string
  default     = "olist-data-lake-leeyao-oregon"
}

variable "snowflake_account" {
  description = "Snowflake account identifier"
  type        = string
  sensitive   = true
}

variable "snowflake_username" {
  description = "Snowflake login username"
  type        = string
  sensitive   = true
}

variable "snowflake_password" {
  description = "Snowflake login password"
  type        = string
  sensitive   = true
}
```

### s3.tf — S3 Bucket

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = var.s3_bucket_name

  tags = {
    Project     = "BTS-Flights-Pipeline"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

# Enable versioning — previously configured manually in the console
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Block all public access — security baseline
resource "aws_s3_bucket_public_access_block" "data_lake_public_access" {
  bucket = aws_s3_bucket.data_lake.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle rule: move raw data to cheaper storage after 90 days
resource "aws_s3_bucket_lifecycle_configuration" "data_lake_lifecycle" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "archive-raw-data"
    status = "Enabled"

    filter {
      prefix = "bts-flights/raw/"
    }

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
  }
}
```

### iam.tf — IAM Role for Pipeline Access

```hcl
# IAM policy: allow read/write access to the data lake bucket
resource "aws_iam_policy" "data_lake_access" {
  name        = "bts-flights-data-lake-access"
  description = "Read/write access to the BTS Flights data lake S3 bucket"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.data_lake.arn,
          "${aws_s3_bucket.data_lake.arn}/*"
        ]
      }
    ]
  })
}

# IAM user for pipeline service account
resource "aws_iam_user" "pipeline_user" {
  name = "bts-flights-pipeline-user"

  tags = {
    Project   = "BTS-Flights-Pipeline"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_user_policy_attachment" "pipeline_user_policy" {
  user       = aws_iam_user.pipeline_user.name
  policy_arn = aws_iam_policy.data_lake_access.arn
}
```

### snowflake.tf — Snowflake Resources

```hcl
# Database
resource "snowflake_database" "bts_flights" {
  name    = "BTS_FLIGHTS_DB"
  comment = "Database for BTS Flights pipeline — Project 3"
}

# Schemas — four-layer dbt model structure
resource "snowflake_schema" "raw" {
  database = snowflake_database.bts_flights.name
  name     = "RAW"
  comment  = "Raw ingestion layer — direct COPY INTO from S3"
}

resource "snowflake_schema" "staging" {
  database = snowflake_database.bts_flights.name
  name     = "STAGING"
  comment  = "dbt staging layer — type casting, renaming"
}

resource "snowflake_schema" "intermediate" {
  database = snowflake_database.bts_flights.name
  name     = "INTERMEDIATE"
  comment  = "dbt intermediate layer — business logic"
}

resource "snowflake_schema" "marts" {
  database = snowflake_database.bts_flights.name
  name     = "MARTS"
  comment  = "dbt marts layer — final consumable models"
}

# Virtual Warehouse
resource "snowflake_warehouse" "compute" {
  name           = "COMPUTE_WH"
  warehouse_size = "X-SMALL"
  auto_suspend   = 60
  auto_resume    = true
  comment        = "General compute warehouse — auto-suspends after 60s"
}

# Role for dbt and pipeline access
resource "snowflake_role" "pipeline_role" {
  name    = "BTS_PIPELINE_ROLE"
  comment = "Role used by dbt and ingestion scripts"
}

# Grant privileges to the pipeline role
resource "snowflake_grant_privileges_to_role" "pipeline_database_access" {
  privileges = ["USAGE"]
  role_name  = snowflake_role.pipeline_role.name

  on_account_object {
    object_type = "DATABASE"
    object_name = snowflake_database.bts_flights.name
  }
}

resource "snowflake_grant_privileges_to_role" "pipeline_warehouse_access" {
  privileges = ["USAGE", "OPERATE"]
  role_name  = snowflake_role.pipeline_role.name

  on_account_object {
    object_type = "WAREHOUSE"
    object_name = snowflake_warehouse.compute.name
  }
}
```

### outputs.tf

```hcl
output "s3_bucket_name" {
  description = "Name of the data lake S3 bucket"
  value       = aws_s3_bucket.data_lake.bucket
}

output "s3_bucket_arn" {
  description = "ARN of the data lake S3 bucket"
  value       = aws_s3_bucket.data_lake.arn
}

output "snowflake_database" {
  description = "Snowflake database name"
  value       = snowflake_database.bts_flights.name
}

output "snowflake_warehouse" {
  description = "Snowflake warehouse name"
  value       = snowflake_warehouse.compute.name
}
```

---

## 6. Project 4 — Codifying the Kafka + Spark Streaming Infrastructure

Project 4 adds a streaming layer: Kafka and Zookeeper run locally via Docker, and the Snowflake layer expands to support streaming data.

### Directory Structure

```
project4-terraform/
├── main.tf
├── variables.tf
├── terraform.tfvars
├── s3.tf             # reuses project-level prefix on shared bucket
├── snowflake.tf      # streaming-specific Snowflake objects
├── docker.tf         # Docker network and volume management
└── outputs.tf
```

### docker.tf — Docker Network and Volumes

Terraform has a Docker provider that can manage local Docker resources — networks, volumes, containers, and images. This replaces implicit Docker Compose network setup with explicit, versioned definitions.

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

# Dedicated Docker network for the streaming pipeline
resource "docker_network" "kafka_network" {
  name   = "kafka-streaming-network"
  driver = "bridge"

  labels {
    label = "project"
    value = "cms-medicare-streaming"
  }

  labels {
    label = "managed_by"
    value = "terraform"
  }
}

# Persistent volume for Kafka data
resource "docker_volume" "kafka_data" {
  name = "kafka-data-volume"

  labels {
    label = "project"
    value = "cms-medicare-streaming"
  }
}

# Persistent volume for Zookeeper data
resource "docker_volume" "zookeeper_data" {
  name = "zookeeper-data-volume"

  labels {
    label = "project"
    value = "cms-medicare-streaming"
  }
}
```

### snowflake.tf — Streaming Snowflake Objects

```hcl
# Database for CMS Medicare streaming project
resource "snowflake_database" "cms_medicare" {
  name    = "CMS_MEDICARE_DB"
  comment = "Database for CMS Medicare Streaming Pipeline — Project 4"
}

# Streaming-specific schemas
resource "snowflake_schema" "streaming_raw" {
  database = snowflake_database.cms_medicare.name
  name     = "STREAMING_RAW"
  comment  = "Raw streaming data from Kafka via Spark Structured Streaming"
}

resource "snowflake_schema" "streaming_staging" {
  database = snowflake_database.cms_medicare.name
  name     = "STREAMING_STAGING"
  comment  = "dbt staging layer for streaming data"
}

resource "snowflake_schema" "streaming_marts" {
  database = snowflake_database.cms_medicare.name
  name     = "STREAMING_MARTS"
  comment  = "Final consumable models for CMS Medicare analytics"
}

# Larger warehouse for Spark → Snowflake writes
resource "snowflake_warehouse" "streaming_compute" {
  name                = "STREAMING_WH"
  warehouse_size      = "SMALL"
  auto_suspend        = 120
  auto_resume         = true
  max_cluster_count   = 2      # multi-cluster for concurrent streaming writes
  min_cluster_count   = 1
  scaling_policy      = "ECONOMY"
  comment             = "Warehouse for Spark Structured Streaming writes to Snowflake"
}

# Snowflake stage pointing to S3 prefix for this project
resource "snowflake_stage" "cms_s3_stage" {
  name        = "CMS_S3_STAGE"
  database    = snowflake_database.cms_medicare.name
  schema      = snowflake_schema.streaming_raw.name
  url         = "s3://olist-data-lake-leeyao-oregon/cms-medicare/"
  credentials = "AWS_KEY_ID='${var.aws_access_key_id}' AWS_SECRET_KEY='${var.aws_secret_access_key}'"
  comment     = "External stage pointing to CMS Medicare S3 prefix"
}
```

### s3.tf — Project-Level S3 Prefix

Since Project 3 and Project 4 share the same S3 bucket with project-level prefixes, rather than creating a new bucket, we define an S3 object that acts as the prefix "folder":

```hcl
# S3 prefix object for CMS Medicare project data
# Shared bucket: olist-data-lake-leeyao-oregon
# Project prefix: cms-medicare/

resource "aws_s3_object" "cms_prefix" {
  bucket  = "olist-data-lake-leeyao-oregon"
  key     = "cms-medicare/"
  content = ""

  tags = {
    Project     = "CMS-Medicare-Streaming"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}
```

---

## 7. The Full Terraform Workflow

Once your `.tf` files are written, the workflow is three commands:

```bash
# Step 1: Initialize — download providers, set up backend
terraform init

# Step 2: Plan — preview what will be created/changed/destroyed
terraform plan

# Step 3: Apply — execute the changes
terraform apply
```

### What `terraform plan` output looks like:

```
Terraform will perform the following actions:

  # aws_s3_bucket.data_lake will be created
  + resource "aws_s3_bucket" "data_lake" {
      + bucket = "olist-data-lake-leeyao-oregon"
      + region = "us-west-2"
    }

  # snowflake_database.bts_flights will be created
  + resource "snowflake_database" "bts_flights" {
      + name = "BTS_FLIGHTS_DB"
    }

  # snowflake_warehouse.compute will be created
  + resource "snowflake_warehouse" "compute" {
      + name           = "COMPUTE_WH"
      + warehouse_size = "X-SMALL"
      + auto_suspend   = 60
    }

Plan: 12 to add, 0 to change, 0 to destroy.
```

You see exactly what will happen before anything changes. This is the safety net that manual infrastructure creation completely lacks.

---

## 8. Version Control and Reproducibility

Once your Terraform files are committed to Git, your infrastructure becomes:

**Reviewable**: every infrastructure change is a pull request. Team members can review, comment, and approve before anything is applied.

**Auditable**: `git log` shows who changed what infrastructure and when.

**Reproducible**: a new team member or a CI/CD system can recreate the entire environment with three commands:

```bash
git clone https://github.com/jonathanlyao/tech-blog.git
cd project3-terraform
terraform init && terraform apply
```

### What to commit vs. what to exclude

```gitignore
# .gitignore for Terraform projects

# State file — contains sensitive resource details, never commit
terraform.tfstate
terraform.tfstate.backup

# Variable values file — contains secrets
terraform.tfvars

# Local Terraform directory
.terraform/
.terraform.lock.hcl  # commit this one — it pins provider versions
```

**Commit**: all `.tf` files, `terraform.lock.hcl`
**Never commit**: `terraform.tfvars`, `*.tfstate`

---

## 9. One-Command Teardown and Rebuild

This is where Terraform delivers its most immediate practical value for portfolio projects.

**Teardown** (remove all infrastructure to avoid cloud costs):

```bash
terraform destroy
```

```
Terraform will perform the following actions:

  # aws_s3_bucket.data_lake will be destroyed
  # snowflake_database.bts_flights will be destroyed
  # snowflake_warehouse.compute will be destroyed
  ...

Plan: 0 to add, 0 to change, 12 to destroy.

Do you really want to destroy all resources? yes

Destroy complete! Resources: 12 destroyed.
```

**Rebuild** (restore everything for a demo or interview):

```bash
terraform apply
# All 12 resources recreated in under 2 minutes
```

Without Terraform, recreating the same environment after a teardown means going back to the AWS Console, Snowflake UI, and repeating every manual step — with no guarantee you'll remember them all correctly.

---

## 10. What Terraform Does Not Replace

Terraform manages **infrastructure**, not **data** or **application logic**.

| Terraform manages | Terraform does NOT manage |
|---|---|
| S3 bucket creation and configuration | Files inside the S3 bucket |
| Snowflake database and warehouse | Data inside Snowflake tables |
| Docker network and volumes | Docker container runtime behavior |
| IAM roles and policies | Application code and pipeline logic |
| Kafka topic configuration | Messages inside Kafka topics |

Your Python ingestion scripts, dbt models, Airflow DAGs, and Spark jobs all remain exactly where they are. Terraform only handles the underlying infrastructure those tools run on.

---

## 11. Summary and Next Steps

Terraform brings three concrete capabilities to a data engineering portfolio that manual infrastructure creation cannot provide:

**Reproducibility**: the entire infrastructure of a project can be recreated from scratch with `terraform apply` — on any machine, by anyone with the credentials.

**Version control**: infrastructure changes are code changes. Every modification is tracked, reviewable, and reversible through Git history.

**One-command teardown and rebuild**: spin down all cloud resources to avoid costs, then bring everything back up before a demo in under two minutes.

For Project 3 and Project 4, the practical application is direct: the S3 bucket, Snowflake databases, schemas, warehouses, IAM roles, and Docker networks that were created manually can all be expressed as `.tf` files, committed to Git alongside the pipeline code, and executed with three commands.

The infrastructure is no longer something you remember — it's something you read.

---

*This article is based on hands-on experience from two portfolio projects:*
*Project 3 — US DOT BTS Flights end-to-end batch pipeline (Python → AWS S3 → Snowflake → dbt Core four-layer model → Apache Airflow);*
*Project 4 — CMS Medicare Spending & Hospital Quality streaming pipeline (Kafka → Spark Structured Streaming → Snowflake → dbt → Airflow).*
*Full source code available on [GitHub](https://github.com/jonathanlyao).*