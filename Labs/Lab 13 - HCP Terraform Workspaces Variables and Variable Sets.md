# Lab 13 — Workspaces: Architecture, Variables, Variable Sets, Sensitive Values, and Environment Variables (HCP Terraform + AWS)

## Description

You learn **HCP Terraform** (formerly Terraform Cloud) **workspace architecture** and how teams supply configuration without committing secrets: **Terraform variables**, **variable sets**, **sensitive** values, and **environment variables** for provider authentication. You deploy a small **AWS S3** workload where **remote runs** read settings from the workspace instead of `terraform.tfvars`. This complements [Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md) (self-managed S3 state) and [Lab 12](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md) (multi-environment roots).

> **Naming note:** HCP **workspaces** (this lab) are **not** the same as the local CLI command `terraform workspace select`. Enterprises use HCP workspaces (or Terraform Enterprise workspaces) for **remote state**, **RBAC**, and **variable injection**.

## Prerequisites

- Completed **Lab 10** or **Lab 12** (remote state and environment concepts)  
- [HCP Terraform](https://app.terraform.io/signup/account) account (free tier is sufficient)  
- `terraform login` succeeds  
- `aws sts get-caller-identity` succeeds locally (to verify AWS before remote runs)  
- Terraform 1.5+  
- Optional: [HCP Terraform AWS provider credentials setup](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/variables#environment-variables)

---

## Topic explanation

### Workspace architecture

HCP Terraform organizes infrastructure automation in layers:

```text
Organization (e.g. northwind-training)
 └── Project (e.g. audit-platform)
      ├── Variable set: northwind-global          ← shared across workspaces
      ├── Workspace: northwind-audit-dev          ← state + vars + runs
      ├── Workspace: northwind-audit-qa           ← optional second workspace
      └── (optional) VCS repo → triggered plans on PR
```

| Component | Purpose |
|-----------|---------|
| **Organization** | Billing, users, teams, policy sets |
| **Project** | Groups related workspaces; assigns variable sets |
| **Workspace** | One state file, one variable scope, run queue |
| **Run** | Remote `plan` / `apply` executed on HCP agents |
| **Execution mode** | Remote (default), local, or agent pools (Enterprise) |

**State:** Each workspace stores its own state in HCP (no S3 `backend` block in the root module when using the `cloud` block—HCP is the backend).

**Run workflow (typical):**

```text
Developer: terraform plan (local CLI) → uploads config → HCP runs plan
Reviewer: approve in UI or via VCS check
terraform apply → HCP applies using workspace variables
```

Compare to **Lab 12**: four folders under `environments/` map cleanly to **four workspaces** with the same root module and different workspace variables.

### Variables (Terraform variables)

**Terraform variables** become `var.<name>` in configuration. In HCP, set them per workspace or via a **variable set**.

| Setting | Effect |
|---------|--------|
| **Category: Terraform variable** | Injected as `TF_VAR_name` internally for runs |
| **HCL** | Values parsed as HCL (`true`, `["a"]`, maps) |
| **String** | Literal string only |
| **Description** | Documents intent for operators |

**Priority (simplified):** workspace-specific Terraform variables override values from a variable set when the same key is defined in both (last-writer rules are documented in HashiCorp docs—avoid duplicate keys across sets).

**In code** (this lab):

```hcl
variable "environment" { type = string }
variable "unique_suffix" { type = string }
variable "webhook_token" {
  type      = string
  sensitive = true
}
```

Do **not** commit `terraform.tfvars` with production values; use the workspace UI, API, or `tfe` provider for persistence.

### Variable sets

A **variable set** is a reusable bundle of variables (Terraform and/or environment) attached to one or more workspaces or an entire **project**.

| Use case | Example variables |
|----------|-------------------|
| Org-wide tags | `cost_center`, `owner`, `application` |
| Region defaults | `aws_region` = `us-east-1` |
| Feature flags | `enable_versioning` = `true` |

**This lab:** create **`northwind-global`** with `cost_center`, `owner`, `application`, `aws_region` and assign it to the **audit-platform** project so **dev** and **qa** workspaces inherit them.

### Sensitive variables

Mark **Terraform variables** or **environment variables** as **Sensitive** in HCP:

- Values are **masked** in the UI and run logs  
- Terraform marks `sensitive = true` outputs and variables to reduce accidental exposure  
- Still treat them as secrets: restrict workspace access with teams and SSO  

**Terraform variable** `webhook_token` in this lab is `sensitive = true` in code **and** marked sensitive in HCP.

**Never** paste sensitive values into PR comments or module README files.

### Environment variables

**Environment variables** affect the **runner process**, not `var.*` directly (unless you use `TF_VAR_`—see below).

| Variable | Category | Sensitive | Purpose |
|----------|----------|-----------|---------|
| `AWS_ACCESS_KEY_ID` | Env | Often yes | AWS provider auth on remote agent |
| `AWS_SECRET_ACCESS_KEY` | Env | Yes | AWS provider auth |
| `AWS_DEFAULT_REGION` | Env | No | Default region for AWS SDK |
| `TF_VAR_unique_suffix` | Env | No | Alternative way to set a Terraform variable |
| `TF_LOG` | Env | No | Debug logging (`INFO`—use briefly) |

**Recommended production pattern:** **OIDC** or **dynamic AWS credentials** instead of long-lived keys; for this lab, use scoped IAM user keys with S3-only permissions.

**`TF_VAR_` prefix:** HCP Terraform variables are preferred; locally, `export TF_VAR_environment=dev` mirrors workspace behavior for `terraform plan`.

---

## Use case / scenario

**Northwind Traders** is adopting **HCP Terraform** for the FinOps audit bucket ([Lab 12](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md)). Platform engineering wants:

1. **One workspace per environment** (`northwind-audit-dev`, `northwind-audit-qa`)  
2. **Variable set** `northwind-global` for owner and cost center  
3. **Sensitive** webhook token as a Terraform variable (not in Git)  
4. **Environment variables** for AWS credentials on remote runs only  

You create `labs/work/lab-13/hcp-audit-workspace/`, connect it to HCP, configure variables, and run a remote **plan/apply** that creates one private S3 bucket in **dev**.

---

## Step-by-step instructions

> **Lab root folder:** **`labs/work/lab-13/hcp-audit-workspace/`** (from repository root).

### Step 1 — Verify local tooling

```powershell
terraform version
aws sts get-caller-identity
terraform login
```

Complete the browser flow for `app.terraform.io`.

### Step 2 — Create HCP organization and project (UI)

In [HCP Terraform](https://app.terraform.io):

1. Create or use an **organization** (e.g. `northwind-training-<yourinitials>`).  
2. Create **project** `audit-platform`.  
3. Note your **organization name** for the `cloud` block.

### Step 3 — Create lab folder and Terraform files

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-13\hcp-audit-workspace" -Force | Out-Null
Set-Location "labs\work\lab-13\hcp-audit-workspace"
```

#### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  cloud {
    organization = "REPLACE_WITH_YOUR_ORG"

    workspaces {
      name = "northwind-audit-dev"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.70"
    }
  }
}
```

Replace `REPLACE_WITH_YOUR_ORG`. You will create workspace **`northwind-audit-dev`** in Step 5.

**Alternative (tag-based workspaces):** after tagging workspaces in the UI with `project:northwind` and `env:dev`:

```hcl
# workspaces {
#   tags = ["project:northwind", "env:dev"]
# }
```

This lab uses an explicit **workspace name** for clarity.

#### `variables.tf`

```hcl
variable "aws_region" {
  type        = string
  description = "AWS region (often from variable set)"
}

variable "environment" {
  type        = string
  description = "Environment name for naming and tags"

  validation {
    condition     = contains(["dev", "qa", "uat", "production"], var.environment)
    error_message = "environment must be dev, qa, uat, or production."
  }
}

variable "application" {
  type        = string
  description = "Application name tag"
}

variable "owner" {
  type        = string
  description = "Owner contact (from variable set)"
}

variable "cost_center" {
  type        = string
  description = "Cost center code (from variable set)"
}

variable "unique_suffix" {
  type        = string
  description = "Globally unique S3 suffix (initials)"
}

variable "webhook_token" {
  type        = string
  description = "Inbound webhook token (sensitive; set only in HCP)"
  sensitive   = true
}

variable "force_destroy" {
  type        = bool
  description = "Allow bucket delete when non-empty"
  default     = true
}
```

#### `providers.tf`

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Application = var.application
      Owner       = var.owner
      CostCenter  = var.cost_center
      ManagedBy   = "terraform"
      Lab         = "13-hcp-workspaces"
    }
  }
}
```

#### `main.tf`

```hcl
locals {
  bucket_name = lower("nwt-finops-${var.environment}-use1-audit-logs-${var.unique_suffix}")

  # Proves sensitive variable is wired; never output the token
  webhook_configured = length(var.webhook_token) > 0
}

resource "aws_s3_bucket" "audit" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy

  tags = {
    Environment        = var.environment
    WebhookConfigured  = local.webhook_configured ? "true" : "false"
  }
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
      sse_algorithm = "AES256"
    }
  }
}
```

#### `outputs.tf`

```hcl
output "bucket_id" {
  description = "Audit bucket name"
  value       = aws_s3_bucket.audit.id
}

output "environment" {
  value = var.environment
}

output "workspace_architecture_note" {
  value = "State for this root module lives in HCP workspace northwind-audit-dev"
}
```

#### `terraform.tfvars.example` (do not commit real `terraform.tfvars`)

```hcl
# For LOCAL plan only when cloud {} is commented out — prefer HCP variables.
aws_region     = "us-east-1"
environment    = "dev"
application    = "northwind-audit"
owner          = "platform-team@contoso.com"
cost_center    = "CC-1042"
unique_suffix  = "yourinitials"
webhook_token  = "local-only-placeholder"
force_destroy  = true
```

#### `.gitignore` (lab folder)

```gitignore
.terraform/
*.tfstate
*.tfstate.*
terraform.tfvars
*.auto.tfvars
.env
```

### Step 4 — Create variable set `northwind-global`

In HCP Terraform → your organization → **Variable sets** → **Create variable set**:

| Key | Category | HCL | Value | Sensitive |
|-----|----------|-----|-------|-----------|
| `aws_region` | Terraform | yes | `"us-east-1"` | no |
| `application` | Terraform | yes | `"northwind-audit"` | no |
| `owner` | Terraform | yes | `"platform-team@contoso.com"` | no |
| `cost_center` | Terraform | yes | `"CC-1042"` | no |

**Assign** the set to project **audit-platform** (applies to all workspaces in that project).

### Step 5 — Create workspace `northwind-audit-dev`

**Workspaces** → **audit-platform** → **Create workspace** → **CLI-driven workflow** (or VCS if you prefer).

- Name: **`northwind-audit-dev`**  
- Execution mode: **Remote** (default)  

Confirm the variable set **northwind-global** is inherited (visible under workspace **Variables**).

Add **workspace-specific Terraform variables**:

| Key | HCL | Value | Sensitive |
|-----|-----|-------|-----------|
| `environment` | yes | `"dev"` | no |
| `unique_suffix` | yes | `"yourinitials"` | no |
| `webhook_token` | yes | `"hcp-lab-webhook-token"` | **yes** |
| `force_destroy` | yes | `true` | no |

### Step 6 — Configure environment variables (AWS on remote runs)

In workspace **northwind-audit-dev** → **Variables** → **Environment variables**:

| Key | Value | Sensitive |
|-----|-------|-----------|
| `AWS_ACCESS_KEY_ID` | your access key | **yes** |
| `AWS_SECRET_ACCESS_KEY` | your secret key | **yes** |
| `AWS_DEFAULT_REGION` | `us-east-1` | no |

Use an IAM principal limited to S3 create/delete for this lab.

**Exercise (concept):** add **`TF_VAR_unique_suffix`** as an environment variable and **remove** the workspace Terraform variable of the same name—confirm `plan` still resolves `var.unique_suffix`. Then restore the Terraform variable approach (clearer for operators).

### Step 7 — Initialize and remote plan

```powershell
Set-Location labs\work\lab-13\hcp-audit-workspace
terraform init
terraform plan
```

The CLI uploads configuration; HCP runs **plan** using workspace + variable set values. Open the run in the UI and verify:

- **Variables** tab shows inherited set + workspace overrides (sensitive values masked)  
- Plan creates `aws_s3_bucket.audit` with name containing `dev`  

### Step 8 — Remote apply (dev workspace)

```powershell
terraform apply
```

Confirm in the UI run log that **webhook_token** is not printed. Note **state** is stored under the workspace, not local `terraform.tfstate`.

### Step 9 — Optional second workspace (QA architecture)

Create **`northwind-audit-qa`** in the same project:

1. Duplicate workspace-specific variables: `environment` = `"qa"`, `force_destroy` = `false`, new `unique_suffix` if desired.  
2. Change `versions.tf` workspace name to `northwind-audit-qa` **or** use **tags** and separate local folders.  
3. Run **`terraform plan` only** (no apply) to show how the **same code** + **different workspace variables** changes the bucket name.

Document in your notes how this maps to [Lab 12](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md) `environments/qa/`.

### Step 10 — Architecture worksheet

Fill in this table from your HCP org:

| Layer | Your value |
|-------|------------|
| Organization | |
| Project | audit-platform |
| Variable set | northwind-global |
| Workspace (dev) | northwind-audit-dev |
| Terraform variables (from set) | |
| Workspace-only Terraform variables | |
| Sensitive Terraform variables | |
| Environment variables | |

### Step 11 — Local mirror (sensitive + env vars)

Without HCP (troubleshooting only), comment out the `cloud` block and run:

```powershell
$env:TF_VAR_environment = "dev"
$env:TF_VAR_webhook_token = "local-test"
$env:AWS_DEFAULT_REGION = "us-east-1"
terraform plan
```

Compare with workspace injection—same `var.*`, different delivery mechanism.

---

## Cleanup

```powershell
terraform destroy
```

In HCP, you may **delete** workspace `northwind-audit-dev` after destroy if you do not need the history.

Remove **environment variables** (AWS keys) from the workspace when the lab ends.

Delete `labs/work/lab-13/` locally when finished.

---

## Lab completion checklist

- [ ] Understand **organization → project → workspace** architecture  
- [ ] Root module uses **`cloud`** block; state lives in HCP  
- [ ] **Variable set** assigned to project; shared keys not duplicated in Git  
- [ ] **Workspace variables** set per environment (`environment`, `unique_suffix`)  
- [ ] **Sensitive** Terraform variable configured and masked in UI/logs  
- [ ] **Environment variables** supply AWS credentials for remote runs  
- [ ] Remote **plan** and **apply** succeeded for dev  
- [ ] Optional: second workspace (qa) planned with different variables  

## Self-review questions

| Topic | Question |
|-------|----------|
| Workspace architecture | How does one HCP workspace relate to one state file? |
| Variables | When should a value be HCL vs string in the UI? |
| Variable sets | What belongs in a set vs workspace-specific variables? |
| Sensitive variables | Why mark sensitive in both HCP and `variable { sensitive = true }`? |
| Environment variables | Why are AWS keys env vars instead of Terraform variables? |
| vs Lab 10 | When would you choose HCP vs S3+DynamoDB backend? |
| vs CLI workspaces | What breaks if you confuse `terraform workspace` with HCP workspaces? |

## Previous lab

[Lab 12 — Enterprise Best Practices](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md).

## Next lab

[Lab 14 — HCP Terraform Remote Execution](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md): remote runs, speculative plans, apply approval, cost estimation, and run history.