# Lab 11 — Git Integration: Branching, Feature Branches, Pull Requests, Repository Structure, and Versioning

## Description

You practice **Git workflows** used with Terraform in enterprises: **branching**, **feature branches**, **pull requests**, a standard **repository layout**, and a **versioning strategy** for modules and releases. This lab is **not a cloud apply lab**—you build a small Terraform repo under Git, run `fmt`/`validate` on a branch, and simulate review/merge. Optional: push to GitHub and open a real PR with `gh`.

## Prerequisites

- Completed [Lab 09](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md) and [Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md) (or equivalent module/state concepts)  
- [Git](https://git-scm.com/downloads) installed (`git --version`)  
- Terraform 1.5+ (`terraform validate` only—no AWS/Azure credentials required for core steps)  
- Optional: [GitHub CLI](https://cli.github.com/) (`gh auth login`) and a personal GitHub repo for a live PR

---

## Topic explanation

### Git integration with Terraform

Infrastructure code is **versioned like application code**. Git provides history, review, rollback, and automation hooks (CI running `terraform plan` on pull requests).

**Never commit:**

- `terraform.tfstate` / `*.tfstate.*`  
- `.terraform/`  
- `terraform.tfvars` with secrets  
- `.env` with cloud keys  

Use `.gitignore` from day one.

### Git branching

| Branch type | Typical name | Purpose |
|-------------|--------------|---------|
| **Default / integration** | `main` or `master` | Production-ready code; protected |
| **Development** | `develop` (optional) | Integration branch before release |
| **Feature** | `feature/<ticket>-<short-name>` | One change set per task |
| **Hotfix** | `hotfix/<issue>` | Urgent fix to production config |
| **Release** | `release/1.2.0` (optional) | Stabilize before tag |

**Trunk-based** teams often use only `main` + short-lived feature branches (this lab uses that model).

### Feature branch workflow

```text
main ──●──────────────●────── merge PR
        \            /
         ●──●──●────●   feature/add-standard-tags
```

1. Branch from `main`  
2. Commit Terraform changes in small, reviewable commits  
3. Open **pull request**  
4. CI runs `terraform fmt -check`, `validate`, and `plan` (in real orgs)  
5. Peer review → merge → delete feature branch  

### Pull requests (PRs)

A **pull request** (GitHub/GitLab/Azure DevOps) is a request to merge one branch into another. For Terraform, the PR description should include:

- **Intent** — what infrastructure changes and why  
- **Plan summary** — paste sanitized `terraform plan` output or link to CI artifact  
- **Risk** — destroys/replacements, blast radius  
- **Rollback** — revert commit or previous tag  

**Review checklist (common):**

- [ ] No secrets in diff  
- [ ] `terraform fmt` applied  
- [ ] Variables documented; defaults safe  
- [ ] Module `version` pins updated intentionally  
- [ ] State backend unchanged without platform approval  

### Repository structure

A layout that scales for teams:

```text
terraform-platform/
├── .gitignore
├── README.md
├── modules/                    # Reusable modules (Lab 09 style)
│   └── s3-bucket/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
├── environments/               # Root modules per environment
│   ├── dev/
│   │   ├── main.tf             # Calls modules, dev-specific vars
│   │   ├── backend.tf          # Remote state (Lab 10 style)
│   │   ├── providers.tf
│   │   └── terraform.tfvars.example
│   └── prod/
│       └── ...
└── .github/                    # Optional: PR template, CI
    └── pull_request_template.md
```

**Principles:**

- **Modules** = reusable building blocks; versioned with **Git tags** or private registry  
- **Environments** = separate roots, separate **state keys** (never share one state file for dev and prod)  
- **Least code in root** — compose modules, pass `environment`, `region`, etc.

### Versioning strategy

| Artifact | Versioning mechanism | Example |
|----------|----------------------|---------|
| **Git commits** | SHA | `a1b2c3d` — audit trail |
| **Releases / modules** | **Semantic versioning** (tags) | `v1.2.0` — `MAJOR.MINOR.PATCH` |
| **Terraform Registry modules** | `version = "1.2.0"` in `module` block | Pins consumer to module release |
| **Environments** | Branch or tag policy | `prod` only deploys from `v*` tags or `main` |

**Semver guidance for Terraform modules:**

- **MAJOR** — breaking variable/output changes  
- **MINOR** — backward-compatible features  
- **PATCH** — fixes only  

Tag after merge to `main`: `git tag -a v1.0.0 -m "Initial s3-bucket module release"`.

---

## Use case / scenario

**Wide World Importers** is moving Terraform from laptops to a **Git platform**. They need:

- A standard **repo structure** (`modules/` + `environments/dev`)  
- All changes via **feature branches** and **PRs**  
- **v1.0.0** tag on the first reusable module release  
- PR template requiring `terraform validate` before merge  

You create `labs/work/lab-11/terraform-platform/`, initialize Git, complete one feature branch cycle, and tag a release.

---

## Step-by-step instructions

> **Lab root folder:** **`labs/work/lab-11/terraform-platform/`** (from repository root). This is your **mini platform repo** for the lab.

### Step 1 — Create repository layout

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-11\terraform-platform\modules\s3-bucket" -Force | Out-Null
New-Item -ItemType Directory -Path "labs\work\lab-11\terraform-platform\environments\dev" -Force | Out-Null
Set-Location "labs\work\lab-11\terraform-platform"
```

### Step 2 — Create `.gitignore`

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraform.lock.hcl

# Secrets
terraform.tfvars
*.auto.tfvars
.env
*.pem

# OS / IDE
.DS_Store
Thumbs.db
.idea/
.vscode/
```

### Step 3 — Create module skeleton (`modules/s3-bucket`)

Minimal module for **validate only** (no provider credentials needed if only structure is validated from env root calling module—actually validate needs full config).

Use a **null** module pattern - or aws provider with validate on module only. Simplest: module with no resources, only `terraform` block - validate might pass from environment.

**`modules/s3-bucket/versions.tf`**

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

**`modules/s3-bucket/variables.tf`**

```hcl
variable "bucket_name" {
  type = string
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

**`modules/s3-bucket/main.tf`**

```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  tags   = var.tags
}
```

**`modules/s3-bucket/outputs.tf`**

```hcl
output "bucket_id" {
  value = aws_s3_bucket.this.id
}
```

### Step 4 — Create `environments/dev` root

**`environments/dev/providers.tf`**

```hcl
terraform {
  required_version = ">= 1.5.0"
}

provider "aws" {
  region = var.aws_region
}
```

**`environments/dev/variables.tf`**

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "bucket_name" {
  type = string
}
```

**`environments/dev/main.tf`**

```hcl
module "artifacts" {
  source = "../../modules/s3-bucket"

  bucket_name = var.bucket_name
  tags = {
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

**`environments/dev/terraform.tfvars.example`**

```hcl
bucket_name = "wwi-dev-artifacts-yourinitials"
```

**`environments/dev/backend.tf`** (commented—Lab 10 pattern)

```hcl
# terraform {
#   backend "s3" {
#     bucket         = "your-state-bucket"
#     key            = "environments/dev/terraform.tfstate"
#     region         = "us-east-1"
#     dynamodb_table = "terraform-locks"
#     encrypt        = true
#   }
# }
```

### Step 5 — Root `README.md` for the platform repo

```markdown
# terraform-platform (lab)

## Layout
- `modules/` — shared modules (tag with semver)
- `environments/dev` — development root module

## Workflow
1. Branch from `main`: `feature/<name>`
2. `terraform fmt` && `terraform validate` in affected environment
3. Open PR → review → merge
4. Tag releases: `v1.0.0`

## CI (recommended)
- PR: fmt check, validate, plan (with cloud credentials in CI only)
```

### Step 6 — Initialize Git on `main`

```powershell
git init
git checkout -b main
git add .
git commit -m "chore: initial repository structure for Terraform platform"
```

### Step 7 — Feature branch: add standard tags

```powershell
git checkout -b feature/add-cost-center-tag
```

Edit `environments/dev/main.tf` module tags:

```hcl
  tags = {
    environment = "dev"
    managed_by  = "terraform"
    cost_center = "CC-1001"
  }
```

Format and validate (no apply required for this lab):

```powershell
Set-Location environments\dev
terraform init -backend=false
terraform fmt -recursive ..\..
terraform validate
Set-Location ..\..
```

Commit:

```powershell
git add environments/dev/main.tf
git commit -m "feat(dev): add cost_center tag to artifacts bucket module call"
```

### Step 8 — Pull request workflow

#### Option A — Local review (no remote)

Simulate a PR with a second terminal or mental checklist:

1. `git diff main...feature/add-cost-center-tag`  
2. Verify only intended `.tf` files changed  
3. Merge locally:

```powershell
git checkout main
git merge --no-ff feature/add-cost-center-tag -m "Merge PR: add cost_center tag"
git branch -d feature/add-cost-center-tag
```

`--no-ff` preserves a merge commit (visible in history like a merged PR).

#### Option B — GitHub PR (optional)

```powershell
gh repo create terraform-platform-lab --private --source=. --push
git push -u origin feature/add-cost-center-tag
gh pr create --base main --head feature/add-cost-center-tag `
  --title "feat(dev): add cost_center tag" `
  --body "## Summary`n- Adds cost_center tag for chargeback`n`n## Terraform`n- [x] fmt`n- [x] validate`n- [ ] plan (run in CI with credentials)"
gh pr merge --merge
```

### Step 9 — PR template (optional file)

Create `.github/pull_request_template.md`:

```markdown
## Summary

<!-- What infrastructure change and why -->

## Terraform checklist

- [ ] `terraform fmt` applied
- [ ] `terraform validate` passed
- [ ] `terraform plan` reviewed (attach or link CI)
- [ ] No secrets in diff
- [ ] State / backend changes approved (if any)

## Risk

<!-- Replacements, destroys, downtime -->
```

Commit on `main` after merge if you add the template in a follow-up branch.

### Step 10 — Versioning strategy: tag module release

After module is stable on `main`:

```powershell
git tag -a v1.0.0 -m "s3-bucket module: initial release"
git tag -l
```

Consumers can reference:

```hcl
module "artifacts" {
  source = "git::https://github.com/org/terraform-platform.git//modules/s3-bucket?ref=v1.0.0"
  # ...
}
```

**Patch example:** fix typo in module README → `v1.0.1`. **Breaking variable rename** → `v2.0.0`.

Document in root `README.md`:

```markdown
## Versioning
| Tag | Change |
|-----|--------|
| v1.0.0 | Initial s3-bucket module |
```

### Step 11 — Branching diagram exercise

Draw or list your repo state:

- How many commits on `main`?  
- Is feature branch deleted after merge?  
- Which tag points to which module version?  

---

## Optional: apply from `environments/dev`

Only if you have AWS credentials and want to reinforce Git + Terraform:

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars
# edit bucket_name — do not commit terraform.tfvars
terraform apply
terraform destroy
```

Never commit `terraform.tfvars`.

---

## Cleanup

- Delete remote repo if created for practice (`gh repo delete` with confirmation).  
- Remove local folder when finished: `labs/work/lab-11/` (lab work is gitignored under `work/`).  
- If you applied AWS resources, `terraform destroy` from `environments/dev`.

---

## Lab completion checklist

- [ ] Repository structure: `modules/`, `environments/dev`, `.gitignore`  
- [ ] Feature branch created and merged (local or GitHub PR)  
- [ ] `terraform fmt` and `validate` run before merge  
- [ ] PR description/checklist understood  
- [ ] Annotated tag `v1.0.0` created  
- [ ] Versioning strategy documented (semver + module `ref`)  

## Self-review questions

| Topic | Question |
|-------|----------|
| Git branching | When would you use `hotfix/` vs `feature/`? |
| Feature branch | Why keep branches short-lived? |
| Pull requests | What belongs in the plan section of a PR? |
| Repository structure | Why separate state per environment? |
| Versioning | When is a MAJOR version bump required? |

## Previous lab

[Lab 10 — Remote State (S3)](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md).

## Next lab

[Lab 12 — Enterprise Best Practices](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md): folder structure, naming and tagging standards, secrets management, and Dev / QA / UAT / Production roots with reusable AWS modules.