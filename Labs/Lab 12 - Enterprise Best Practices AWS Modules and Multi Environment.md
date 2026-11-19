# Lab 12 — Enterprise Best Practices: Folder Structure, Naming, Tagging, Secrets, and Multi-Environment AWS Modules

## Description

You apply **enterprise Terraform practices** on **AWS**: a standard **folder structure**, **naming** and **tagging** standards, **secrets management** (no credentials in Git), and an **environment strategy** for **Dev**, **QA**, **UAT**, and **Production**. You build **reusable enterprise modules** (`naming`, `s3-audit-bucket`) and wire four **environment roots** that share the same module contract but differ by `terraform.tfvars`. **Apply only in Dev** to control cost; run **`terraform plan`** in the other environments to verify naming, tags, and configuration drift.

## Prerequisites

- Completed [Lab 09](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md), [Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md), and [Lab 11](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md) (or equivalent module, state, and Git workflow knowledge)  
- `aws sts get-caller-identity` succeeds; S3 create/delete; SSM **PutParameter** / **GetParameter** for one lab secret  
- Terraform 1.5+  
- Optional: remote **S3 backend** from Lab 10 (commented `backend.tf` snippets per environment)

---

## Topic explanation

### Enterprise folder structure

Platform teams keep **modules** separate from **environment roots** so application teams consume stable contracts while platform engineers evolve implementation.

```text
enterprise-aws/
├── README.md
├── docs/
│   ├── NAMING.md
│   └── TAGGING.md
├── modules/
│   ├── naming/                 # Pure logic: consistent resource name prefix
│   └── s3-audit-bucket/        # Reusable S3 with security baselines
└── environments/
    ├── dev/
    ├── qa/
    ├── uat/
    └── production/
```

| Layer | Responsibility | State |
|-------|----------------|-------|
| **`modules/`** | Reusable building blocks; README; semver in Git | Embedded in parent state |
| **`environments/<env>/`** | Compose modules; env-specific variables | **One state file per environment** |
| **`docs/`** | Human policies (naming, tagging) | Not applied by Terraform |

**Never** mix Dev and Production resources in one root module or one state file.

### Naming standards

Consistent names support **search**, **IAM conditions**, **billing**, and **incident response**.

**Pattern (this lab):**

```text
{organization}-{business_unit}-{environment}-{region_code}-{workload}-{purpose}
```

| Segment | Example | Notes |
|---------|---------|-------|
| `organization` | `nwt` | Short org code |
| `business_unit` | `finops` | Team or product line |
| `environment` | `dev`, `qa`, `uat`, `production` | Lowercase in names; map from variable |
| `region_code` | `use1` | Short code for `us-east-1` |
| `workload` | `audit` | Application or data domain |
| `purpose` | `logs` | Resource role |

**Rules:**

- Lowercase, hyphens (S3 allows; adapt for resources with stricter rules)  
- **Globally unique** suffix for S3 (add initials or account id in tfvars)  
- No secrets or PII in resource names  

### Tagging standards

Tags drive **cost allocation**, **ownership**, and **compliance**. Enterprises often enforce a **minimum tag set** on every resource.

| Tag key | Required | Example (Dev) |
|---------|----------|----------------|
| `Environment` | Yes | `dev` |
| `Application` | Yes | `northwind-audit` |
| `CostCenter` | Yes | `CC-1042` |
| `Owner` | Yes | `platform-team@contoso.com` |
| `ManagedBy` | Yes | `terraform` |
| `DataClassification` | Yes | `internal` (stricter in UAT/Production) |

Use **`provider "aws" { default_tags { ... } }`** for org-wide defaults, then **`merge()`** module-specific tags. This lab validates required keys inside the **S3 module** so missing tags fail at plan time.

### Secrets management

| Do | Do not |
|----|--------|
| Store secrets in **AWS Systems Manager Parameter Store** (SecureString) or **Secrets Manager** | Commit `terraform.tfvars` with secrets |
| Read with `data "aws_ssm_parameter"` (`with_decryption = true`) | Put API keys in `variables.tf` defaults |
| Document paths in `terraform.tfvars.example` (path only, no value) | Output secret values from Terraform |
| Inject secrets at **runtime** (CI OIDC, instance roles) | Pass secrets on the command line in shared logs |

**Lab pattern:** one SecureString parameter per environment path, e.g. `/northwind/dev/audit/webhook_token`. Terraform references it for a **non-output** local (proves wiring without exposing the value).

### Environment strategy

| Environment | Purpose | Typical access | This lab |
|-------------|---------|----------------|----------|
| **Dev** | Fast iteration; developer sandboxes | Broad write | **`terraform apply`** |
| **QA** | Automated test; parity checks | CI service principal | **`plan` only** |
| **UAT** | Business acceptance | Controlled write | **`plan` only** |
| **Production** | Live workloads | Change windows; approvals | **`plan` only** |

**Enterprise patterns (concept):**

- **Account per environment** (strongest isolation) or **OU + SCPs** in AWS Organizations  
- **Separate state keys**: `environments/dev/terraform.tfstate`, `.../production/...`  
- **Promotion**: merge to `main` → pipeline applies Dev → automated gates → QA → UAT → Production  
- **Variable files**: `terraform.tfvars` per env (gitignored); committed **`terraform.tfvars.example`** only  

---

## Use case / scenario

**Northwind Traders** hired you as a **platform engineer**. FinOps must log audit events to **private S3 buckets** in four lifecycle stages. Leadership mandates:

1. One **enterprise repo layout** (`modules/` + `environments/{dev,qa,uat,production}`)  
2. **Naming** and **tagging** policies enforced in code  
3. **Webhook tokens** in **SSM**, not in Git  
4. The same **`s3-audit-bucket`** module in every environment with environment-specific classification and retention hints  

You deliver the repo under **`labs/work/lab-12/enterprise-aws/`**, validate all four roots, **apply Dev**, and **`plan`** the upper environments to compare bucket names and tags.

---

## Step-by-step instructions

> **Lab root folder:** From **repository root**, use **`labs/work/lab-12/enterprise-aws/`**.

### Step 1 — Verify AWS access

```powershell
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
```

### Step 2 — Create enterprise folder structure

```powershell
$root = "labs\work\lab-12\enterprise-aws"
New-Item -ItemType Directory -Path "$root\docs" -Force | Out-Null
New-Item -ItemType Directory -Path "$root\modules\naming" -Force | Out-Null
New-Item -ItemType Directory -Path "$root\modules\s3-audit-bucket" -Force | Out-Null
foreach ($env in @("dev","qa","uat","production")) {
  New-Item -ItemType Directory -Path "$root\environments\$env" -Force | Out-Null
}
Set-Location $root
```

```text
labs/work/lab-12/enterprise-aws/
  README.md
  docs/
  modules/
    naming/
    s3-audit-bucket/
  environments/
    dev/ | qa/ | uat/ | production/
```

### Step 3 — Policy docs (naming and tagging)

#### `docs/NAMING.md`

```markdown
# Northwind naming standard

Pattern: `{organization}-{business_unit}-{environment}-{region_code}-{workload}-{purpose}`

- Use lowercase and hyphens unless AWS restricts the resource type.
- S3 bucket names: append a unique suffix (initials) in tfvars.
- Production uses environment code `production` in the name segment (not `prod`).
```

#### `docs/TAGGING.md`

```markdown
# Northwind mandatory tags

Required on all resources: Environment, Application, CostCenter, Owner, ManagedBy, DataClassification.

| Environment | DataClassification default |
|-------------|----------------------------|
| dev | internal |
| qa | internal |
| uat | confidential |
| production | confidential |
```

### Step 4 — Module `modules/naming`

#### `modules/naming/variables.tf`

```hcl
variable "organization" {
  type        = string
  description = "Short organization code"
}

variable "business_unit" {
  type        = string
  description = "Business unit or product line"
}

variable "environment" {
  type        = string
  description = "Environment name: dev, qa, uat, production"

  validation {
    condition     = contains(["dev", "qa", "uat", "production"], var.environment)
    error_message = "environment must be dev, qa, uat, or production."
  }
}

variable "region_code" {
  type        = string
  description = "Short region code, e.g. use1 for us-east-1"
}

variable "workload" {
  type        = string
  description = "Workload or application domain"
}

variable "purpose" {
  type        = string
  description = "Resource purpose segment"
}

variable "unique_suffix" {
  type        = string
  description = "Globally unique suffix (initials, account id fragment)"
}
```

#### `modules/naming/main.tf`

```hcl
locals {
  name_prefix = lower(
    "${var.organization}-${var.business_unit}-${var.environment}-${var.region_code}-${var.workload}-${var.purpose}"
  )

  bucket_name = lower("${local.name_prefix}-${var.unique_suffix}")
}
```

#### `modules/naming/outputs.tf`

```hcl
output "name_prefix" {
  description = "Hyphenated name prefix for resources"
  value       = local.name_prefix
}

output "bucket_name" {
  description = "S3 bucket name including unique suffix"
  value       = local.bucket_name
}
```

### Step 5 — Enterprise module `modules/s3-audit-bucket`

#### `modules/s3-audit-bucket/versions.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.70"
    }
  }
}
```

#### `modules/s3-audit-bucket/variables.tf`

```hcl
variable "bucket_name" {
  type        = string
  description = "Globally unique bucket name from naming module"
}

variable "tags" {
  type        = map(string)
  description = "Tags merged onto the bucket (must include mandatory keys)"

  validation {
    condition = alltrue([
      for key in ["Environment", "Application", "CostCenter", "Owner", "ManagedBy", "DataClassification"] :
      contains(keys(var.tags), key)
    ])
    error_message = "tags must include Environment, Application, CostCenter, Owner, ManagedBy, and DataClassification."
  }
}

variable "force_destroy" {
  type        = bool
  description = "Allow bucket delete when non-empty (true in dev labs only)"
  default     = false
}

variable "kms_key_arn" {
  type        = string
  description = "Optional CMK ARN; if null, SSE-S3 (AES256) is used"
  default     = null
}
```

#### `modules/s3-audit-bucket/main.tf`

```hcl
resource "aws_s3_bucket" "audit" {
  bucket        = var.bucket_name
  force_destroy = var.force_destroy
  tags          = var.tags
}

resource "aws_s3_bucket_public_access_block" "audit" {
  bucket = aws_s3_bucket.audit.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "audit" {
  bucket = aws_s3_bucket.audit.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.kms_key_arn == null ? "AES256" : "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = var.kms_key_arn != null
  }
}

resource "aws_s3_bucket_versioning" "audit" {
  bucket = aws_s3_bucket.audit.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

#### `modules/s3-audit-bucket/outputs.tf`

```hcl
output "bucket_id" {
  description = "Bucket name"
  value       = aws_s3_bucket.audit.id
}

output "bucket_arn" {
  description = "Bucket ARN"
  value       = aws_s3_bucket.audit.arn
}
```

#### `modules/s3-audit-bucket/README.md`

```markdown
# s3-audit-bucket

Enterprise audit log bucket: private, encrypted, versioned; enforces mandatory tags via variable validation.

## Inputs

| Name | Required | Description |
|------|----------|-------------|
| bucket_name | yes | From `naming` module |
| tags | yes | Must include all mandatory tag keys |
| force_destroy | no | Lab/dev only |
| kms_key_arn | no | Optional CMK |

## Outputs

`bucket_id`, `bucket_arn`
```

### Step 6 — Shared environment pattern

Each environment under `environments/<env>/` uses the same files; only **`terraform.tfvars`** (from example) changes.

#### `environments/dev/providers.tf` (copy to qa, uat, production)

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.70"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Application = var.application
      Owner       = var.owner
      CostCenter  = var.cost_center
    }
  }
}
```

#### `environments/dev/variables.tf` (copy to qa, uat, production)

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "organization" {
  type = string
}

variable "business_unit" {
  type = string
}

variable "environment" {
  type = string
}

variable "region_code" {
  type = string
}

variable "workload" {
  type = string
}

variable "purpose" {
  type = string
}

variable "unique_suffix" {
  type = string
}

variable "application" {
  type = string
}

variable "owner" {
  type = string
}

variable "cost_center" {
  type = string
}

variable "data_classification" {
  type = string
}

variable "force_destroy" {
  type    = bool
  default = false
}

variable "ssm_webhook_token_path" {
  type        = string
  description = "SSM parameter path (SecureString); value never committed"
}
```

#### `environments/dev/main.tf` (copy to qa, uat, production)

```hcl
module "naming" {
  source = "../../modules/naming"

  organization    = var.organization
  business_unit   = var.business_unit
  environment     = var.environment
  region_code     = var.region_code
  workload        = var.workload
  purpose         = var.purpose
  unique_suffix   = var.unique_suffix
}

data "aws_ssm_parameter" "webhook_token" {
  name            = var.ssm_webhook_token_path
  with_decryption = true
}

locals {
  # Proves secret wiring; never output or log this value
  _webhook_configured = length(data.aws_ssm_parameter.webhook_token.value) > 0

  standard_tags = {
    Environment          = var.environment
    Application          = var.application
    CostCenter           = var.cost_center
    Owner                = var.owner
    ManagedBy            = "terraform"
    DataClassification   = var.data_classification
    Lab                  = "12-enterprise"
    WebhookConfigured  = local._webhook_configured ? "true" : "false"
  }
}

module "audit_bucket" {
  source = "../../modules/s3-audit-bucket"

  bucket_name   = module.naming.bucket_name
  tags          = local.standard_tags
  force_destroy = var.force_destroy
}
```

#### `environments/dev/outputs.tf` (copy to qa, uat, production)

```hcl
output "name_prefix" {
  value = module.naming.name_prefix
}

output "audit_bucket_id" {
  value = module.audit_bucket.bucket_id
}

output "ssm_parameter_name" {
  value     = var.ssm_webhook_token_path
  sensitive = false
}
```

#### `environments/dev/backend.tf` (optional — copy to other envs; change `key`)

```hcl
# terraform {
#   backend "s3" {
#     bucket         = "your-state-bucket"
#     key            = "northwind/environments/dev/terraform.tfstate"
#     region         = "us-east-1"
#     dynamodb_table = "terraform-locks"
#     encrypt        = true
#   }
# }
```

Use keys `.../qa/...`, `.../uat/...`, `.../production/...` for other environments.

### Step 7 — Environment-specific `terraform.tfvars.example`

Create **`terraform.tfvars`** locally (gitignored); copy from the matching example.

#### `environments/dev/terraform.tfvars.example`

```hcl
organization       = "nwt"
business_unit      = "finops"
environment        = "dev"
region_code        = "use1"
workload           = "audit"
purpose            = "logs"
unique_suffix      = "yourinitials"
application        = "northwind-audit"
owner              = "platform-team@contoso.com"
cost_center        = "CC-1042"
data_classification = "internal"
force_destroy      = true
ssm_webhook_token_path = "/northwind/dev/audit/webhook_token"
```

#### `environments/qa/terraform.tfvars.example`

```hcl
organization       = "nwt"
business_unit      = "finops"
environment        = "qa"
region_code        = "use1"
workload           = "audit"
purpose            = "logs"
unique_suffix      = "yourinitials"
application        = "northwind-audit"
owner              = "platform-team@contoso.com"
cost_center        = "CC-1042"
data_classification = "internal"
force_destroy      = false
ssm_webhook_token_path = "/northwind/qa/audit/webhook_token"
```

#### `environments/uat/terraform.tfvars.example`

```hcl
organization       = "nwt"
business_unit      = "finops"
environment        = "uat"
region_code        = "use1"
workload           = "audit"
purpose            = "logs"
unique_suffix      = "yourinitials"
application        = "northwind-audit"
owner              = "platform-team@contoso.com"
cost_center        = "CC-1042"
data_classification = "confidential"
force_destroy      = false
ssm_webhook_token_path = "/northwind/uat/audit/webhook_token"
```

#### `environments/production/terraform.tfvars.example`

```hcl
organization       = "nwt"
business_unit      = "finops"
environment        = "production"
region_code        = "use1"
workload           = "audit"
purpose            = "logs"
unique_suffix      = "yourinitials"
application        = "northwind-audit"
owner              = "platform-team@contoso.com"
cost_center        = "CC-1042"
data_classification = "confidential"
force_destroy      = false
ssm_webhook_token_path = "/northwind/production/audit/webhook_token"
```

### Step 8 — Root `README.md`

```markdown
# Northwind enterprise-aws (Lab 12)

## Layout
- `modules/naming` — canonical name prefix and bucket name
- `modules/s3-audit-bucket` — reusable audit bucket with tag enforcement
- `environments/*` — one root module per lifecycle stage

## Workflow
1. Create SSM SecureString per environment path
2. `terraform init` && `validate` in each environment
3. Apply **dev** only; plan qa, uat, production
4. destroy dev when finished

## Policies
See `docs/NAMING.md` and `docs/TAGGING.md`.
```

### Step 9 — Create SSM parameters (secrets lab step)

Run once per environment you will **plan** or **apply** (at minimum **dev**):

```powershell
aws ssm put-parameter --name "/northwind/dev/audit/webhook_token" --value "lab-dev-placeholder" --type SecureString --overwrite
aws ssm put-parameter --name "/northwind/qa/audit/webhook_token" --value "lab-qa-placeholder" --type SecureString --overwrite
aws ssm put-parameter --name "/northwind/uat/audit/webhook_token" --value "lab-uat-placeholder" --type SecureString --overwrite
aws ssm put-parameter --name "/northwind/production/audit/webhook_token" --value "lab-prod-placeholder" --type SecureString --overwrite
```

### Step 10 — Validate all environments

```powershell
foreach ($env in @("dev","qa","uat","production")) {
  Set-Location "environments\$env"
  Copy-Item terraform.tfvars.example terraform.tfvars -ErrorAction SilentlyContinue
  terraform fmt -recursive
  terraform init
  terraform validate
  Set-Location ..\..
}
```

The same `unique_suffix` is fine across environments because the **environment** segment is already part of the bucket name.

### Step 11 — Apply Dev; plan upper environments

```powershell
Set-Location environments\dev
# Ensure terraform.tfvars exists and unique_suffix is globally unique for S3
terraform plan
terraform apply
terraform output
```

```powershell
foreach ($env in @("qa","uat","production")) {
  Set-Location "environments\$env"
  terraform plan -out="$env.tfplan"
  Set-Location ..\..
}
```

Compare **planned** bucket names: environment segment should change (`nwt-finops-qa-...` vs `...-production-...`). Confirm **DataClassification** tags differ between dev/qa and uat/production in the plan output.

### Step 12 — Enterprise exercise (tag policy failure)

Intentionally remove `CostCenter` from `local.standard_tags` in **dev** `main.tf`, run `terraform plan`, and confirm the **variable validation** error from `s3-audit-bucket`. Restore the tag before continuing.

### Step 13 — Reusability check

Answer in your notes:

1. What would you change in the **module** to add a 30-day lifecycle rule without touching every environment root?  
2. Which single variable drives the S3 bucket name across all four environments?  
3. Where would Production **apply** run in a real org (workstation vs CI)?  

---

## Cleanup

```powershell
Set-Location environments\dev
terraform destroy
```

Optional — remove lab SSM parameters:

```powershell
aws ssm delete-parameter --name "/northwind/dev/audit/webhook_token"
aws ssm delete-parameter --name "/northwind/qa/audit/webhook_token"
aws ssm delete-parameter --name "/northwind/uat/audit/webhook_token"
aws ssm delete-parameter --name "/northwind/production/audit/webhook_token"
```

Remove `labs/work/lab-12/` when finished (under gitignored `work/`).

---

## Lab completion checklist

- [ ] Enterprise folder structure: `modules/`, `environments/{dev,qa,uat,production}`, `docs/`  
- [ ] **Naming** module produces consistent `name_prefix` and `bucket_name`  
- [ ] **s3-audit-bucket** module: encryption, public access block, versioning, mandatory tag validation  
- [ ] **Mandatory tags** present in all environments  
- [ ] **SSM** SecureString used; no secret values in committed files  
- [ ] **Dev** applied successfully; **QA / UAT / Production** planned  
- [ ] Optional: per-environment **S3 backend** keys documented  

## Self-review questions

| Topic | Question |
|-------|----------|
| Folder structure | Why are modules not copied into each environment folder? |
| Naming standards | When would you use `production` vs `prod` in AWS? |
| Tagging standards | How do `default_tags` and module `merge()` interact? |
| Secrets management | Why avoid `output` for SSM-backed secrets? |
| Environment strategy | Why is separate state per environment non-negotiable? |
| Dev / QA / UAT / Production | Who typically approves a Production apply? |

## Previous lab

[Lab 11 — Git Integration](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md).

## Next lab

[Lab 13 — HCP Terraform Workspaces](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md): workspace architecture, Terraform variables, variable sets, sensitive values, and environment variables for remote AWS runs.