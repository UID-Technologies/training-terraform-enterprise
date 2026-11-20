# Lab 14 — Remote Execution: Remote Runs, Speculative Plans, Apply Approval, Cost Estimation, and Run History (HCP Terraform + AWS)

## Description

You operate **HCP Terraform** (formerly Terraform Cloud) **remote execution** end to end: **remote runs** on HashiCorp agents, **speculative plans** from version control, **apply approval** gates, **cost estimation** on plans, and **run history** for audits. You extend the [Lab 13](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md) audit bucket workspace pattern with governance settings and a controlled apply. Optional steps connect **GitHub** for pull-request speculative plans ([Lab 11](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md) Git skills).

## Prerequisites

- Completed [Lab 13](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md) (HCP org, workspace, variables, AWS env vars)  
- `terraform login` and the same HCP **organization** as Lab 13  
- AWS credentials in workspace **environment variables** (remote runs)  
- Terraform 1.5+  
- Optional: GitHub repository and **VCS connection** in HCP for speculative plan exercises

---

## Topic explanation

### Remote runs

A **remote run** executes `terraform plan` or `terraform apply` on **HCP-managed agents**, not on your laptop. Your CLI (or VCS webhook) **uploads configuration** and variable values; the platform holds **state** and enforces **locking**.

| Trigger | Typical use |
|---------|-------------|
| **CLI** (`terraform plan` / `apply`) | Engineer-driven runs from `labs/work/lab-14/` |
| **UI** (Run → Plan / Apply) | Break-glass or operators without local config |
| **VCS push / PR** | Continuous validation on branches |
| **API / automation** | Pipelines, scheduled drift detection |

**Run lifecycle:**

```text
Queued → Planning → Planned (awaiting apply) → Applying → Applied
                              ↘ Discarded / Errored
```

| Phase | Remote run behavior |
|-------|---------------------|
| **Plan** | Reads state; computes diff; stores plan artifact |
| **Apply** | Uses saved plan; updates state; writes run log |

**Execution mode** (workspace setting):

- **Remote** (this lab) — agents run Terraform in HCP  
- **Local** — CLI runs locally; state still remote  
- **Agent** (Enterprise) — private agents in your network  

### Speculative plans

A **speculative plan** is a **plan-only** run triggered from a **non-default VCS branch** (for example a feature branch or open pull request). It answers: “What would change if we merged this?” **without** updating workspace state and **without** allowing apply from that run.

| Property | Speculative plan | Normal plan (default branch) |
|----------|------------------|------------------------------|
| Updates state | No | No (until apply) |
| Can apply | No | Yes (if policy allows) |
| Typical trigger | PR / push to branch | Push to `main` or CLI |
| Purpose | PR checks, early review | Pre-apply gate |

Enable **Speculative plans** under workspace **Version Control** settings when a VCS provider is connected.

```text
feature/add-versioning ── push ──► HCP speculative plan ──► PR comment / status check
main ────────────────── push ──► plan (non-speculative) ──► optional auto-apply
```

If you are **CLI-only** (no VCS), you still run **remote plans**, but speculative plans require a connected repository—see optional Step 9.

### Apply approval

Production and shared workspaces usually **block unattended applies**.

| Control | Setting (workspace) |
|---------|---------------------|
| **Manual apply** | User must confirm apply in UI after plan (common for Production) |
| **Auto apply** | Apply automatically when plan succeeds on default branch (Dev only) |
| **Plan-only** | Disable apply for auditors |
| **Run tasks** / policy | Org policies (Sentinel/OPA) before apply |

**CLI flow with manual apply:** `terraform apply` creates a plan run; you **confirm apply** in the HCP run UI (or approve when prompted, depending on workspace configuration).

**Separation of duties:** one person opens the PR and plan; another approves apply—record both in **run history**.

### Cost estimation

HCP Terraform can show **estimated monthly cost** for planned changes (integration with [Infracost](https://www.infracost.io/) in organization settings). On a **plan** run, open the **Cost** or **Cost estimation** panel (wording varies by UI version).

| What it estimates | What it does not replace |
|-------------------|---------------------------|
| Delta cost of **create/change** resources in the plan | FinOps chargeback rules, negotiated rates, tax |
| Rough monthly run-rate for supported resources | Exact AWS Cost Explorer for past usage |

For this lab’s **S3 bucket**, estimates are often **near zero**—the exercise is to **find the panel** and correlate a larger change (optional: add **versioning** or lifecycle rules) with a visible plan diff, not to memorize dollar amounts.

Enable at **Organization settings → Cost estimation** (admin), then re-run a plan.

### Run history

Each workspace keeps a **run history**: who triggered the run, commit SHA (if VCS), plan/apply status, duration, and links to logs.

| Use | How |
|-----|-----|
| **Audit** | Who applied to dev last Tuesday? |
| **Incident** | Compare last good **Applied** run to current **Errored** run |
| **Cost review** | Open past plans with cost estimates |
| **Compliance** | Export or retain logs per retention policy |

Filter runs by **Needs confirmation**, **Errored**, **Discarded**, and type **Plan** vs **Apply**.

---

## Use case / scenario

**Northwind Traders** is ready to tighten **remote execution** before opening **Production** workspaces:

1. All **applies** in `northwind-audit-dev` require **explicit approval**  
2. **Feature branches** get **speculative plans** on pull requests  
3. **FinOps** reviews **cost estimation** on every plan  
4. **Platform** uses **run history** to prove who approved each apply  

You configure the workspace, run remote **plan/apply** from `labs/work/lab-14/hcp-remote-execution/`, inspect history and cost, and optionally wire **GitHub** for speculative plans.

---

## Step-by-step instructions

> **Lab root folder:** **`labs/work/lab-14/hcp-remote-execution/`**

You may **reuse** HCP workspace `northwind-audit-dev` from Lab 13 or create **`northwind-audit-execution`** in project **audit-platform** (recommended if Lab 13 state should stay untouched).

### Step 1 — Workspace settings (remote execution + apply approval)

In HCP Terraform → **audit-platform** → your workspace → **Settings**:

| Setting | Value for lab |
|---------|----------------|
| **Execution mode** | Remote |
| **Auto apply** | **Off** |
| **Apply method** | **Manual apply** (or “Require manual approval” — match your UI labels) |
| **Terraform version** | Latest 1.5+ compatible with your CLI |
| **Speculative plans** | **On** (required for optional VCS step) |

Copy **workspace variables** and **environment variables** from Lab 13 if you created a new workspace.

### Step 2 — Enable cost estimation (organization)

Organization **Settings** → **Cost estimation** → enable for **AWS** (organization admin may be required).

Note: if your org cannot enable it, document the setting location and complete the rest of the lab; cost panels may be empty.

### Step 3 — Create Terraform configuration

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-14\hcp-remote-execution" -Force | Out-Null
Set-Location "labs\work\lab-14\hcp-remote-execution"
```

#### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  cloud {
    organization = "REPLACE_WITH_YOUR_ORG"

    workspaces {
      name = "northwind-audit-execution"
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

Adjust `organization` and `workspaces.name` to match your workspace.

#### `variables.tf`

```hcl
variable "aws_region" {
  type = string
}

variable "environment" {
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

variable "unique_suffix" {
  type = string
}

variable "force_destroy" {
  type    = bool
  default = true
}

variable "enable_bucket_versioning" {
  type        = bool
  description = "Toggle for speculative plan / cost lab diff"
  default     = false
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
      Lab         = "14-remote-execution"
    }
  }
}
```

#### `main.tf`

```hcl
locals {
  bucket_name = lower("nwt-finops-${var.environment}-use1-audit-exec-${var.unique_suffix}")
}

resource "aws_s3_bucket" "audit" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy

  tags = {
    Environment = var.environment
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

resource "aws_s3_bucket_versioning" "audit" {
  count  = var.enable_bucket_versioning ? 1 : 0
  bucket = aws_s3_bucket.audit.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

#### `outputs.tf`

```hcl
output "bucket_id" {
  value = aws_s3_bucket.audit.id
}

output "versioning_enabled" {
  value = var.enable_bucket_versioning
}
```

#### `terraform.tfvars.example`

```hcl
aws_region               = "us-east-1"
environment              = "dev"
application              = "northwind-audit"
owner                    = "platform-team@contoso.com"
cost_center              = "CC-1042"
unique_suffix            = "yourinitials"
force_destroy            = true
enable_bucket_versioning = false
```

Set the same values as **Terraform variables** in HCP (do not commit `terraform.tfvars`).

### Step 4 — Remote run: first plan (CLI)

```powershell
terraform login
terraform init
terraform plan
```

Open the run in the HCP UI. Verify:

- **Run type** is **Plan**  
- **Execution** shows **Remote**  
- **Run status** ends at **Planned** (or **Finished** for plan-only)  
- **Cost estimation** section (if enabled)  

Record **run ID** and **duration** in your notes for **run history** comparison.

### Step 5 — Remote run: apply with approval

```powershell
terraform apply
```

With **manual apply**:

1. CLI uploads configuration and starts a **plan** phase on HCP.  
2. When the plan succeeds, **confirm apply** in the HCP run page (or follow CLI prompt if your workspace allows CLI confirmation).  
3. Wait until status is **Applied**.  

**Exercise:** pause before confirming and locate **Needs confirmation** in the workspace **Runs** list—this is the **apply approval** queue.

After apply, confirm `outputs` in the UI run **Outputs** tab.

### Step 6 — Run history review

Workspace → **Runs**:

1. Select the successful **Apply** run → note **User** or **Source** (CLI vs UI).  
2. Open the prior **Plan-only** run from Step 4 → confirm it did **not** change state.  
3. Use filters to show only **Errored** or **Discarded** (if any from earlier attempts).  

**Worksheet:**

| Run | Type | Source | State updated? | Approved by |
|-----|------|--------|----------------|-------------|
| | Plan | CLI | No | N/A |
| | Apply | CLI | Yes | |

### Step 7 — Cost estimation exercise

1. In HCP, set workspace Terraform variable `enable_bucket_versioning` = `true` (HCL: `true`).  
2. Run `terraform plan` again (remote).  
3. Open the new run; review **Cost estimation** and the **plan diff** (`aws_s3_bucket_versioning`).  
4. Set `enable_bucket_versioning` back to `false` and plan again to see the destroy/change in the diff (do not apply unless you want versioning on).

Discuss: why S3 versioning might increase **storage** cost over time even when monthly estimate is small.

### Step 8 — UI-triggered remote run

In the workspace, click **Actions** → **Start new run** → **Plan only**.

Compare this run in **run history** to CLI-triggered runs (source = UI). No local folder required for this action—useful for operators.

### Step 9 — Optional: speculative plans (VCS + feature branch)

Requires **GitHub** (or GitLab/ADO) connected to the workspace.

1. Push `labs/work/lab-14/hcp-remote-execution/` to a new repo (or monorepo path).  
2. HCP workspace → **Version Control** → connect repo; set **Terraform Working Directory** if not repo root.  
3. Default branch: `main` (with `enable_bucket_versioning = false` in HCP variables).  
4. Locally:

```powershell
git checkout -b feature/enable-versioning
```

5. Change `main.tf` default in code **or** add a `README` change plus set `enable_bucket_versioning` default to `true` only on branch via a code change:

```hcl
# In variables.tf on branch — set default = true for demo
variable "enable_bucket_versioning" {
  default = true
}
```

6. Push branch and open a **pull request**.  
7. In HCP (or PR checks), open the **speculative plan** run:

- Confirm run is marked **Speculative**  
- Confirm **Apply** is **not** available  
- Use output in PR description (intent, risk, cost screenshot)

8. Merge PR → default branch run should be a **non-speculative** plan; apply only if policy allows.

### Step 10 — Governance summary

Write three sentences for Northwind stakeholders:

1. When **auto apply** is allowed vs forbidden  
2. Who may **approve** applies in dev vs production  
3. How **speculative plans** reduce production surprises  

---

## Cleanup

```powershell
terraform destroy
```

Confirm apply again if **manual apply** is enabled.

Optional: delete workspace `northwind-audit-execution` after destroy.

Remove `labs/work/lab-14/` locally.

---

## Lab completion checklist

- [ ] Workspace **remote** execution confirmed  
- [ ] **Auto apply** disabled; **manual apply** exercised  
- [ ] CLI **remote plan** and **remote apply** completed  
- [ ] **Run history** reviewed (plan vs apply, source, user)  
- [ ] **Cost estimation** enabled (or documented) and viewed on a plan  
- [ ] Variable toggle `enable_bucket_versioning` used to produce a meaningful plan diff  
- [ ] Optional: **speculative plan** on a feature branch / PR  
- [ ] Optional: **UI-triggered** plan-only run  

## Self-review questions

| Topic | Question |
|-------|----------|
| Remote runs | Where is state updated during a remote apply? |
| Speculative plans | Can you apply from a speculative plan? Why not? |
| Apply approval | What is the risk of auto apply on the default branch? |
| Cost estimation | Why might FinOps still use AWS Cost Explorer? |
| Run history | What evidence does history provide for SOC2-style change control? |
| Lab 13 vs 14 | Lab 13 focuses on variables; what does Lab 14 add? |

## Previous lab

[Lab 13 — HCP Terraform Workspaces](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md).

## Next lab

[Lab 15 — Install and Configure Terrakube](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md): self-hosted Terraform automation on Minikube with Helm, HTTPS, workspaces, and Git-driven runs.