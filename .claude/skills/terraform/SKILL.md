---
name: terraform
description: Terraform infrastructure patterns for Polaris services on AWS and Azure. All projects must use S3+DynamoDB remote state (mandatory). Covers module structure, backend configuration, IAM role assumption, environment separation (dev/staging/prod), KMS encryption, security best practices, HIPAA compliance, naming conventions, and multi-cloud (EKS, RDS, VPC, SES, Lambda, AKS, APIM).
---

# Terraform Patterns

Terraform best practices derived from `polaris-email-integration` and `polaris-infra`. All Polaris infrastructure is AWS-first with a secondary Azure footprint. These patterns are mandatory for all new Terraform code.

**Core Mandate:** All Polaris Terraform projects — whether service-specific or shared infrastructure — must use S3 + DynamoDB remote state with KMS encryption. Local state is prohibited. This is a security and compliance requirement.

## When to Activate

- Creating new Terraform modules or environments
- Adding AWS or Azure infrastructure resources
- Reviewing Terraform for security, compliance, or structure issues
- Setting up remote state or backend configuration
- Managing environment separation (dev / staging / prod)
- Designing IAM roles, KMS encryption, or secrets management
- Configuring EKS, RDS, VPC, SES, Lambda, or other Polaris-used services

---

## Project Structure

All Terraform lives under a `terraform/` root. Split by cloud provider, then by environment-directories + reusable modules.

**Single-Service Repos** (e.g., polaris-email-integration):
```
terraform/aws/
├── environments/
│   ├── dev/
│   │   └── ses-email/          # Single service per environment
│   │       ├── terraform.tf    # Backend + versions
│   │       ├── providers.tf    # AWS provider
│   │       ├── main.tf         # Orchestrates modules
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── terraform.tfvars
│   ├── staging/
│   │   └── ses-email/          # Same structure as dev
│   │       ├── terraform.tf
│   │       ├── providers.tf
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── terraform.tfvars
│   ├── prod/
│   │   └── ses-email/          # Same structure as dev
│   │       ├── terraform.tf
│   │       ├── providers.tf
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── terraform.tfvars
│   └── global/                 # Reserved for ACM, Route53
├── modules/                    # Reusable local modules
│   ├── backend/                # S3 + DynamoDB for state
│   ├── kms/
│   ├── s3/
│   ├── iam/
│   ├── ses/
│   ├── lambda/
│   ├── lambda_layer/
│   ├── logs/
│   └── smtp_user/
```

**Multi-Component Repos** (e.g., polaris-infra — shared infrastructure):
```
terraform/aws/
├── environments/
│   ├── global/                 # ACM, Route53, EC2 key pairs — created once
│   ├── dev/
│   │   ├── infra/              # VPC, privatelinks
│   │   ├── eks/
│   │   ├── db/
│   │   ├── s3/
│   │   ├── cloudfront/
│   │   ├── api_gateway/
│   │   └── waf/
│   ├── staging/
│   │   └── (same structure)
│   └── prod/
│       └── (same structure)
└── modules/                    # Reusable, versioned local modules
    ├── vpc/
    ├── eks/
    ├── rds/
    ├── kms/
    ├── s3/
    ├── iam/
    ├── ses/
    ├── lambda/
    └── ...
```

**Rules:**
- Environment directories are **consumers** of modules — they contain only `module` blocks, data sources, and a backend config.
- Never put resource blocks directly into environment directories; always encapsulate in a module.
- Each environment sub-directory (`dev/infra`, `dev/db`, etc.) has its own state file — do not put everything in one state.
- Place environment-specific values in `terraform.tfvars`, never hardcoded in `.tf` files.

---

## Versions & Provider Pins

Every root module must define its `terraform` block in `terraform.tf`:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
```

**Rules:**
- Use `~> X.Y` (pessimistic constraint operator) for provider versions — allows patch updates, blocks major bumps.
- Lock exact versions in `.terraform.lock.hcl` and commit it.
- Minimum Terraform version: `>= 1.0`. Prefer `>= 1.5.7` for EKS modules (required for `check` blocks).

---

## Remote State (AWS S3 + DynamoDB)

**All Polaris Terraform — whether service-specific or shared infrastructure — must use S3-backed remote state with DynamoDB locking.** Never use local state. Local state is a security risk that violates HIPAA audit requirements and creates deployment coupling to individual developers.

```hcl
# terraform/aws/environments/dev/infra/terraform.tf
# Mandatory for all service and infra repos
terraform {
  backend "s3" {
    bucket         = "polaris-dev-tf-statefiles"
    region         = "us-east-1"
    encrypt        = true
    profile        = "waveum-dev"
    dynamodb_table = "polaris-dev-tf-state-lock"
    key            = "terraform/aws/environments/dev/infra/terraform.tfstate"
  }
}
```

**Mandatory backend rules:**
- `encrypt = true` is mandatory — state files contain secrets (passwords, API keys, credentials).
- `dynamodb_table` is mandatory for distributed locking (prevents concurrent applies and state corruption).
- Each environment sub-directory, and each service repo, gets its own `key` path — never share a state file between unrelated resources or across repos.
- State file is never committed to git (always in `.gitignore`).
- Use the AWS CLI profile (`profile = "waveum-dev"`) rather than static credentials.
- **All repos — service or infra — require S3 + DynamoDB backend.** Pre-provision the backend S3 bucket and DynamoDB table as part of environment bootstrap (terraform-init responsibility, not developer responsibility).

---

## Running Terraform (Per-Environment Directories)

**Single-Service Repos** (e.g., polaris-email-integration):

```bash
# Dev
cd terraform/aws/environments/dev/ses-email
terraform init
terraform plan
terraform apply

# Staging
cd terraform/aws/environments/staging/ses-email  # If following same structure
terraform init
terraform plan
terraform apply

# Prod
cd terraform/aws/environments/prod/ses-email
terraform init
terraform plan
terraform apply
```

**Multi-Component Repos** (e.g., polaris-infra):

```bash
# Deploy a single component (e.g., VPC in dev)
cd terraform/aws/environments/dev/infra
terraform init
terraform apply

# Deploy EKS in dev
cd terraform/aws/environments/dev/eks
terraform init
terraform apply
```

**Rules:**
- Run terraform **from the environment directory**, not from `terraform/aws/`
- Each environment directory has its own `terraform.tf` with a unique backend `key`
- Do not run terraform from the root `terraform/aws/` directory
- Each environment is independent — apply them separately, never in batch

---

## AWS Provider: Role Assumption

Polaris services assume a dedicated deploy role rather than using long-lived credentials:

```hcl
# terraform/aws/environments/dev/ses-email/providers.tf
provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = "arn:aws:iam::${var.aws_account_id}:role/waveum-polaris-ses-terraform-deploy-role"
  }
}
```

**Rules:**
- Every service must have its own scoped deploy role (least privilege).
- The deploy role policy lives alongside the Terraform code in `policies/<role>-policies.json`.
- Use `assume_role` in the provider block — never pass `access_key` / `secret_key` directly.
- For polaris-infra (shared infra), use an AWS CLI profile instead: `profile = "waveum-dev"` in the provider block.
- Example deploy role ARN: `arn:aws:iam::183047399597:role/waveum-polaris-ses-terraform-deploy-role`

---

## Backend Module (S3 + DynamoDB)

Every Polaris project must create and manage its own S3 bucket + DynamoDB table for Terraform state. Use the reusable `modules/backend/` pattern:

```hcl
# modules/backend/s3.tf
resource "aws_s3_bucket" "terraform_state" {
  bucket = "${var.project_name}-tf-statefiles"
  tags   = merge(var.tags, { Purpose = "terraform-state" })
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```hcl
# modules/backend/dynamodb.tf
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "${var.project_name}-tf-state-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = merge(var.tags, { Purpose = "terraform-state-lock" })
}
```

**Backend Module Integration:**

Call the module from the environment root (`main.tf`):

```hcl
module "backend" {
  source = "../../modules/backend"

  project_name = local.project_name
  environment  = local.environment
  aws_region   = var.aws_region
  kms_key_arn  = module.kms.key_arn
  tags         = local.tags

  depends_on = [module.kms]
}
```

**Bootstrap Process:**

```bash
cd terraform/aws/environments/dev/ses-email

# Step 1: Initialize without backend (uses local state)
terraform init

# Step 2: Apply to create S3 bucket + DynamoDB table
terraform apply

# Step 3: Re-initialize with remote backend (migrates state to S3)
terraform init -migrate-state

# Verify state is now remote
terraform state list
```

**Rules:**
- Backend module must be called **after** KMS module (depends on KMS key)
- S3 bucket name: `${project_name}-tf-statefiles`
- DynamoDB table name: `${project_name}-tf-state-lock`
- Always enable versioning, KMS encryption, and PITR
- Block all public access on S3 bucket

---

## Naming Conventions

All resource names follow: **`${project_name}-${environment}-${resource_type}[-suffix]`**

| Resource | Pattern | Example |
|----------|---------|---------|
| S3 bucket | `${project}-${env}-${purpose}-${random_hex}` | `polaris-ses-email-processing-dev-email-storage-a1b2c3` |
| KMS alias | `alias/${project}-${env}-key` | `alias/polaris-ses-email-processing-dev-key` |
| IAM role | `${project}-${env}-${role}` | `polaris-ses-email-processing-dev-lambda-role` |
| Lambda | `${project}-${env}-${function}` | `polaris-ses-email-processing-dev-email-processor` |
| CloudWatch log group | `/aws/${service}/${project}-${env}-${purpose}` | `/aws/ses/polaris-ses-email-processing-dev-email-processing` |
| Secrets Manager | `${project}-${env}-${purpose}` | `polaris-ses-email-processing-dev-smtp-credentials` |
| EKS cluster | `${project}-${env}` | `waveum-dev` |
| VPC | `${project}-${env}-vpc` | `waveum-dev-vpc` |

**Rules:**
- No uppercase letters. Use hyphens, not underscores, in resource names.
- Always include `environment` in the name to prevent accidental cross-environment resource confusion.
- Always append a random suffix (via `random_id`) to globally unique resources like S3 buckets.

---

## Tagging

All resources must include a standard tag block. Define it in `locals` at the root and pass it to every module.

```hcl
locals {
  tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "devops"         # for polaris-infra team-owned resources
  }
}
```

Pass via `tags = local.tags` or `tags = var.tags` in modules. Never set tags ad-hoc on individual resources.

---

## Variables & Outputs

### Variables (`variables.tf`)

```hcl
variable "aws_account_id" {
  description = "AWS account ID"
  type        = string
}

variable "aws_region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Short project identifier used in resource names"
  type        = string
}

variable "environment" {
  description = "Deployment environment: dev, staging, or prod"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}
```

**Rules:**
- Every variable must have a `description`.
- Use `validation` blocks for constrained inputs (environment, CIDR blocks, etc.).
- Avoid `default` for security-sensitive variables (account IDs, role ARNs) so they must be explicitly set.
- Use `sensitive = true` for secrets/passwords to suppress output.

### Outputs (`outputs.tf`)

```hcl
output "kms_key_arn" {
  description = "ARN of the KMS CMK used for encryption"
  value       = module.kms.key_arn
}

output "smtp_secret_arn" {
  description = "ARN of the Secrets Manager secret containing SMTP credentials"
  value       = module.smtp_user.secret_arn
  sensitive   = true
}
```

**Rules:**
- Every output must have a `description`.
- Mark sensitive outputs with `sensitive = true`.
- Expose only what downstream consumers need — not every internal resource attribute.

---

## Environment Separation

Use `terraform.tfvars` files per environment; never use separate branches or Terragrunt unless the team has agreed to it.

```
environments/
├── dev/terraform.tfvars
├── staging/terraform.tfvars
└── prod/terraform.tfvars
```

```hcl
# environments/dev/terraform.tfvars
environment             = "dev"
kms_deletion_window     = 7
log_retention_days      = 7
recipient_emails        = ["agent@dev.polaris.waveum.io"]
notification_sender_domain = "dev.polaris.waveum.io"
```

```hcl
# environments/prod/terraform.tfvars
environment             = "prod"
kms_deletion_window     = 30          # Maximum protection
log_retention_days      = 2557        # 7 years — HIPAA compliance
recipient_emails        = ["agent@polaris.waveum.io"]
notification_sender_domain = "polaris.waveum.io"
```

**Rules:**
- Production must always use maximum `kms_deletion_window` (30 days).
- HIPAA environments must retain logs for at least 2557 days (7 years).
- `email_operations_principals` should be empty in prod unless explicitly required with justification.

---

## KMS Encryption

Every Polaris environment uses a Customer-Managed Key (CMK) for sensitive data. Create one CMK per environment:

```hcl
# modules/kms/main.tf
resource "aws_kms_key" "this" {
  description             = "KMS CMK for HIPAA-compliant data encryption - ${var.project_name}-${var.environment}"
  deletion_window_in_days = var.deletion_window_days
  enable_key_rotation     = true   # MANDATORY — never disable
  policy                  = data.aws_iam_policy_document.kms_policy.json
  tags                    = var.tags
}

resource "aws_kms_alias" "this" {
  name          = "alias/${var.project_name}-${var.environment}-key"
  target_key_id = aws_kms_key.this.key_id
}
```

**KMS Policy principals to include** (restrict to minimum needed):
- Root account (`arn:aws:iam::${account_id}:root`) — full admin
- CloudWatch Logs — with `kms:ViaService` condition
- SES — with `aws:SourceAccount` condition
- S3 — with `aws:CallerAccount` condition
- Lambda — via the Lambda execution role

**Rules:**
- `enable_key_rotation = true` is **non-negotiable** for all CMKs.
- **CMK required for:** Terraform state files (S3 + DynamoDB), CloudWatch Logs (prod), Lambda env vars (prod), RDS (prod), DynamoDB (prod)
- **AWS-managed key acceptable for:** Non-sensitive S3 buckets (temporary artifacts, build outputs, non-PII logs), dev/staging resources, transient data with short lifespans
- **Secrets Manager always uses AWS-managed key** (`aws/secretsmanager`) — never specify `kms_key_id`

---

## S3 Security

**Encryption Strategy: Case-by-Case**

Choose encryption based on bucket sensitivity:

| Bucket Type | CMK Required? | Use Case | Notes |
|---|---|---|---|
| Terraform state | ✅ **Yes** | Remote state files | Contains secrets, credentials, resources |
| Production data | ✅ **Yes** | Email, patient records, PII | HIPAA, audit compliance |
| CloudWatch Logs (prod) | ✅ **Yes** | Structured logs with tracing | May contain audit trails |
| Build artifacts (dev/staging) | ❌ No | CI/CD outputs, transient builds | Non-sensitive, short-lived |
| Lambda layer cache (dev) | ❌ No | Layer ZIP files, dev only | Easy to recreate, non-critical |
| SES bounce notifications | ✅ **Yes** | Email metadata, compliance logs | Required for SES integration audit trail |

**CMK Encryption (sensitive data):**

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}
```

**AWS-Managed Key (non-sensitive data):**

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

**All buckets require:**

```hcl
resource "aws_s3_bucket" "this" {
  bucket = "${var.project_name}-${var.environment}-${var.purpose}-${random_id.suffix.hex}"
  tags   = var.tags
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id
  rule { object_ownership = "BucketOwnerPreferred" }
}
```

**Rules:**
- All four public access block settings must be `true` — no exceptions.
- Versioning enabled on all buckets.
- Choose encryption type (CMK or AES256) based on sensitivity (see table above).
- Random suffix on bucket name to ensure global uniqueness.

---

## IAM Patterns

### Least-Privilege Role

```hcl
resource "aws_iam_role" "lambda_role" {
  name               = "${var.project_name}-${var.environment}-lambda-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
  tags               = var.tags
}

data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role_policy" "lambda_policy" {
  name   = "${var.project_name}-${var.environment}-lambda-policy"
  role   = aws_iam_role.lambda_role.id
  policy = data.aws_iam_policy_document.lambda_permissions.json
}

data "aws_iam_policy_document" "lambda_permissions" {
  statement {
    actions   = ["s3:GetObject", "s3:ListBucket"]
    resources = [var.s3_bucket_arn, "${var.s3_bucket_arn}/*"]
  }
  statement {
    actions   = ["kms:Decrypt", "kms:GenerateDataKey"]
    resources = [var.kms_key_arn]
  }
  statement {
    actions   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
    resources = ["arn:aws:logs:*:*:*"]
  }
}
```

### Dynamic Principal ARN Resolution

When principals can be specified as either usernames or full ARNs:

```hcl
locals {
  resolved_principals = [
    for p in var.principals :
    can(regex("^arn:aws:iam::", p))
      ? p
      : "arn:aws:iam::${var.aws_account_id}:user/${p}"
  ]
}
```

**Rules:**
- Use `data.aws_iam_policy_document` for all IAM policies — never inline JSON strings.
- Apply conditions (`aws:SourceAccount`, `aws:CallerAccount`, `kms:ViaService`) to restrict service principals.
- Create roles conditionally with `count = length(var.principals) > 0 ? 1 : 0` when principals may be empty.
- SMTP users must restrict `ses:SendEmail` to specific sender domains only.

---

## Lambda Patterns

### Separate Layer from Function Code

```hcl
resource "aws_lambda_layer_version" "dependencies" {
  layer_name               = "${var.project_name}-${var.environment}-dependencies"
  filename                 = "${path.module}/dependencies.zip"
  source_code_hash         = filebase64sha256("${path.module}/dependencies.zip")
  compatible_runtimes      = ["python3.12"]
  compatible_architectures = ["x86_64"]
}

resource "aws_lambda_function" "this" {
  function_name = "${var.project_name}-${var.environment}-${var.function_name}"
  runtime       = "python3.12"
  handler       = "lambda_function.lambda_handler"
  role          = var.lambda_role_arn
  filename      = "${path.module}/lambda_function.zip"
  source_code_hash = filebase64sha256("${path.module}/lambda_function.zip")
  layers        = [aws_lambda_layer_version.dependencies.arn]
  kms_key_arn   = var.kms_key_arn

  environment {
    variables = {
      S3_BUCKET   = var.s3_bucket_name
      ENVIRONMENT = var.environment
    }
  }

  tags = var.tags
}
```

**Rules:**
- Always include `source_code_hash` — Terraform only re-deploys Lambda when the hash changes.
- Separate heavy dependencies (boto3, etc.) into a Lambda Layer — reduces function package to ~5KB vs 45MB+ and speeds deployments by ~6×.
- Always encrypt Lambda environment variables with the project KMS key (`kms_key_arn`).
- Use `python3.12` as the default runtime unless a different runtime is explicitly justified.

---

## Secrets Manager

```hcl
resource "aws_secretsmanager_secret" "smtp_credentials" {
  name        = "${var.project_name}-${var.environment}-smtp-credentials"
  description = "SES SMTP credentials for outbound email"
  # No kms_key_id — relies on AWS-managed key (aws/secretsmanager)
  # CMK is not used for Secrets Manager to reduce complexity
  tags        = var.tags
}

resource "aws_secretsmanager_secret_version" "smtp_credentials" {
  secret_id = aws_secretsmanager_secret.smtp_credentials.id
  secret_string = jsonencode({
    smtp_host     = "email-smtp.${var.aws_region}.amazonaws.com"
    smtp_port     = "587"
    smtp_user     = aws_iam_access_key.smtp_key.id
    smtp_password = aws_iam_access_key.smtp_key.ses_smtp_password_v4
  })
}
```

**Rules:**
- Secrets Manager uses the **AWS-managed key** (`aws/secretsmanager`) — do not set `kms_key_id` on secrets.
- Use `jsonencode()` to store structured secrets, not raw strings.
- Mark Terraform outputs referencing secret ARNs/values as `sensitive = true`.
- Never output the actual secret value — only the ARN.

---

## VPC Design (AWS)

Standard CIDR allocation for Polaris environments:

```hcl
module "vpc" {
  source = "../../../modules/vpc"

  cidr             = "10.48.0.0/16"
  azs              = ["us-east-1a", "us-east-1b"]

  public_subnets   = ["10.48.1.0/24", "10.48.2.0/24"]
  private_subnets  = ["10.48.4.0/23", "10.48.6.0/23"]
  intra_subnets    = ["10.48.10.0/24", "10.48.11.0/24"]
  database_subnets = ["10.48.13.0/24", "10.48.14.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true    # Cost optimization for non-prod; use one_nat_gateway_per_az = true for prod

  enable_dns_hostnames = true
  enable_dns_support   = true

  # VPC Flow Logs — mandatory for security monitoring
  enable_flow_log                                 = true
  flow_log_destination_type                       = "cloud-watch-logs"
  create_flow_log_cloudwatch_log_group            = true
  create_flow_log_cloudwatch_iam_role             = true
  flow_log_traffic_type                           = "ALL"
  flow_log_max_aggregation_interval               = 60
  flow_log_cloudwatch_log_group_retention_in_days = 30
}
```

**Rules:**
- VPC Flow Logs are mandatory in all environments with `flow_log_traffic_type = "ALL"`.
- Use `single_nat_gateway = true` for dev/staging (cost saving). Use `one_nat_gateway_per_az = true` for prod (HA).
- Database subnets must be isolated — never in public or private subnets.
- Always enable DNS hostnames and DNS support.

---

## CloudWatch Logs Retention

| Environment | Retention |
|-------------|-----------|
| dev | 7 days |
| staging | 30 days |
| prod | 2557 days (7 years — HIPAA) |

```hcl
resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/${var.service}/${var.project_name}-${var.environment}-${var.purpose}"
  retention_in_days = var.retention_in_days
  kms_key_id        = var.kms_key_arn
  tags              = var.tags
}
```

**Always pass `kms_key_id`** — CloudWatch log groups containing PHI or security events must be encrypted at rest with the project CMK.

---

## Azure Provider & State

```hcl
# terraform.tf — versions, backend, and provider version constraints in one block
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "polaris-tfstate-rg"
    storage_account_name = "polaristfstate"
    container_name       = "tfstate"
    key                  = "dev/terraform.tfstate"
  }
}

# providers.tf
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

Azure modules in use: `aks`, `apim`, `dns`, `key_vault`, `monitoring`, `network`, `sql`.

---

## Security Checklist

Before every `terraform apply` or PR merge, verify:

- [ ] No hardcoded credentials (`access_key`, `secret_key`, passwords) in any `.tf` file
- [ ] `enable_key_rotation = true` on all `aws_kms_key` resources
- [ ] All `aws_s3_bucket_public_access_block` has all four `true`
- [ ] `encrypt = true` in all S3 backends
- [ ] `dynamodb_table` set for state locking in all shared backends
- [ ] All `aws_cloudwatch_log_group` resources have `kms_key_id` set
- [ ] `aws_secretsmanager_secret` resources do **not** set `kms_key_id` (relies on AWS-managed key `aws/secretsmanager`)
- [ ] Lambda functions have `kms_key_arn` for environment variable encryption
- [ ] IAM policies use `data.aws_iam_policy_document` not inline JSON
- [ ] No `*` wildcards in IAM resource ARNs (except CloudWatch log ARNs)
- [ ] `sensitive = true` set on outputs that contain secrets or ARNs of secrets
- [ ] VPC Flow Logs enabled with `flow_log_traffic_type = "ALL"`
- [ ] Production KMS deletion window = 30 days
- [ ] Production log retention = 2557 days (HIPAA)

---

## Module Design Rules

1. **Single responsibility** — one module manages one logical resource group (e.g., `kms`, `s3`, `iam`). Never combine unrelated resources.
2. **`path.module`** — always use `path.module` for file references inside a module (e.g., Lambda zip files).
3. **No provider blocks in modules** — only in root modules / environment directories.
4. **Tag passthrough** — accept a `tags` variable and apply to every taggable resource.
5. **Output everything useful** — ARNs, IDs, names. Consumers need them; don't force data source lookups.
6. **Conditional resources** — use `count = var.enabled ? 1 : 0` (or `for_each`) to make resources optional.
7. **`depends_on` sparingly** — only when Terraform cannot infer dependency from references.

---

## Reference Implementations

**polaris-email-integration** — Single-service Terraform project with S3 + DynamoDB state:
- Per-environment directories: `environments/dev/ses-email/`, `environments/staging/ses-email/`, `environments/prod/ses-email/`
- Reusable `modules/backend/`, `modules/kms/`, `modules/s3/`, etc.
- Bootstrap pattern for creating state backend
- Reference: `polaris-email-integration`

**polaris-infra** — Multi-component shared infrastructure:
- Per-component directories: `environments/dev/infra/`, `environments/dev/eks/`, `environments/dev/db/`, etc.
- Each component has independent state file
- Reference: `polaris-infra`

Both projects demonstrate the **same core pattern**: remote S3 + DynamoDB state with KMS encryption.

