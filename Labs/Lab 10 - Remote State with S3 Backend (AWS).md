# Lab 10 — Remote State: S3 Backend and State Locking (AWS)

## Description

**Module 11 (curriculum): Remote state.** In this lab you implement the **AWS S3 backend** for Terraform state, with **DynamoDB locking** and encryption. You bootstrap the state store, **migrate** from local state to S3, then manage a small workload whose state lives remotely. Hands-on is **AWS only**; Azure Storage backend is summarized for comparison.

## Prerequisites

- Completed [Lab 09](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md) or equivalent `terraform init` / `apply` experience  
- `aws sts get-caller-identity` succeeds  
- Permission to create S3 buckets and DynamoDB tables (IAM, S3, DynamoDB)  
- Terraform 1.5+

---

## Topic explanation

### Why remote state?

| Local `terraform.tfstate` | Remote backend |
|---------------------------|----------------|
| On one laptop | Shared by team and CI/CD |
| Easy to lose or leak | Central, access-controlled store |
| No locking | **State locking** prevents concurrent corrupting runs |

Terraform stores resource mappings in **state**. Remote backends keep that file off laptops and enforce collaboration rules.

### AWS — S3 backend (this lab)

| Setting | Purpose |
|---------|---------|
| `bucket` | S3 bucket holding `terraform.tfstate` object(s) |
| `key` | Object path, e.g. `lab-10/app/terraform.tfstate` |
| `region` | Bucket region |
| `encrypt` | Server-side encryption for state at rest |
| `dynamodb_table` | **Lock** table (conditional writes) |
| `use_lockfile` (1.11+) | Optional S3-native lock alternative; this lab uses **DynamoDB** (common pattern) |

After `terraform init`, state reads/writes go to S3; locks go to DynamoDB during `plan`/`apply`.

### Azure — Storage backend (concept only)

Not exercised in this lab; teams on Azure often use:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstateunique"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

Requires a **pre-created** storage account and container (bootstrap problem is the same as S3: something must exist before the backend can point at it).

### Bootstrap pattern (AWS)

You cannot store state in a bucket that **only exists in the same state file** on first run. Typical approach:

1. **Phase A** — Apply with **local state** to create S3 + DynamoDB (bootstrap resources).  
2. **Phase B** — Add `backend "s3"` block with those names; `terraform init -migrate-state`.  
3. **Phase C** — Continue managing bootstrap + app resources; state object lives in S3.

### State locking flow

```text
terraform plan/apply
  → acquire lock (DynamoDB)
  → read/write state (S3)
  → release lock
```

If a lock is stuck after a crash, use `terraform force-unlock <LOCK_ID>` only when no other run is active.

---

## Use case / scenario

**Litware DevOps** runs Terraform from GitHub Actions and three engineers’ laptops. They need:

- One **encrypted** S3 bucket for state in `us-east-1`  
- A **DynamoDB** table `terraform-locks`  
- State key per app: `lab-10/litware/terraform.tfstate`  
- A sample **SSM Parameter** resource to prove new resources land in remote state  

You implement bootstrap, migrate, then add the parameter.

---

## Step-by-step instructions

> **Lab root folder:** **`labs/work/lab-10/`** (from repository root).

### Step 1 — Verify AWS access

```powershell
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
```

### Step 2 — Create lab root folder

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-10" -Force | Out-Null
Set-Location "labs\work\lab-10"
```

### Phase A — Bootstrap (local state)

Create these files **without** a `backend "s3"` block yet.

#### `providers.tf`

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
      managed_by = "terraform"
      lab        = "10-remote-state"
    }
  }
}
```

#### `variables.tf`

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "state_bucket_name" {
  type        = string
  description = "Globally unique S3 bucket name for Terraform state"
}

variable "dynamodb_table_name" {
  type    = string
  default = "terraform-locks-lab10"
}

variable "environment" {
  type    = string
  default = "lab"
}
```

#### `bootstrap.tf`

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket        = var.state_bucket_name
  force_destroy = true

  tags = {
    Name        = "terraform-state"
    description = "Remote state bucket for lab 10"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = var.dynamodb_table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "terraform-state-locks"
  }
}
```

#### `app.tf` (workload tracked in state)

```hcl
resource "aws_ssm_parameter" "lab_marker" {
  name        = "/lab/10/remote-state/marker"
  type        = "String"
  value       = "remote-state-lab-${var.environment}"
  description = "Lab 10 marker parameter"
  overwrite   = true
}
```

#### `outputs.tf`

```hcl
output "state_bucket_name" {
  value = aws_s3_bucket.terraform_state.id
}

output "dynamodb_table_name" {
  value = aws_dynamodb_table.terraform_locks.name
}

output "ssm_parameter_name" {
  value = aws_ssm_parameter.lab_marker.name
}
```

#### `terraform.tfvars`

```hcl
aws_region          = "us-east-1"
state_bucket_name   = "tfstate-lab10-yourinitials-unique"
dynamodb_table_name = "terraform-locks-lab10-yourinitials"
environment         = "lab"
```

Use a **globally unique** `state_bucket_name` (lowercase, no underscores in bucket name—use hyphens).

#### `bootstrap-apply.ps1` (reference commands)

```powershell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Confirm outputs: bucket name, DynamoDB table, SSM parameter.

### Phase B — Configure S3 backend and migrate

Create **`backend.tf`** (values must match what you created in Phase A):

```hcl
terraform {
  backend "s3" {
    bucket         = "tfstate-lab10-yourinitials-unique" # same as state_bucket_name
    key            = "lab-10/litware/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks-lab10-yourinitials"
    encrypt        = true
  }
}
```

**Important:** Replace bucket and table strings with **your** names from `terraform.tfvars`. Backend blocks do not accept variables.

Re-initialize and **migrate**:

```powershell
terraform init -migrate-state
```

Answer `yes` to copy local state into S3. Local `terraform.tfstate` may remain as backup; S3 is now authoritative.

Verify:

```powershell
aws s3 ls "s3://<your-state-bucket>/lab-10/litware/"
```

You should see `terraform.tfstate`.

### Phase C — Change infrastructure with remote state

1. Update `app.tf` parameter value or add a tag in `default_tags`.  
2. Run:

```powershell
terraform plan
terraform apply
```

3. Open S3 → bucket → object → **Versions** (optional) to see versioning after multiple applies.

4. Confirm locking (optional): start a long `terraform apply` in one window and run `terraform plan` in another—the second should **wait for lock**.

### Step — Inspect state (read-only)

```powershell
terraform state list
terraform state show aws_ssm_parameter.lab_marker
```

---

## Cleanup

**Order matters** when the state bucket holds its own bootstrap resources.

1. If you added other resources, remove them first:

```powershell
terraform destroy
```

2. When only bootstrap remains, you may need to **remove backend** temporarily to destroy bucket (chicken-and-egg), or empty bucket then destroy:

```powershell
aws s3 rm "s3://<your-state-bucket>" --recursive
terraform destroy
```

3. Delete DynamoDB table if it remains:

```powershell
aws dynamodb delete-table --table-name <your-lock-table>
```

4. Remove local artifacts:

```powershell
Remove-Item terraform.tfstate* -ErrorAction SilentlyContinue
Remove-Item .terraform -Recurse -Force -ErrorAction SilentlyContinue
```

---

## Lab completion checklist

- [ ] S3 state bucket: versioning, encryption, public access blocked  
- [ ] DynamoDB lock table created  
- [ ] `terraform init -migrate-state` moved state to S3  
- [ ] `terraform plan`/`apply` work with remote state  
- [ ] Understand Azure `backend "azurerm"` at a high level (not applied in this lab)  
- [ ] Cleanup completed  

## Comparison summary (curriculum Module 11)

| Cloud | Backend type | State blob | Locking |
|-------|--------------|------------|---------|
| **AWS** (lab) | `backend "s3"` | S3 object (`key`) | DynamoDB table |
| **Azure** (concept) | `backend "azurerm"` | Blob in container | Blob lease (native) |

## Next steps

- Separate **bootstrap** stack in its own repo/state  
- Use **IAM** least privilege for CI role (S3 Get/Put, DynamoDB lock items)  
- Consider **Terraform Cloud** or **HCP Terraform** for managed remote state  

## Next lab

[Lab 11 — Git Integration](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md).

## Previous lab

[Lab 09 — Terraform Modules](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md).