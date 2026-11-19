# Lab 09 — Terraform Modules: Local, Registry, Versioning, Private Modules, and Design Standards (AWS)

## Description

You move from single-root configuration (Labs 01–08) to **reusable Terraform modules** on **AWS**. You build a **local module**, consume a **public Terraform Registry** module, practice **version pinning**, learn **private module** sources, and apply a **module design standards** checklist. The apply path creates one **S3 bucket** (low cost, easy destroy with `force_destroy`).

## Prerequisites

- Completed [Lab 06](./Lab%2006%20-%20VPC,%20Subnets,%20Internet%20Gateway,%20Route%20Tables,%20Security%20Groups,%20Elastic%20IP%20(AWS).md) (or Lab 07/08) and [Lab 01](./Lab%2001%20-%20Create%20an%20Azure%20Resource%20Group%20with%20Terraform.md)-level Terraform comfort  
- `aws sts get-caller-identity` succeeds  
- Permission to create/delete S3 buckets in your account  
- Terraform 1.5+ with network access to [registry.terraform.io](https://registry.terraform.io)

---

## Topic explanation

### What is a Terraform module?

A **module** is a container for multiple resources used together. The **root module** is your `labs/work/lab-09/` directory; **child modules** are called with `module` blocks.

Benefits:

- **Reuse** — One definition for many teams or environments  
- **Abstraction** — Callers pass inputs; implementation stays inside the module  
- **Consistency** — Shared tagging and naming patterns  

### Local modules

| Item | Detail |
|------|--------|
| **Source** | Relative path, e.g. `source = "./modules/s3-bucket"` |
| **When** | Inner-source, fast iteration, module development on your machine |
| **Versioning** | No `version` argument on local paths; promote via **Git tags** or registry |

On `terraform init`, Terraform installs child modules under `.terraform/modules`.

### Registry modules (public)

| Item | Detail |
|------|--------|
| **Source** | `source = "namespace/name/provider"` (public [Terraform Registry](https://registry.terraform.io/browse/modules)) |
| **When** | Community patterns (labeling, VPC, S3, etc.) |
| **`version`** | Pin semver in production, e.g. `version = "0.25.3"` |

This lab uses **[cloudposse/label/null](https://registry.terraform.io/modules/cloudposse/label/null)**—a widely used **naming and tagging** module (no AWS resources by itself).

### Module versioning

| Mechanism | Example | Typical use |
|-----------|---------|-------------|
| **Registry semver** | `version = "1.2.0"` | Public or private Terraform Registry |
| **Git ref** | `source = "git::https://github.com/org/mod.git?ref=v1.2.0"` | Private modules on GitHub/GitLab |
| **Git subdirectory** | `...git//modules/s3-bucket?ref=v2.0.0` | Monorepos |
| **Local path** | `source = "./modules/s3-bucket"` | Development; share via Git/registry for real use |

After changing `source` or `version`, run **`terraform init -upgrade`** when you intend to bump versions.

### Private modules

Private modules are not published on the public registry. Common patterns:

1. **Private Terraform Registry** (HCP Terraform / Terraform Enterprise) — `source = "app.terraform.io/org/name/aws"` + `version`  
2. **Git HTTPS/SSH** — `git::ssh://git@github.com/org/terraform-aws-modules.git//modules/s3-bucket?ref=v1.0.0`  
3. **S3 artifact** — zipped module in a bucket (advanced; less common in labs)

Use IAM, Git tokens, or registry tokens for access—never commit secrets in Terraform source strings.

### Module design principles

1. **Single responsibility** — e.g. “standard lab S3 bucket,” not “entire landing zone” in v1  
2. **Clear contract** — Document required/optional variables and outputs  
3. **No hard-coded account/region** — Pass `aws_region`, name context, tags as variables  
4. **`versions.tf`** in the child module — Pin `aws` provider constraints  
5. **README** in the module — Inputs, outputs, example `module` block  
6. **Semantic versioning** — Breaking input renames = major version bump  

### Module standards (checklist for this course)

| Standard | Expectation |
|----------|-------------|
| Layout | `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf` |
| Naming | `snake_case` variables; consistent resource naming |
| Tags | `Environment`, `ManagedBy`, `Module` via registry label + merges |
| State | One state at root; child modules are embedded in parent state |
| Testing | `terraform validate` from **root** after module changes |

---

## Use case / scenario

**Adventure Works Platform** on AWS requires:

1. A **local module** `s3-bucket` that enforces `force_destroy` for lab sandboxes and standard tags  
2. **[cloudposse/label/null](https://registry.terraform.io/modules/cloudposse/label/null)** from the **public registry** for consistent **IDs and tags** across accounts  
3. All shared modules **version-pinned**; production copies live in **private Git** with `ref=vX.Y.Z`

Your **root module** in `labs/work/lab-09/` combines the registry **label** module with the **local S3** module to provision one bucket.

---

## Step-by-step instructions

> **Lab root folder:** From **repository root**, use **`labs/work/lab-09/`** and **`labs/work/lab-09/modules/s3-bucket/`**.

### Step 1 — Verify AWS access

```powershell
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
```

### Step 2 — Create lab root and module folders

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-09\modules\s3-bucket" -Force | Out-Null
Set-Location "labs\work\lab-09"
```

```text
labs/work/lab-09/
  providers.tf
  variables.tf
  main.tf
  outputs.tf
  terraform.tfvars
  modules/
    s3-bucket/
      versions.tf
      variables.tf
      main.tf
      outputs.tf
      README.md
```

### Step 3 — Create the local child module (`modules/s3-bucket`)

#### `modules/s3-bucket/versions.tf`

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

#### `modules/s3-bucket/variables.tf`

```hcl
variable "bucket_name" {
  type        = string
  description = "Globally unique S3 bucket name"
}

variable "tags" {
  type        = map(string)
  description = "Tags applied to the bucket"
  default     = {}
}

variable "force_destroy" {
  type        = bool
  description = "Allow bucket delete when non-empty (labs only)"
  default     = true
}
```

#### `modules/s3-bucket/main.tf`

```hcl
resource "aws_s3_bucket" "this" {
  bucket        = var.bucket_name
  force_destroy = var.force_destroy
  tags          = var.tags
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

#### `modules/s3-bucket/outputs.tf`

```hcl
output "bucket_id" {
  description = "S3 bucket name (ID)"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "S3 bucket ARN"
  value       = aws_s3_bucket.this.arn
}

output "bucket_region" {
  description = "Bucket region"
  value       = aws_s3_bucket.this.region
}
```

#### `modules/s3-bucket/README.md` (module standards)

```markdown
# s3-bucket module

Creates one private-by-default S3 bucket with standard inputs.

## Inputs

| Name | Required | Description |
|------|----------|-------------|
| bucket_name | yes | Globally unique bucket name |
| tags | no | Tag map |
| force_destroy | no | Allow non-empty delete (default true for labs) |

## Outputs

| Name | Description |
|------|-------------|
| bucket_id | Bucket name |
| bucket_arn | Bucket ARN |
| bucket_region | AWS region |

## Example

module "bucket" {
  source       = "./modules/s3-bucket"
  bucket_name  = "my-unique-bucket-name"
  tags         = { Environment = "lab" }
  force_destroy = true
}
```

### Step 4 — Create the root module (registry label + local S3)

#### `providers.tf` (root)

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
      lab        = "09-modules"
    }
  }
}
```

#### `variables.tf` (root)

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "namespace" {
  type    = string
  default = "fab"
}

variable "stage" {
  type    = string
  default = "lab"
}

variable "name" {
  type    = string
  default = "mod"
}

variable "attributes" {
  type    = list(string)
  default = ["09", "artifacts"]
}
```

#### `main.tf` (root)

```hcl
# Registry module — Module Versioning (pin exact semver)
module "label" {
  source  = "cloudposse/label/null"
  version = "0.25.3"

  namespace  = var.namespace
  stage      = var.stage
  name       = var.name
  attributes = var.attributes
}

# Local module — Local Modules + Module Design
module "s3_bucket" {
  source = "./modules/s3-bucket"

  bucket_name   = module.label.id
  tags          = module.label.tags
  force_destroy = true
}
```

#### `outputs.tf` (root)

```hcl
output "bucket_name" {
  value = module.s3_bucket.bucket_id
}

output "bucket_arn" {
  value = module.s3_bucket.bucket_arn
}

output "label_id" {
  description = "ID string from cloudposse/label (used as bucket name)"
  value       = module.label.id
}

output "label_tags" {
  description = "Standard tags from the registry label module"
  value       = module.label.tags
}
```

#### `terraform.tfvars`

```hcl
aws_region  = "us-east-1"
namespace   = "fab"
stage       = "lab"
name        = "mod"
attributes  = ["09", "yourinitials"]
```

The **label** module builds a lowercase, hyphenated **unique id** suitable for S3 bucket naming (verify in plan if your initials make the name valid—S3 names must be 3–63 characters, lowercase).

### Step 5 — Private module examples (conceptual)

Add to `private-module.example.tf` (or comment in `main.tf`):

```hcl
# Private Git module (Private Modules + Module Versioning)
# module "s3_private" {
#   source = "git::https://github.com/your-org/terraform-aws-s3-bucket.git?ref=v1.0.0"
#
#   bucket_name   = "do-not-use-in-lab"
#   tags          = module.label.tags
#   force_destroy = true
# }

# Git monorepo subdirectory + SSH
# source = "git::ssh://git@github.com/your-org/terraform-modules.git//modules/s3-bucket?ref=v2.1.0"

# Private Terraform Registry
# module "s3_hcp" {
#   source  = "app.terraform.io/your-org/s3-bucket/aws"
#   version = "1.0.0"
#   ...
# }
```

### Step 6 — Initialize and inspect modules

```powershell
terraform init
```

Expect downloads for provider **aws**, registry module **label**, and local **./modules/s3-bucket**.

```powershell
terraform modules
```

You should see `root`, `module.label`, and `module.s3_bucket`.

### Step 7 — Plan and apply

```powershell
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

**Plan review:**

- `module.label` — no AWS resources (null label module).  
- `module.s3_bucket` — creates `aws_s3_bucket` and public access block.

If bucket name is **already taken** globally, adjust `attributes` in `terraform.tfvars` and re-plan.

### Step 8 — Verify

```powershell
terraform output
aws s3api head-bucket --bucket (terraform output -raw bucket_name)
aws s3 ls "s3://$(terraform output -raw bucket_name)"
```

### Step 9 — Practice module versioning

1. Open [cloudposse/label/null](https://registry.terraform.io/modules/cloudposse/label/null/latest) and note another published version (or keep `0.25.3`).  
2. Change `version` in `main.tf` and run:

```powershell
terraform init -upgrade
terraform plan
```

3. Discuss when to commit **`.terraform.lock.hcl`** for reproducible CI.

### Step 10 — Optional module standards exercise

1. Add `variable "extra_tags"` to the child module; in `main.tf` use `tags = merge(module.label.tags, var.extra_tags)` passed from root.  
2. Update the module **README** inputs table.  
3. Run `terraform validate` from the **root** only.

---

## Cleanup

```powershell
terraform destroy
```

Confirm `yes`. Empty the bucket first if destroy complains (or rely on `force_destroy = true`).

```powershell
aws s3 rb "s3://$(terraform output -raw bucket_name)" --force
```

---

## Lab completion checklist

- [ ] Local module `modules/s3-bucket/` with `versions.tf`  
- [ ] Registry module `cloudposse/label/null` with **pinned `version`**  
- [ ] `terraform init` installs local + registry modules  
- [ ] S3 bucket created with **label-generated name and tags**  
- [ ] Private module `git::` / HCP registry examples documented  
- [ ] Child module README lists inputs/outputs  
- [ ] `terraform destroy` succeeded  

## Module standards self-review

| Question | Your answer |
|----------|-------------|
| Are all required variables documented in the child README? | |
| Are outputs minimal (id, arn, region)? | |
| Is the AWS provider constrained inside the child module? | |
| Could another team consume `s3-bucket` with three arguments? | |

## Next steps (after the course)

- Extract Lab 06–08 networking into a **private Git module** with semver tags  
- Publish modules to **HCP Terraform** private registry  
- Add **policy-as-code** (OPA/Sentinel) for required module inputs  

## Next lab

[Lab 10 — Remote State with S3 Backend (AWS)](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md).

## Course wrap-up

You completed Labs 01–09: Azure and AWS infrastructure plus **AWS-based Terraform modules**. Continue with **remote state** in Lab 10.