# Lab 18 — Terraform CI/CD with GitHub Actions and Terrakube

## Description

You build a **Terraform CI/CD pipeline** for **Contoso**: **GitHub Actions** for **fmt**, **validate**, and **plan** on every push and pull request; **GitHub Environments** for **deployment approval** gates; then **Terrakube** for **authoritative remote plan/apply** and **persistent state**. No AWS or Azure credentials required—the **`random`** provider only.

**Duration:** 2.5–3 hours | **Level:** beginner to intermediate

Builds on [Lab 11](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md), [Lab 16–17](./Lab%2017%20-%20Terrakube%20Workspaces%20and%20Remote%20Terraform%20Execution.md) (Terrakube + GitHub workspaces).

## Prerequisites

- GitHub account with permission to create repositories and **Actions**  
- [Lab 15](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md) + [Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md): Terrakube reachable; GitHub VCS configured  
- Git installed; optional [GitHub CLI](https://cli.github.com/) (`gh`)  
- Terraform 1.5+ locally (optional, for `terraform fmt` before push)

### Critical teaching limitation

| Runner | State | Role in this lab |
|--------|-------|------------------|
| **GitHub Actions** (`ubuntu-latest`) | **None** persisted (`init -backend=false`) | **CI**: catch bad HCL early |
| **Terrakube executor** | **Workspace remote state** | **Authoritative** plan/apply |

**Do not** treat the GitHub Actions **plan** as the deployment plan after Terrakube has applied. Production uses one backend and one execution path (here: Terrakube).

---

## Topic explanation

### Business scenario

Contoso developers today run `terraform init/plan/apply` on laptops. Problems:

- Inconsistent Terraform versions  
- No mandatory review  
- State on individual machines  
- No central run history  
- Production changes without approval  

**Solution:** Terraform in **GitHub**; **Actions** for CI; **Terrakube** for gated remote execution and state ([Lab 14](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md) shows the same pattern on HCP).

### Target architecture

```text
Developer ── push/PR ──► GitHub
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
     GitHub Actions (CI)          Terrakube workspace
     fmt, validate, plan          plan / apply + state
     (ephemeral, no state)        (authoritative)
```

**Part F handoff (Minikube class):** After merge to `main`, run **Terrakube plan/apply manually**—GitHub-hosted runners cannot reach `terrakube-api.minikube.net`. Full automation needs a **public Terrakube API**, **VCS webhooks**, or **authenticated API call** from Actions (verify against your Terrakube version).

### GitHub Actions building blocks

| Concept | Meaning |
|---------|---------|
| `on:` | Triggers (`push`, `pull_request`, `workflow_dispatch`) |
| `jobs` | Work units |
| `runs-on` | Runner OS (`ubuntu-latest`) |
| `steps` | Shell commands or `uses:` actions |
| `environment:` | Deployment protection (reviewers, secrets) |

---

## Use case / scenario

Repository **`terraform-github-actions-lab`** contains root-module Terraform and two workflows:

- **`.github/workflows/terraform-ci.yml`** — validation on push/PR  
- **`.github/workflows/terraform-deploy.yml`** — approval gate demo  

Terrakube workspace **`github-actions-development`** runs apply after CI and review.

**Optional local clone:** `labs/work/lab-18/terraform-github-actions-lab/` (mirror of your GitHub repo).

---

## Step-by-step instructions

### Step 1 — Create the GitHub repository

Create **`terraform-github-actions-lab`** (public or private) with README.

```powershell
gh repo create terraform-github-actions-lab --public --clone --description "Terraform CI/CD lab"
```

Or use [Create a new repository](https://github.com/new).

Clone to the lab folder if you did not use `--clone`:

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-18" -Force | Out-Null
Set-Location "labs\work\lab-18"
git clone https://github.com/YOUR-USERNAME/terraform-github-actions-lab.git
Set-Location terraform-github-actions-lab
```

### Step 2 — Add Terraform configuration

Create **`main.tf`**:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "development"
}

resource "random_pet" "application" {
  prefix    = var.environment
  length    = 2
  separator = "-"
}

output "application_name" {
  value = random_pet.application.id
}
```

| Block | Purpose |
|-------|---------|
| `terraform` | Version and provider constraints |
| `variable "environment"` | Configurable prefix |
| `random_pet` | Lab resource without cloud API |
| `output` | Verify in Terrakube job output |

Create **`.gitignore`**:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
*.tfvars
*.tfvars.json
```

**Commit `.terraform.lock.hcl`** after first `terraform init` (do not ignore it)—keeps provider versions consistent in CI.

```powershell
terraform init -backend=false
git add main.tf .gitignore .terraform.lock.hcl
git commit -m "Add Terraform configuration"
git push
```

### Step 3 — Create the CI workflow

Create **`.github/workflows/terraform-ci.yml`**:

```yaml
name: Terraform CI Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read

jobs:
  terraform:
    name: Terraform Validation and Plan
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.8"

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Init
        run: terraform init -backend=false

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -input=false
```

| Step | Purpose |
|------|---------|
| `checkout@v4` | Clone repo on runner |
| `setup-terraform@v3` | Pin CLI version |
| `fmt -check` | Enforce formatting |
| `init -backend=false` | Providers without remote backend |
| `validate` | Static configuration check |
| `plan` | Proposed changes (no persistent state on runner) |

```powershell
git add .github/workflows/terraform-ci.yml
git commit -m "Add Terraform CI workflow"
git push
```

### Step 4 — Verify the CI workflow

GitHub → **Actions** → **Terraform CI Pipeline** → latest run.

**Expected:** format, init, validate succeed; plan shows **1 to add** (no state on runner).

If `fmt -check` fails:

```powershell
terraform fmt
git add -u
git commit -m "terraform fmt"
git push
```

**Checkpoint:** all CI steps green.

### Step 5 — Change configuration and re-run CI

In `main.tf`, set `length = 3`. Commit and push.

Open the new run → **Terraform Plan** step. Expect plan with `length = 3`.

**Why always “create”?** Each Actions run is **stateless**. The runner does not see Terrakube state. This is expected until CI uses a **remote backend** or you rely on **Terrakube** for real plans ([Lab 10](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md)).

### Step 6 — Manual workflow dispatch

`workflow_dispatch` is already in Step 3. Test: **Actions** → **Terraform CI Pipeline** → **Run workflow** → branch `main`.

**Checkpoint:** pipeline runs on demand and on push/PR.

### Step 7 — Create GitHub deployment environments

Repository → **Settings** → **Environments** → **New environment**:

| Name | Purpose |
|------|---------|
| `terraform-development` | Dev deployment gate |
| `terraform-production` | Prod gate (optional reviewers) |

For **`terraform-production`**, enable **Required reviewers** if your GitHub plan supports it (private repos / Team features). Classrooms without reviewers: use **PR review** + manual Terrakube apply instead.

### Step 8 — Create deployment approval workflow

Create **`.github/workflows/terraform-deploy.yml`**:

```yaml
name: Terraform Deployment Approval

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        type: choice
        options:
          - development
          - production

permissions:
  contents: read

jobs:
  deployment-gate:
    runs-on: ubuntu-latest
    environment:
      name: terraform-${{ inputs.environment }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Display Deployment Context
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Commit: ${{ github.sha }}"

      - name: Confirm Approval Gate
        run: |
          echo "GitHub deployment gate passed."
          echo "Infrastructure execution will be handled by Terrakube."
```

This demonstrates **approval before deploy**—it does **not** run `terraform apply` on GitHub.

```powershell
git add .github/workflows/terraform-deploy.yml
git commit -m "Add deployment approval workflow"
git push
```

### Step 9 — Test deployment approval

**Actions** → **Terraform Deployment Approval** → **Run workflow** → `environment = production`.

If reviewers are configured, approve the pending deployment. Job should complete after the gate.

**Checkpoint:** participants see how **environment protection** delays execution.

### Step 10 — Pull request and CI on a feature branch

```powershell
git checkout -b feature/change-application-name
```

Change default in `main.tf`:

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "testing"
}
```

```powershell
git add main.tf
git commit -m "Update application environment default"
git push -u origin feature/change-application-name
```

Open a **pull request** to `main`. Confirm **Terraform CI Pipeline** runs on the PR.

### Step 11 — Demonstrate CI failure and recovery

On the feature branch, introduce an error:

```hcl
resource "random_pet" "application" {
  prefix    = var.environment
  length    = "invalid"
  separator = "-"
}
```

Push; CI should fail at **validate** or **plan**.

Fix:

```hcl
  length    = 3
```

Push again; CI should pass.

**Teaching point:** CI blocks invalid config before Terrakube apply.

Merge the PR when ready (or keep separate branches per class policy).

### Step 12 — Create Terrakube workspace

Terrakube UI → **New workspace** → Version Control:

| Setting | Value |
|---------|--------|
| Workspace name | `github-actions-development` |
| Repository | `terraform-github-actions-lab` |
| Branch | `main` |
| Working directory | `/` |
| Terraform version | `>= 1.5.0` compatible |
| Terraform variable | `environment` = `development` |

Save. Start a **Plan** job; confirm clone, init, and plan succeed.

### Step 13 — Remote apply and persistent state

Run **Apply** (or approved workflow) in Terrakube.

**Expected apply excerpt:**

```text
random_pet.application: Creation complete
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Run a second **Plan** without code changes:

```text
No changes. Your infrastructure matches the configuration.
```

**Checkpoint:** Terrakube state is **authoritative**; GitHub Actions plan is **not**.

### Step 14 — GitHub CI + Terrakube handoff

1. Merge a validated PR to `main`.  
2. Confirm **Actions** CI is green on `main`.  
3. In Terrakube, run **Plan** on `github-actions-development` against updated `main`.  
4. Review and **Apply** if appropriate.

**Automation note:** wiring CI → Terrakube requires webhooks or API calls from Actions to a **reachable** Terrakube API—not available for Minikube-only APIs without a tunnel.

### Step 15 — Optional: API-triggered Terrakube job (advanced)

For production designs, a deployment job after approval might call Terrakube’s HTTP API with a token stored in GitHub **secrets**. Endpoints and payloads vary by version—verify [Terrakube API documentation](https://docs.terrakube.io/) for your release. Not required to complete this lab.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|--------|-----|
| `terraform: command not found` | Setup action failed | Check `hashicorp/setup-terraform@v3` |
| `fmt -check` failed | Unformatted HCL | `terraform fmt` and commit |
| `init` failed | Registry/network | Retry; verify `versions` block |
| `validate` failed | Invalid config | Fix types/syntax (Step 11) |
| Plan always **1 to add** in Actions | No backend on runner | Expected; use Terrakube plan |
| Deployment waits | Environment protection | Approve or use dev environment |
| Terrakube clone fails | VCS/SSH/OAuth | [Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md) |
| Job queued in Terrakube | Executor down | `kubectl get pods -n terrakube` |

---

## Cleanup

- Delete Terrakube workspace **`github-actions-development`** when finished (follow UI for state).  
- Delete or archive GitHub repo **`terraform-github-actions-lab`**.  
- Remove `labs/work/lab-18/` locally.

---

## Lab completion checklist

- [ ] Repo **`terraform-github-actions-lab`** with `main.tf`, `.gitignore`, lock file  
- [ ] **`terraform-ci.yml`** runs on push, PR, and `workflow_dispatch`  
- [ ] CI: fmt, init, validate, plan succeed  
- [ ] Plan on runner understood as **stateless**  
- [ ] Environments **`terraform-development`** and **`terraform-production`** created  
- [ ] **`terraform-deploy.yml`** approval workflow tested  
- [ ] PR + intentional CI failure/recovery demonstrated  
- [ ] Terrakube workspace **`github-actions-development`** plan/apply complete  
- [ ] Terrakube second plan shows **no changes**  
- [ ] Handoff: green CI on `main` → manual Terrakube plan documented  

## Self-review questions

| Topic | Question |
|-------|----------|
| CI vs CD | What does this repo deploy automatically—and what does it not? |
| State | Why does Actions plan differ from Terrakube plan after apply? |
| Apply on GHA | Why avoid `apply -auto-approve` on every PR? |
| Environments | What does `environment: terraform-production` enforce? |
| Authoritative plan | Where should Contoso run apply before production? |
| vs Lab 14 | How are GitHub Environments similar to HCP apply approval? |

## Instructor recommendation (live demo)

1. **Green CI** → break `length` type → fix → pass.  
2. **Manual dispatch** + **production** environment approval wait.  
3. **Terrakube apply** + second plan **no changes** on same repo.

Emphasize: **GitHub Actions = quality gate; Terrakube = execution and state** (Minikube class: manual step between them).

## Previous lab

[Lab 17 — Terrakube Workspaces and Remote Execution](./Lab%2017%20-%20Terrakube%20Workspaces%20and%20Remote%20Terraform%20Execution.md).

## Next lab

[Lab 19 — Terraform Security and AWS Secrets Manager](./Lab%2019%20-%20Terraform%20Security%20and%20AWS%20Secrets%20Manager.md): secret containers, IAM least privilege, state security, optional GitHub OIDC.

## Reference documentation

- [GitHub Actions](https://docs.github.com/en/actions)  
- [hashicorp/setup-terraform](https://github.com/hashicorp/setup-terraform)  
- [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)  
- [Terrakube documentation](https://docs.terrakube.io/)