# Lab 20 — Terrakube Private Module Registry, GitHub, and Private S3 Bucket (AWS)

## Description

You build a **reusable private S3 module** in **GitHub**, **publish** it to the **Terrakube private module registry**, create a **new Terrakube workspace** connected to GitHub, inject **AWS credentials** securely for the **executor**, and run **remote plan/apply** to create a **private** S3 bucket in AWS.

**Duration:** 120–150 minutes | **Level:** intermediate

Builds on [Lab 09](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md) (module design), [Lab 15–17](./Lab%2017%20-%20Terrakube%20Workspaces%20and%20Remote%20Terraform%20Execution.md) (Terrakube), [Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md) (GitHub VCS), and [Lab 19](./Lab%2019%20-%20Terraform%20Security%20and%20AWS%20Secrets%20Manager.md) (secrets discipline).

## Prerequisites

- Terrakube running (**https://terrakube-ui.minikube.net** or your install); registry host e.g. **https://terrakube-reg.minikube.net**  
- GitHub **VCS** connected ([Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md))  
- `aws sts get-caller-identity` succeeds locally (proves your AWS account works)  
- IAM: **S3** create/delete/list; keys used only in Terrakube workspace settings—not in Git  
- Terraform 1.5+

---

## Topic explanation

### Architecture

```text
GitHub: terrakube-aws-s3-platform
  ├── modules/s3-private-bucket/     → published to Terrakube Registry (semver tag)
  └── workloads/s3-dev/                → root module consumed by workspace

Terrakube
  ├── Registry (private): ORG/s3-private-bucket/aws @ 1.0.0
  └── Workspace: s3-private-dev (VCS → workloads/s3-dev)
         ├── Environment vars: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (sensitive)
         └── Terraform vars: bucket_name, aws_region
               │
               ▼
         Executor → AWS API → private S3 bucket
```

### Where AWS access key and secret belong

| Store here | OK for lab? | Notes |
|------------|-------------|--------|
| **Terrakube workspace → Environment variables** | **Yes (this lab)** | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`; mark **Sensitive** |
| **Terrakube variable set** (org/project) | Yes | Shared non-production keys only; still sensitive |
| **Git / `main.tf` / `terraform.tfvars` in repo** | **Never** | Will leak in VCS history |
| **Terraform variables in Git** | **Never** for keys | Use for `bucket_name`, `environment`, not credentials |
| **AWS Secrets Manager** ([Lab 19](./Lab%2019%20-%20Terraform%20Security%20and%20AWS%20Secrets%20Manager.md)) | For **app** secrets | DB passwords, API tokens—not the usual pattern for Terraform **executor** static keys |
| **IAM role for executor** (EKS/IRSA, instance profile) | **Production best** | Replace long-lived keys when executor runs in AWS |

**Rule:** The **Terrakube executor** needs AWS credentials in its **runtime environment** (environment variables or cloud IAM). **Participants** configure them only in the Terrakube UI, never in the GitHub repository.

### Private module registry (Terrakube)

1. Module code lives in **Git** under `modules/s3-private-bucket/`.  
2. **Semver tag** on the repo (e.g. `v1.0.0`) defines a registry version ([Publishing private modules](https://docs.terrakube.io/private-registry/private-registry/publishing-private-modules)).  
3. **Registry → Publish module** links the Git repo and **folder** `/modules/s3-private-bucket/`.  
4. Root code references:

```hcl
module "bucket" {
  source  = "terrakube-reg.minikube.net/YOUR_ORG/s3-private-bucket/aws"
  version = "1.0.0"
  # ...
}
```

Replace `terrakube-reg.minikube.net` and `YOUR_ORG` with your Terrakube registry hostname and organization name from the **Registry** UI module page.

The executor resolves the registry over HTTPS; local `terraform login terrakube-reg.minikube.net` matters for **CLI on your laptop**, not for typical **remote** workspace runs (Terrakube handles registry auth internally).

---

## Use case / scenario

**Northwind Platform** ships a **private S3 bucket module** on the **Terrakube private registry**. Application teams deploy via workspace **`s3-private-dev`** and **central AWS credentials**—no keys in Git.

**Local mirror (optional):** `labs/work/lab-20/terrakube-aws-s3-platform/` before pushing to GitHub.

---

## Step-by-step instructions

> **From repository root**, use paths below relative to that root.

### Step 1 — Verify AWS and Terrakube

```powershell
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
kubectl get pods -n terrakube
```

Open **https://terrakube-ui.minikube.net** and confirm GitHub VCS ([Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md)).

### Step 2 — Create GitHub repository

Create **`terrakube-aws-s3-platform`** (public or private with VCS/SSH).

```powershell
gh repo create terrakube-aws-s3-platform --private --description "Terrakube private S3 module lab"
```

### Step 3 — Scaffold repository layout locally

```powershell
$base = "labs\work\lab-20\terrakube-aws-s3-platform"
New-Item -ItemType Directory -Path "$base\modules\s3-private-bucket" -Force | Out-Null
New-Item -ItemType Directory -Path "$base\workloads\s3-dev" -Force | Out-Null
Set-Location $base
```

```text
terrakube-aws-s3-platform/
  modules/s3-private-bucket/
  workloads/s3-dev/
  .gitignore
```

Root **`.gitignore`**:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
```

After `terraform init` in `workloads/s3-dev`, **commit** `.terraform.lock.hcl` for consistent provider versions on the executor.

### Step 4 — Author module `modules/s3-private-bucket`

#### `modules/s3-private-bucket/versions.tf`

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

#### `modules/s3-private-bucket/variables.tf`

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
  description = "Allow non-empty delete (labs only)"
  default     = true
}
```

#### `modules/s3-private-bucket/main.tf`

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

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}
```

#### `modules/s3-private-bucket/outputs.tf`

```hcl
output "bucket_id" {
  value = aws_s3_bucket.this.id
}

output "bucket_arn" {
  value = aws_s3_bucket.this.arn
}
```

#### `modules/s3-private-bucket/README.md`

```markdown
# s3-private-bucket

Private-by-default S3 bucket: public access block, SSE-S3, ownership controls.

## Inputs

bucket_name (required), tags, force_destroy
```

Commit and push **module only** first:

```powershell
git init
git add .
git commit -m "Add s3-private-bucket module"
git branch
git remote add origin https://github.com/YOUR-USERNAME/terrakube-aws-s3-platform.git
git push origin main
```

### Step 5 — Publish module to Terrakube private registry

1. Terrakube UI → **Registry** → **Publish module**.  
2. Select GitHub **VCS**; repository **`terrakube-aws-s3-platform`**.  
3. Set **module folder** to `/modules/s3-private-bucket/` (leading and trailing slashes per Terrakube docs).  
4. **Provider:** `aws` | **Name:** `s3-private-bucket` | **Organization:** your training org.  
5. **Publish**.

Create and push a **semver** tag:

```powershell
git tag v1.0.0
git push origin v1.0.0
```

Wait for registry refresh (default up to ~5–10 minutes; see [Terrakube module refresh](https://docs.terrakube.io/2.34.0/private-registry/private-registry)).

**Checkpoint:** Registry shows version **1.0.0** (or **v1.0.0** per UI). Copy the **module source** string from the UI.

Example shape:

```hcl
source  = "terrakube-reg.minikube.net/northwind/s3-private-bucket/aws"
version = "1.0.0"
```

### Step 6 — Add workload root `workloads/s3-dev`

Create **`workloads/s3-dev/versions.tf`**:

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
```

**`workloads/s3-dev/variables.tf`**:

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "bucket_name" {
  type        = string
  description = "Globally unique bucket name"
}

variable "environment" {
  type    = string
  default = "dev"
}
```

**`workloads/s3-dev/providers.tf`**:

```hcl
provider "aws" {
  region = var.aws_region
}
```

**`workloads/s3-dev/main.tf`** (replace registry host and org from Step 5):

```hcl
module "private_bucket" {
  source  = "terrakube-reg.minikube.net/YOUR_ORG/s3-private-bucket/aws"
  version = "1.0.0"

  bucket_name = var.bucket_name

  tags = {
    Environment = var.environment
    ManagedBy   = "terrakube"
    Lab         = "20-private-s3"
  }

  force_destroy = true
}
```

**`workloads/s3-dev/outputs.tf`**:

```hcl
output "bucket_id" {
  value = module.private_bucket.bucket_id
}

output "bucket_arn" {
  value = module.private_bucket.bucket_arn
}
```

**`workloads/s3-dev/terraform.tfvars.example`** (not committed with real names):

```hcl
bucket_name = "nwt-terrakube-dev-YOURINITIALS"
```

```powershell
git add workloads
git commit -m "Add s3-dev workload using private registry module"
git push
```

### Step 7 — Create Terrakube workspace `s3-private-dev`

**Workspaces → New → Version Control**

| Setting | Value |
|---------|--------|
| Name | `s3-private-dev` |
| VCS | GitHub (Lab 16) |
| Repository | `terrakube-aws-s3-platform` |
| Branch | `main` |
| **Working directory** | `workloads/s3-dev` |
| Terraform version | `>= 1.5.0` compatible |

Save the workspace.

### Step 8 — Configure AWS credentials (workspace only)

Open **`s3-private-dev` → Variables**.

**Environment variables** (category **Environment**, all **Sensitive** where offered):

| Name | Value |
|------|--------|
| `AWS_ACCESS_KEY_ID` | Your lab IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | Your lab IAM user secret key |
| `AWS_DEFAULT_REGION` | `us-east-1` |

**Terraform variables** (category **Terraform**, HCL):

| Name | Example |
|------|---------|
| `bucket_name` | `"nwt-terrakube-dev-yourinitials"` |
| `aws_region` | `"us-east-1"` |
| `environment` | `"dev"` |

```text
NEVER add AWS_ACCESS_KEY_ID to Terraform variables or main.tf.
NEVER commit terraform.tfvars with keys to GitHub.
```

Optional: attach a **variable set** at project level for `aws_region` and `environment`; keep **keys workspace-specific**.

**Production note:** Use an IAM user/role limited to `s3:*` on a prefix, or OIDC—not AdministratorAccess.

### Step 9 — Remote plan

**Jobs → Plan** on workspace **`s3-private-dev`**.

Confirm in logs:

- Module download from **private registry**  
- Plan: create `module.private_bucket.aws_s3_bucket.this` and related resources  

If **registry auth** fails on the executor, verify registry TLS hostname matches Helm `values.yaml` and that the module version exists.

### Step 10 — Remote apply and verify AWS

Run **Apply** (approve if required).

**Expected:** S3 bucket created.

```powershell
aws s3api get-bucket-versioning --bucket nwt-terrakube-dev-yourinitials
aws s3api get-public-access-block --bucket nwt-terrakube-dev-yourinitials
```

Console → **S3** → bucket is **not** public.

Inspect workspace **State** and **Outputs** (`bucket_id`, `bucket_arn`).

### Step 11 — Release module v1.0.1 (optional)

Change module description in `README.md`, commit, tag:

```powershell
git tag v1.0.1
git push origin v1.0.1
```

Confirm new version in **Registry**; update workload `version = "1.0.1"` in a later change when ready.

### Step 12 — Destroy and cleanup

1. Terrakube: **destroy** run or apply removal of bucket (workspace job).  
2. `terraform destroy` equivalent via Terrakube job template.  
3. Remove AWS keys from workspace **environment variables**.  
4. Delete workspace or keep for reuse.  
5. `git push` delete not required; remove `labs/work/lab-20/` locally.

---

## Troubleshooting

| Issue | Check |
|-------|--------|
| Module version not found | Tag is **semver**; wait for registry refresh |
| 401/403 registry | Org name; module published; HTTPS trust |
| AWS Unauthorized | Workspace env vars; IAM policy for S3 |
| Wrong directory | Working directory `workloads/s3-dev` |
| Plan uses wrong module | `source` / `version` in `main.tf` |
| Keys in GitHub | Rotate keys; remove from history (gitleaks) |

```powershell
kubectl logs -n terrakube -l app=executor --tail=100
```

---

## Lab completion checklist

- [ ] GitHub repo **`terrakube-aws-s3-platform`** with module + workload  
- [ ] Module **published** in Terrakube **private registry** with tag **v1.0.0**  
- [ ] Workspace **`s3-private-dev`**; working dir **`workloads/s3-dev`**  
- [ ] **AWS keys** only in Terrakube **environment variables** (sensitive)  
- [ ] Terraform vars for `bucket_name` / region—**no** secrets in Git  
- [ ] Remote **plan** pulls private module  
- [ ] Remote **apply** creates **private** S3 bucket  
- [ ] AWS console/CLI confirms public access block  
- [ ] **Destroy** completed; keys removed from workspace  

## Self-review questions

| Topic | Question |
|-------|----------|
| Registry | Why tag the Git repo before the registry shows a version? |
| Credentials | Why are AWS keys environment variables, not Terraform variables? |
| Private S3 | Which resources enforce “no public access”? |
| vs Lab 09 | Local path `source = "./modules/..."` vs private registry `source`? |
| vs Lab 19 | Would you store the **executor access key** in Secrets Manager? |
| Production | What replaces static access keys for the executor? |

## Previous lab

[Lab 19 — Terraform Security and AWS Secrets Manager](./Lab%2019%20-%20Terraform%20Security%20and%20AWS%20Secrets%20Manager.md).

## Reference documentation

- [Publishing private modules](https://docs.terrakube.io/private-registry/private-registry/publishing-private-modules)  
- [Using private modules](https://docs.terrakube.io/private-registry/private-registry/using-private-modules)  
- [Terrakube workspace variables](https://docs.terrakube.io/) (environment vs Terraform variables)  
- [AWS S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
