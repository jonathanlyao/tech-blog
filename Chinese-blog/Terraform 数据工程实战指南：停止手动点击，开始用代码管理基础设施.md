# Terraform 数据工程实战指南：停止手动点击，开始用代码管理基础设施

> **每一个手动创建的资源，都是一个将来必须手动重建的资源。Terraform 终结这个循环。**

---

## 目录

1. [Terraform 是什么？](#1-terraform-是什么)
2. [数据工程师为什么需要它](#2-数据工程师为什么需要它)
3. [必须掌握的核心概念](#3-必须掌握的核心概念)
4. [问题所在：真实项目中的手动基础设施](#4-问题所在真实项目中的手动基础设施)
5. [Project 3 — 将 BTS Flights Pipeline 基础设施代码化](#5-project-3--将-bts-flights-pipeline-基础设施代码化)
6. [Project 4 — 将 Kafka + Spark 流处理基础设施代码化](#6-project-4--将-kafka--spark-流处理基础设施代码化)
7. [完整的 Terraform 工作流](#7-完整的-terraform-工作流)
8. [版本控制与可复现性](#8-版本控制与可复现性)
9. [一键销毁与重建](#9-一键销毁与重建)
10. [Terraform 不能替代什么](#10-terraform-不能替代什么)
11. [总结与后续步骤](#11-总结与后续步骤)

---

## 1. Terraform 是什么？

Terraform 是由 HashiCorp 开发的开源**基础设施即代码（Infrastructure as Code，IaC）**工具。它允许你用人类可读的配置文件来定义云基础设施——服务器、存储桶、数据库、网络、权限——然后通过命令行来创建、更新或销毁这些基础设施。

原来你需要登录 AWS Console，在页面上一步步点击创建 S3 bucket，现在你只需要写这几行：

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "olist-data-lake-leeyao-oregon"
}
```

运行 `terraform apply`，bucket 就创建好了。运行 `terraform destroy`，它就消失了。整个基础设施的状态都存在于代码中，可以提交到 Git、审查、版本化和共享。

Terraform 是**云无关的（cloud-agnostic）**：无论你在 AWS、Snowflake、GCP、Azure 还是 Kafka 上创建资源，工作流都完全一样。它通过一套叫做 **Provider** 的插件系统与各个平台通信。

---

## 2. 数据工程师为什么需要它

数据工程项目涉及的基础设施资源数量之多，往往超出预期。对于一个 Pipeline 项目，你可能手动创建了：

- S3 bucket（含特定配置：版本控制、生命周期规则、区域）
- Snowflake database、schema、warehouse、role、user
- 用于跨服务访问的 IAM role 和 policy
- 本地开发用的 Docker network 和 volume
- Kafka topic 及其配置
- Airflow connection 和环境变量

以上每一项都是手动创建的。这意味着：

- **无法复现**：在新机器或新环境中重建，必须把所有步骤重做一遍
- **无法审查**：没有任何记录说明创建了什么、什么时候创建、为什么这样配置
- **无法回滚**：某个配置改动导致问题，回退只能靠记忆和手工操作
- **终将遗忘**：六个月后，你不会记得当初具体是怎么搭起来的

Terraform 同时解决上述四个问题。

---

## 3. 必须掌握的核心概念

### Provider（提供商）

Provider 是让 Terraform 与特定平台通信的插件。每个 Provider 暴露一组可以管理的 **Resource**。

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
  account  = var.snowflake_account
  username = var.snowflake_username
  password = var.snowflake_password
  role     = "SYSADMIN"
}
```

### Resource（资源）

Resource 是单个基础设施对象——一个 S3 bucket、一个 Snowflake database、一个 Kafka topic。Resource 是 Terraform 配置的基本构建单元。

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "olist-data-lake-leeyao-oregon"
}
```

### Variable（变量）

Variable 让配置可复用，并将敏感信息从代码中隔离出去。

```hcl
variable "snowflake_account" {
  description = "Snowflake 账户标识符"
  type        = string
  sensitive   = true
}
```

### State（状态）

Terraform 维护一个**状态文件**（`terraform.tfstate`），它将你的配置映射到云上真实存在的资源。Terraform 通过这个文件判断哪些资源已经存在、哪些需要新建、哪些需要更新或删除。

### Plan 与 Apply

```bash
terraform plan    # 预览 Terraform 将要做什么——不执行任何变更
terraform apply   # 确认后执行变更
terraform destroy # 删除配置中定义的所有资源
```

---

## 4. 问题所在：真实项目中的手动基础设施

用我自己的项目经验来说明"手动基础设施"在实际中意味着什么。

### Project 3 — US DOT BTS Flights Pipeline

为了搭建这条 Pipeline，我手动执行了以下操作：

1. 登录 AWS Console → S3 → 在 `us-west-2` 创建 `olist-data-lake-leeyao-oregon`
2. 在控制台中手动配置 bucket 版本控制
3. 登录 Snowflake → 创建 database `BTS_FLIGHTS_DB`、schema `RAW`、warehouse `COMPUTE_WH`
4. 创建 Snowflake role，在 Worksheet 中运行一系列 SQL 语句授权
5. 创建 IAM user，赋予 S3 读取权限，手动下载 access key

**问题**：这些操作没有任何代码记录。如果需要重建这个环境——为了面试演示、新团队成员加入、或者误删了某个资源——必须凭记忆把每一步重做一遍。

### Project 4 — CMS Medicare 流处理 Pipeline

在 Project 3 的基础上，Project 4 还增加了：

1. Kafka + Zookeeper 的 Docker Compose 文件（部分代码化，但网络和 volume 配置是隐式的）
2. 用于流处理 schema 的额外 Snowflake 对象
3. 依赖特定 Snowflake warehouse 规格的 Spark 配置

整个基础设施横跨三个平台（AWS、Snowflake、Docker），是一块一块手动拼起来的，没有统一的 source of truth。

---

## 5. Project 3 — 将 BTS Flights Pipeline 基础设施代码化

以下是用 Terraform 定义 Project 3 全部基础设施的完整方式。

### 目录结构

```
project3-terraform/
├── main.tf           # provider 配置
├── variables.tf      # 输入变量声明
├── terraform.tfvars  # 变量实际值（永远不要提交到 Git）
├── s3.tf             # AWS S3 资源
├── iam.tf            # AWS IAM role 和 policy
├── snowflake.tf      # Snowflake 资源
└── outputs.tf        # 输出值（bucket 名称、warehouse 名称等）
```

### main.tf — Provider 配置

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
  description = "所有资源使用的 AWS 区域"
  type        = string
  default     = "us-west-2"
}

variable "s3_bucket_name" {
  description = "S3 数据湖 bucket 名称"
  type        = string
  default     = "olist-data-lake-leeyao-oregon"
}

variable "snowflake_account" {
  description = "Snowflake 账户标识符"
  type        = string
  sensitive   = true
}

variable "snowflake_username" {
  description = "Snowflake 登录用户名"
  type        = string
  sensitive   = true
}

variable "snowflake_password" {
  description = "Snowflake 登录密码"
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

# 启用版本控制——原来是在控制台手动配置的
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id

  versioning_configuration {
    status = "Enabled"
  }
}

# 阻断所有公开访问——安全基线
resource "aws_s3_bucket_public_access_block" "data_lake_public_access" {
  bucket = aws_s3_bucket.data_lake.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 生命周期规则：原始数据 90 天后转移到更便宜的存储类型
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

### iam.tf — Pipeline 访问 IAM Role

```hcl
# IAM policy：允许对数据湖 bucket 进行读写操作
resource "aws_iam_policy" "data_lake_access" {
  name        = "bts-flights-data-lake-access"
  description = "对 BTS Flights 数据湖 S3 bucket 的读写权限"

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

# Pipeline 服务账号 IAM user
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

### snowflake.tf — Snowflake 资源

```hcl
# Database
resource "snowflake_database" "bts_flights" {
  name    = "BTS_FLIGHTS_DB"
  comment = "BTS Flights Pipeline 数据库 — Project 3"
}

# Schema——dbt 四层模型结构
resource "snowflake_schema" "raw" {
  database = snowflake_database.bts_flights.name
  name     = "RAW"
  comment  = "原始摄入层——从 S3 直接 COPY INTO"
}

resource "snowflake_schema" "staging" {
  database = snowflake_database.bts_flights.name
  name     = "STAGING"
  comment  = "dbt staging 层——类型转换、字段重命名"
}

resource "snowflake_schema" "intermediate" {
  database = snowflake_database.bts_flights.name
  name     = "INTERMEDIATE"
  comment  = "dbt intermediate 层——业务逻辑"
}

resource "snowflake_schema" "marts" {
  database = snowflake_database.bts_flights.name
  name     = "MARTS"
  comment  = "dbt marts 层——最终可消费模型"
}

# 虚拟仓库
resource "snowflake_warehouse" "compute" {
  name           = "COMPUTE_WH"
  warehouse_size = "X-SMALL"
  auto_suspend   = 60
  auto_resume    = true
  comment        = "通用计算仓库——空闲 60 秒后自动挂起"
}

# dbt 和 Pipeline 使用的 Role
resource "snowflake_role" "pipeline_role" {
  name    = "BTS_PIPELINE_ROLE"
  comment = "供 dbt 和摄入脚本使用的角色"
}

# 向 pipeline role 授予权限
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
  description = "数据湖 S3 bucket 名称"
  value       = aws_s3_bucket.data_lake.bucket
}

output "s3_bucket_arn" {
  description = "数据湖 S3 bucket ARN"
  value       = aws_s3_bucket.data_lake.arn
}

output "snowflake_database" {
  description = "Snowflake database 名称"
  value       = snowflake_database.bts_flights.name
}

output "snowflake_warehouse" {
  description = "Snowflake warehouse 名称"
  value       = snowflake_warehouse.compute.name
}
```

---

## 6. Project 4 — 将 Kafka + Spark 流处理基础设施代码化

Project 4 增加了流处理层：Kafka 和 Zookeeper 通过 Docker 在本地运行，Snowflake 层也需要扩展以支持流数据。

### 目录结构

```
project4-terraform/
├── main.tf
├── variables.tf
├── terraform.tfvars
├── s3.tf             # 在共享 bucket 上使用项目级前缀
├── snowflake.tf      # 流处理专用 Snowflake 对象
├── docker.tf         # Docker network 和 volume 管理
└── outputs.tf
```

### docker.tf — Docker Network 与 Volume

Terraform 有一个 Docker Provider，可以管理本地 Docker 资源——network、volume、container 和 image。这将原来 Docker Compose 中隐式的网络配置，替换为显式的、有版本记录的声明。

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

# 流处理 Pipeline 专用 Docker network
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

# Kafka 数据持久化 volume
resource "docker_volume" "kafka_data" {
  name = "kafka-data-volume"

  labels {
    label = "project"
    value = "cms-medicare-streaming"
  }
}

# Zookeeper 数据持久化 volume
resource "docker_volume" "zookeeper_data" {
  name = "zookeeper-data-volume"

  labels {
    label = "project"
    value = "cms-medicare-streaming"
  }
}
```

### snowflake.tf — 流处理 Snowflake 对象

```hcl
# CMS Medicare 流处理项目数据库
resource "snowflake_database" "cms_medicare" {
  name    = "CMS_MEDICARE_DB"
  comment = "CMS Medicare 流处理 Pipeline 数据库 — Project 4"
}

# 流处理专用 schema
resource "snowflake_schema" "streaming_raw" {
  database = snowflake_database.cms_medicare.name
  name     = "STREAMING_RAW"
  comment  = "通过 Spark Structured Streaming 从 Kafka 写入的原始流数据"
}

resource "snowflake_schema" "streaming_staging" {
  database = snowflake_database.cms_medicare.name
  name     = "STREAMING_STAGING"
  comment  = "流数据的 dbt staging 层"
}

resource "snowflake_schema" "streaming_marts" {
  database = snowflake_database.cms_medicare.name
  name     = "STREAMING_MARTS"
  comment  = "CMS Medicare 分析的最终可消费模型"
}

# 用于 Spark → Snowflake 写入的更大规格仓库
resource "snowflake_warehouse" "streaming_compute" {
  name              = "STREAMING_WH"
  warehouse_size    = "SMALL"
  auto_suspend      = 120
  auto_resume       = true
  max_cluster_count = 2       # 多集群支持并发流写入
  min_cluster_count = 1
  scaling_policy    = "ECONOMY"
  comment           = "供 Spark Structured Streaming 写入 Snowflake 使用的仓库"
}

# 指向 S3 项目前缀的 Snowflake External Stage
resource "snowflake_stage" "cms_s3_stage" {
  name        = "CMS_S3_STAGE"
  database    = snowflake_database.cms_medicare.name
  schema      = snowflake_schema.streaming_raw.name
  url         = "s3://olist-data-lake-leeyao-oregon/cms-medicare/"
  credentials = "AWS_KEY_ID='${var.aws_access_key_id}' AWS_SECRET_KEY='${var.aws_secret_access_key}'"
  comment     = "指向 CMS Medicare S3 前缀的外部 Stage"
}
```

### s3.tf — 项目级 S3 前缀

由于 Project 3 和 Project 4 共用同一个 S3 bucket，以项目级前缀区分，这里不创建新 bucket，而是定义一个前缀"文件夹"对象：

```hcl
# CMS Medicare 项目数据的 S3 前缀对象
# 共享 bucket：olist-data-lake-leeyao-oregon
# 项目前缀：cms-medicare/

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

## 7. 完整的 Terraform 工作流

`.tf` 文件写好之后，整个工作流只有三条命令：

```bash
# 第一步：初始化——下载 Provider 插件，设置 backend
terraform init

# 第二步：预览——查看将要创建/修改/销毁的内容
terraform plan

# 第三步：执行——应用变更
terraform apply
```

### `terraform plan` 的输出示例：

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

在任何变更发生之前，你能看清楚将要发生什么。这是手动创建基础设施完全不具备的安全保障。

---

## 8. 版本控制与可复现性

Terraform 文件一旦提交到 Git，你的基础设施就具备了以下特性：

**可审查**：每一次基础设施变更都是一次 Pull Request。团队成员可以在执行前审查、评论和批准。

**可追溯**：`git log` 记录了谁在什么时候修改了什么基础设施。

**可复现**：新团队成员或 CI/CD 系统可以用三条命令重建整个环境：

```bash
git clone https://github.com/jonathanlyao/tech-blog.git
cd project3-terraform
terraform init && terraform apply
```

### 该提交什么，不该提交什么

```gitignore
# Terraform 项目的 .gitignore

# 状态文件——包含敏感资源详情，永远不要提交
terraform.tfstate
terraform.tfstate.backup

# 变量值文件——包含密钥和密码
terraform.tfvars

# 本地 Terraform 目录
.terraform/
.terraform.lock.hcl  # 这个要提交——它锁定了 Provider 版本
```

**提交**：所有 `.tf` 文件、`terraform.lock.hcl`
**绝不提交**：`terraform.tfvars`、`*.tfstate`

---

## 9. 一键销毁与重建

这是 Terraform 对 Portfolio 项目最直接的实用价值所在。

**销毁**（移除所有基础设施，避免云资源费用）：

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

**重建**（为演示或面试恢复全部环境）：

```bash
terraform apply
# 12 个资源在 2 分钟内全部重建完成
```

没有 Terraform，销毁之后重建同样的环境意味着重新回到 AWS Console、Snowflake UI，把每一步手动操作再走一遍——还不能保证你能完全记住每一个细节。

---

## 10. Terraform 不能替代什么

Terraform 管理的是**基础设施**，不是**数据**或**应用逻辑**。

| Terraform 管理的 | Terraform 不管理的 |
|---|---|
| S3 bucket 的创建与配置 | S3 bucket 里的文件 |
| Snowflake database 和 warehouse | Snowflake 表里的数据 |
| Docker network 和 volume | Docker 容器的运行时行为 |
| IAM role 和 policy | 应用代码和 Pipeline 逻辑 |
| Kafka topic 配置 | Kafka topic 里的消息 |

你的 Python 摄入脚本、dbt 模型、Airflow DAG 和 Spark 任务，全部保持原样不变。Terraform 只负责这些工具运行所依赖的底层基础设施。

---

## 11. 总结与后续步骤

Terraform 为数据工程 Portfolio 带来了三项手动创建基础设施无法实现的能力：

**可复现性**：一个项目的全部基础设施可以通过 `terraform apply` 从零重建——在任何机器上，由任何拥有相应凭证的人执行。

**版本控制**：基础设施的变更就是代码的变更。每一次修改都通过 Git 历史记录、可审查、可回滚。

**一键销毁与重建**：随时关停所有云资源避免费用，演示前两分钟内重建全部环境。

对于 Project 3 和 Project 4，应用是直接的：S3 bucket、Snowflake database、schema、warehouse、IAM role 以及 Docker network——这些原本手动创建的资源，都可以表达为 `.tf` 文件，与 Pipeline 代码一起提交到 Git，用三条命令执行。

基础设施不再是你需要记住的东西——它变成了你可以直接阅读的东西。

---

*本文基于以下实际项目经验整理而成：*
*Project 3 — US DOT BTS Flights 端到端批处理管道（Python → AWS S3 → Snowflake → dbt Core 四层模型 → Apache Airflow）；*
*Project 4 — CMS Medicare Spending & Hospital Quality 流处理管道（Kafka → Spark Structured Streaming → Snowflake → dbt → Airflow）。*
*完整代码见 [GitHub](https://github.com/jonathanlyao)。*