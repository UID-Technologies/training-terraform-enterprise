# Lab 19 — Terraform Security and Secrets Management with AWS Secrets Manager

## Description

You apply **Terraform security** practices for **secrets**: avoid hardcoded passwords, use **AWS Secrets Manager**, enforce **least-privilege IAM**, understand **Terraform state** sensitivity, and optionally verify secret **metadata** from **GitHub Actions** with **OIDC** (no long-lived AWS keys). Terraform **provisions** the secret container and IAM; the **secret value** is stored outside Terraform (console/CLI) so it does not enter state.

**Duration:** 90–120 minutes | **Level:** intermediate

Relates to [Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md) (secure state), [Lab 12](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md) (secrets patterns), and optional [Lab 18](./Lab%2018%20-%20Terraform%20CI%20CD%20with%20GitHub%20Actions%20and%20Terrakube.md) (CI/CD).

## Prerequisites

- Completed [Lab 06](./Lab%2006%20-%20VPC,%20Subnets,%20Internet%20Gateway,%20Route%20Tables,%20Security%20Groups,%20Elastic%20IP%20(AWS).md) or later AWS lab  
- `aws sts get-caller-identity` succeeds  
- IAM permissions: `secretsmanager:*` (lab scope), `iam:CreatePolicy`, `iam:GetPolicy`  
- Terraform 1.5+; AWS CLI v2  
- Unique **`lab_suffix`** (initials) per participant  
- Optional **Lab 18**: GitHub repo for OIDC workflow in Step 16  

Avoid **root** or long-lived access keys in shared repos.

---

## Topic explanation

### What is Terraform security?

Terraform security protects **configuration**, **credentials**, **state**, and **execution environments**.

**Anti-pattern:**

```hcl
resource "aws_db_instance" "database" {
  username = "admin"
  password = "MyPassword@123" # NEVER in source control
}
```

### Common risks and mitigations

| Risk | Example | Mitigation |
|------|---------|------------|
| Hardcoded secrets | Password in `main.tf` | Secrets Manager; runtime retrieval |
| Keys in Git | `AWS_ACCESS_KEY_ID` in repo | OIDC / IAM roles ([Lab 08](./Lab%2008%20-%20IAM%20Roles,%20Instance%20Profile,%20and%20EC2%20(AWS).md)) |
| Exposed state | Public S3 bucket | Encrypted remote backend ([Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md)) |
| Excessive IAM | `AdministratorAccess` | Least privilege |
| Unreviewed apply | Direct prod apply | CI/CD + approval ([Lab 14](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md), Lab 18) |
| Logs | Printing secrets | `sensitive = true`; never `get-secret-value` in CI for demos |

### AWS Secrets Manager + Terraform

| Operation | Terraform role | State concern |
|-----------|----------------|---------------|
| **A — Secret container** | `aws_secretsmanager_secret` | Metadata (ARN, name)—OK |
| **B — Secret value** | `aws_secretsmanager_secret_version` | **Value may appear in state** |

**Preferred pattern (this lab):** Terraform creates the **secret** and **IAM**; operators or applications set/read the **value** outside Terraform (console, CLI, SDK at runtime).

### `sensitive = true` vs state

| Technique | Hides CLI output | Keeps value out of state |
|-----------|------------------|-------------------------|
| `sensitive = true` variable | Yes | **No**—often still in state |
| Secret metadata only in Terraform | N/A | Yes for **value** |
| App reads at runtime | N/A | Yes if Terraform never holds value |

---

## Use case / scenario

**Contoso** onboards a database whose credentials must not live in Git. Security requires:

1. Secret **container** managed by Terraform  
2. **Fictional** credentials stored only in Secrets Manager  
3. **Read-only IAM policy** scoped to one secret ARN  
4. Awareness of **state** contents  
5. Optional: GitHub Action that calls **`DescribeSecret` only** (not `GetSecretValue`)

**Lab root:** `labs/work/lab-19/` from repository root.

---

## Step-by-step instructions

> **Lab root folder:** **`labs/work/lab-19/`**

### Step 1 — Verify AWS and set region

```powershell
terraform version
aws --version
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
```

### Step 2 — Create project files

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-19" -Force | Out-Null
Set-Location "labs\work\lab-19"
```

```text
labs/work/lab-19/
  providers.tf
  variables.tf
  main.tf
  outputs.tf
  terraform.tfvars.example
  .gitignore
```

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
}
```

#### `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region for the lab"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "lab_suffix" {
  description = "Unique suffix per participant (initials)"
  type        = string
}
```

#### `main.tf`

```hcl
resource "aws_secretsmanager_secret" "database" {
  name                    = "terraform-training/${var.lab_suffix}/database"
  description             = "Database credentials for Terraform security lab"
  recovery_window_in_days = 7

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Purpose     = "SecurityTraining"
    Lab         = "19-secrets"
  }
}

data "aws_iam_policy_document" "secret_read" {
  statement {
    sid    = "ReadTrainingSecret"
    effect = "Allow"

    actions = [
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret",
    ]

    resources = [
      aws_secretsmanager_secret.database.arn,
    ]
  }
}

resource "aws_iam_policy" "secret_read" {
  name        = "terraform-secret-read-${var.lab_suffix}"
  description = "Read access to one training secret"
  policy      = data.aws_iam_policy_document.secret_read.json
}
```

| Resource | Purpose |
|----------|---------|
| `aws_secretsmanager_secret` | Secret **container** only |
| `aws_iam_policy` | Least-privilege **read** for that ARN |

#### `outputs.tf`

```hcl
output "secret_arn" {
  description = "ARN of the created secret"
  value       = aws_secretsmanager_secret.database.arn
}

output "secret_name" {
  description = "Name of the created secret"
  value       = aws_secretsmanager_secret.database.name
}

output "secret_read_policy_arn" {
  description = "IAM policy for read-only secret access"
  value       = aws_iam_policy.secret_read.arn
}
```

#### `terraform.tfvars.example`

```hcl
lab_suffix = "yourinitials"
```

#### `.gitignore`

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
crash.log
```

Commit **`.terraform.lock.hcl`** if you version-control the lab (not under `work/` in course repos).

### Step 3 — Initialize and validate

```powershell
terraform init
terraform fmt
terraform validate
```

### Step 4 — Create the secret container

```powershell
terraform plan -var="lab_suffix=yourinitials"
terraform apply -var="lab_suffix=yourinitials"
```

**Expected:** Apply creates the **secret**, **IAM policy**, and policy document (typically **2 managed resources** plus data source evaluation). Plan shows only `aws_secretsmanager_secret` and `aws_iam_policy` in the resource graph.

```powershell
terraform output secret_name
terraform output secret_arn
```

**Checkpoint:** AWS Console → **Secrets Manager** → secret exists (no value yet, or empty).

### Step 5 — Store secret value outside Terraform

**Do not** add `aws_secretsmanager_secret_version` with a real password in Terraform for this exercise.

AWS Console → your secret → **Store a new secret value** (plaintext JSON):

```json
{
  "username": "training_user",
  "password": "DemoOnly-NotForProduction-123!"
}
```

Fictional credentials only.

Alternatively (PowerShell):

```powershell
$secretName = terraform output -raw secret_name
aws secretsmanager put-secret-value `
  --secret-id $secretName `
  --secret-string '{"username":"training_user","password":"DemoOnly-NotForProduction-123!"}' `
  --region us-east-1
```

### Step 6 — Verify with AWS CLI

Metadata:

```powershell
aws secretsmanager describe-secret --secret-id $secretName --region us-east-1
```

Retrieve value (**training only**—output appears in terminal; never log in CI):

```powershell
aws secretsmanager get-secret-value --secret-id $secretName --region us-east-1 --query SecretString --output text
```

**Checkpoint:** Value exists in Secrets Manager; Terraform state has **no** password field from a `secret_version` resource.

### Step 7 — Verify IAM policy

Console → **IAM** → **Policies** → `terraform-secret-read-yourinitials`.

**Note:** Attach this policy to an **EC2/Lambda/ECS role** in production; the lab only **creates** the policy. If the secret uses a customer-managed **KMS** key, add `kms:Decrypt` on that key.

### Step 8 — Inspect Terraform state

```powershell
terraform state list
terraform state show aws_secretsmanager_secret.database
```

Confirm state shows **ARN, name, tags**—not the JSON password (you did not use `aws_secretsmanager_secret_version`).

Discuss: if you had used:

```hcl
# ANTI-PATTERN FOR SECRETS IN STATE — do not apply in class
# resource "aws_secretsmanager_secret_version" "database" {
#   secret_id     = aws_secretsmanager_secret.database.id
#   secret_string = var.database_password
# }
```

…the value could appear in **state** even with `sensitive = true` on the variable.

### Step 9 — Secure remote state (concept)

Production state should use an encrypted, locked backend ([Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md)): S3 encryption, IAM restriction, DynamoDB lock, versioning.

Terrakube/HCP workspace state likewise requires platform RBAC ([Lab 13](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md)).

Treat **every** state file as **confidential**.

---

## Optional — Step 10 — GitHub Actions OIDC (metadata only)

**Goal:** Authenticate to AWS **without** access keys; call **`DescribeSecret` only**.

Prerequisites: GitHub repo, IAM OIDC provider for `token.actions.githubusercontent.com`, IAM role with trust limited to your repo/branch.

### Step 10a — IAM trust (summary)

1. IAM → **Identity providers** → GitHub OIDC.  
2. Create role with trust `sts:AssumeRoleWithWebIdentity` and condition:

```json
"StringEquals": {
  "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
},
"StringLike": {
  "token.actions.githubusercontent.com:sub": "repo:YOUR-ORG/YOUR-REPO:ref:refs/heads/main"
}
```

3. Attach policy allowing **`secretsmanager:DescribeSecret`** on your secret ARN only (not `GetSecretValue` for this workflow).

### Step 10b — Workflow

Add **`.github/workflows/aws-secret-metadata.yml`**:

```yaml
name: AWS Secret Metadata Check

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  verify-secret:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Verify AWS Identity
        run: aws sts get-caller-identity

      - name: Check Secret Metadata
        run: |
          aws secretsmanager describe-secret \
            --secret-id "${{ vars.SECRET_NAME }}" \
            --query '{Name:Name,ARN:ARN}' \
            --output json
```

| GitHub setting | Value |
|----------------|--------|
| Secret `AWS_ROLE_ARN` | OIDC role ARN |
| Variable `SECRET_NAME` | e.g. `terraform-training/yourinitials/database` |

Pin action versions to commit SHAs in production.

**Checkpoint:** Workflow assumes role and prints **metadata JSON** only.

---

## Cleanup

Secrets Manager may incur charges; destroy lab resources.

```powershell
terraform destroy -var="lab_suffix=yourinitials"
```

Secret enters **7-day recovery** (`recovery_window_in_days = 7`).

```powershell
aws secretsmanager describe-secret --secret-id $secretName --region us-east-1
```

Optional: remove OIDC role/provider if created only for this lab.

Remove `labs/work/lab-19/` locally.

---

## Lab completion checklist

- [ ] `aws sts get-caller-identity` succeeded  
- [ ] `aws_secretsmanager_secret` created via Terraform  
- [ ] Secret **value** set via console/CLI, **not** `secret_version` in Terraform  
- [ ] `describe-secret` and (training-only) `get-secret-value` demonstrated  
- [ ] IAM policy **read-only** on single secret ARN  
- [ ] `terraform state show` reviewed—no password in state  
- [ ] Understand `sensitive = true` vs state  
- [ ] Optional: GitHub Actions OIDC **DescribeSecret** workflow  
- [ ] `terraform destroy` completed  

## Self-review questions

| Topic | Question |
|-------|----------|
| Container vs value | What does `aws_secretsmanager_secret` create without a version resource? |
| State | When does a password enter Terraform state? |
| IAM | Why not attach `AdministratorAccess` to the app role? |
| CLI | Why avoid `get-secret-value` in GitHub Actions logs? |
| Runtime | Who should call `GetSecretValue` in production—the app or Terraform? |
| Lab 12 | How does SSM Parameter Store compare to Secrets Manager here? |

## Instructor takeaway

**Next exercise:** Lambda with an IAM role using `secret_read` policy, reading credentials at **runtime**—password never in Terraform or image.

## Previous lab

[Lab 18 — Terraform CI/CD with GitHub Actions and Terrakube](./Lab%2018%20-%20Terraform%20CI%20CD%20with%20GitHub%20Actions%20and%20Terrakube.md).

## Reference documentation

- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)  
- [Terraform `aws_secretsmanager_secret`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret)  
- [GitHub OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)  
- [Terraform sensitive values](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)