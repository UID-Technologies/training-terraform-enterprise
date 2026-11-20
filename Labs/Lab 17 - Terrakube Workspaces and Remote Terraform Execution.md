# Lab 17 — Terrakube Workspaces and Remote Terraform Execution

## Description

You create **Terrakube workspaces** linked to GitHub, configure **workspace variables**, and run **`terraform plan`** and **`terraform apply` remotely** on the Terrakube **executor**—without running Terraform on your laptop after the initial Git push. You verify **remote state**, add a second workspace to model **dev vs production**, and re-plan after a Git change. Uses the **`random`** provider only (no cloud charges).

**Duration:** 60–90 minutes

This lab deepens [Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md) (GitHub VCS) and parallels [Lab 13–14](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md) (HCP workspaces and remote runs).

## Prerequisites

- [Lab 15](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md): Terrakube on Minikube; UI at **https://terrakube-ui.minikube.net**  
- [Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md): GitHub **VCS** (OAuth, public URL, or SSH) working  
- GitHub account; Git installed  
- Optional: reuse Lab 16 org/VCS provider **`github-training`**

**Important:** After Step 3, perform Terraform operations **only in Terrakube** (remote jobs), not `terraform apply` on your PC.

---

## Topic explanation

### What is a Terrakube workspace?

A **workspace** is a managed environment: **VCS source**, **variables**, **execution settings**, **remote state**, and **job history** for one deployment target.

```text
Organization
 └── Workspace (demo-development)
        ├── Git: terrakube-remote-execution @ main
        ├── Terraform variables: environment = development
        ├── State: random_pet.application (workspace-scoped)
        └── Jobs: plan / apply logs
```

| Example workspace | Git branch (typical) | Purpose |
|-------------------|----------------------|---------|
| `app-dev` | `develop` | Development |
| `app-qa` | `qa` | QA |
| `app-prod` | `main` | Production |

This lab uses **two workspaces** on the **same** repo and branch with **different Terraform variables**—the common pattern when code is shared but **state must not** be shared ([Lab 12](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md)).

### Remote execution

| Local workflow | Terrakube remote execution |
|----------------|----------------------------|
| `terraform init/plan/apply` on laptop | Executor runs Terraform in Kubernetes |
| Local or ad-hoc backend | **Workspace-managed** remote state |
| Credentials on developer machine | Credentials (when needed) in workspace/org config |

```text
Developer ──► Terrakube UI / API ──► Executor pod ──► terraform plan|apply
                      │                              │
                      └──────── remote state ◄─────────┘
```

**Teaching point:** Participants do **not** need cloud credentials for this lab; the executor still needs outbound access to **GitHub** and the **Terraform Registry**.

### Terraform variables vs environment variables

| Type | Example | Purpose |
|------|---------|---------|
| **Terraform variable** | `environment = development` | Sets `var.environment` in `main.tf` |
| **Environment variable** | `TF_LOG=INFO` | Terraform CLI/runtime logging |

Configure `environment` as a **Terraform variable** in the workspace (not `TF_VAR_` unless your org standardizes on env injection).

### Local Terraform vs Terrakube

| Feature | Local Terraform | Terrakube |
|---------|-----------------|-----------|
| Execution | Developer machine | Central executor |
| State | Local file or custom backend | Workspace remote state |
| Git | Manual / external CI | Integrated VCS workflow |
| Logs | Terminal | Job history in UI |
| Collaboration | Extra tooling | Orgs, workspaces, RBAC |
| Automation | Scripts / CI | Job templates, triggers |

---

## Use case / scenario

**Adventure Works** wants developers to **stop running apply on laptops**. Platform engineering requires:

1. Git repo **`terrakube-remote-execution`** with one root module  
2. Workspace **`demo-development`** with `environment = development`  
3. Remote **plan** and **apply** with logs and state in Terrakube  
4. Workspace **`demo-production`** with `environment = production` and **separate state**  
5. A Git change that triggers a **replace** plan in development  

**Lab folder (Git only):** `labs/work/lab-17/terrakube-remote-execution/`

---

## Step-by-step instructions

> **From repository root**, use **`labs/work/lab-17/terrakube-remote-execution/`**.

### Step 1 — Confirm Terrakube and executor health

```powershell
kubectl get pods -n terrakube
```

Open **https://terrakube-ui.minikube.net**, sign in, and confirm your organization and **GitHub VCS** from Lab 16.

```powershell
kubectl get deployments -n terrakube
kubectl get events -n terrakube --sort-by=.lastTimestamp
```

**Checkpoint:** Executor-related pods/deployments are **Running** / **Ready**.

### Step 2 — Create the GitHub repository

Create **`terrakube-remote-execution`** (public recommended) on GitHub, with or without README.

```powershell
gh repo create terrakube-remote-execution --public --description "Terrakube remote execution lab"
```

Or use [github.com/new](https://github.com/new).

### Step 3 — Prepare Terraform code locally (push only—no local apply)

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-17\terrakube-remote-execution" -Force | Out-Null
Set-Location "labs\work\lab-17\terrakube-remote-execution"
```

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
  type        = string
  description = "Environment name injected by Terrakube workspace variable"
}

resource "random_pet" "application" {
  prefix    = var.environment
  length    = 2
  separator = "-"
}

output "application_name" {
  description = "Lab output for workspace comparison"
  value       = random_pet.application.id
}
```

Create **`.gitignore`**:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
.terraform.lock.hcl
```

Optional local syntax check only:

```powershell
terraform init -backend=false
terraform validate
```

Do **not** run `terraform apply` locally—state belongs in Terrakube.

Push to GitHub:

```powershell
git init
git remote add origin https://github.com/YOUR-USERNAME/terrakube-remote-execution.git
git add main.tf .gitignore
git commit -m "Add remote execution lab Terraform"
git branch -M main
git push -u origin main
```

### Step 4 — Create workspace `demo-development`

Terrakube UI → Organization → **Workspaces** → **New workspace** → **Version Control** workflow.

Docs: [Creating workspaces](https://docs.terrakube.io/user-guide/workspaces/creating-workspaces)

| Property | Value |
|----------|--------|
| Workspace name | `demo-development` |
| VCS provider | Your Lab 16 GitHub connection |
| Repository | `terrakube-remote-execution` |
| Branch | `main` |
| Working directory | `/` (repository root) |
| Terraform version | Compatible with `>= 1.5.0` |
| Execution | Remote Terrakube execution (default) |

Save the workspace.

**Checkpoint:** Workspace appears in the list and is linked to the repository.

### Step 5 — Configure workspace variables (`demo-development`)

Open **`demo-development`** → **Variables**.

| Name | Value | Category | HCL |
|------|-------|----------|-----|
| `environment` | `development` | **Terraform variable** | yes (`"development"` or unquoted per UI) |

Optional (diagnostics only):

| Name | Value | Category |
|------|-------|----------|
| `TF_LOG` | `INFO` | **Environment variable** |

**Checkpoint:** `environment` is visible as a Terraform variable, not only as env text.

### Step 6 — Verify remote execution settings

In **`demo-development`**, confirm:

| Configuration | Expected |
|---------------|----------|
| Repository | GitHub connected |
| Branch | `main` |
| Terraform version | Selected and compatible |
| Workspace variables | `environment = development` |
| Job templates | **Plan** (and **Apply**) available |
| Executor | Healthy (Step 1) |

For the first run, choose a **Plan-only** job template if offered.

### Step 7 — Run Terraform plan remotely

1. Open **Jobs** → **New job** / **Run**.  
2. Select **Terraform Plan** template.  
3. Start execution and open logs.

**Expected log patterns:**

```text
Initializing the backend...
Initializing provider plugins...
- Installing hashicorp/random v...

Terraform will perform the following actions:

  # random_pet.application will be created
  + resource "random_pet" "application" {
      + length = 2
      + prefix = "development"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**What happened:**

1. Terrakube cloned GitHub configuration.  
2. Executor prepared the Terraform runtime.  
3. Providers downloaded inside the job.  
4. Plan compared config to **workspace state** (empty first time).  
5. Results stored in **job history**.

**Key point:** Terraform ran on **Terrakube**, not on your laptop.

### Step 8 — Run Terraform apply remotely

1. Start an **Apply** job (or Plan+Apply workflow).  
2. Approve if the workspace requires approval.  
3. Wait for completion; read logs.

**Expected apply excerpt:**

```text
random_pet.application: Creating...
random_pet.application: Creation complete after ...

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

application_name = "development-happy-tiger"
```

Names vary. If Plan and Apply are separate, apply the **reviewed** plan per your Terrakube version—do not skip plan review.

### Step 9 — Verify remote state

Open workspace **State** (or state tab in your UI version).

Confirm resource address:

```text
random_pet.application
```

Run a second **Plan** job **without** changing Git or variables.

**Expected:**

```text
No changes. Your infrastructure matches the configuration.
```

This shows **idempotent** remote execution and workspace-scoped state.

### Step 10 — Create workspace `demo-production`

Create a second workspace with the **same** repository and branch:

| Property | `demo-development` | `demo-production` |
|----------|--------------------|-------------------|
| Workspace name | `demo-development` | `demo-production` |
| Repository | `terrakube-remote-execution` | Same |
| Branch | `main` | `main` |
| Terraform variable `environment` | `development` | `production` |
| State | Independent | Independent |

Run **plan** and **apply** in **`demo-production`**.

Compare outputs (examples):

```text
demo-development: development-happy-tiger
demo-production:  production-calm-lion
```

**Pattern:** one repo, multiple workspaces, **isolated state**—same idea as [Lab 13](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md) variable sets per workspace.

### Step 11 — Change Git configuration and re-plan

On GitHub (or local push), update `main.tf`:

```hcl
resource "random_pet" "application" {
  prefix    = var.environment
  length    = 3
  separator = "-"
}
```

```powershell
git add main.tf
git commit -m "Change random pet length to 3"
git push
```

In **`demo-development`**, start a new **Plan** job (manual trigger on Minikube unless webhooks are exposed—see [Lab 16](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md)).

**Expected plan:**

```text
-/+ random_pet.application must be replaced

Plan: 1 to add, 0 to change, 1 to destroy.
```

Review, then **Apply** if policy allows. Confirm state shows the updated resource.

**Note:** `demo-production` can run the same plan when ready—production may require approval workflows in real orgs.

### Step 12 — Troubleshooting remote execution

| Issue | Likely cause | Action |
|-------|----------------|--------|
| Job **queued** | Executor unavailable | `kubectl get pods -n terrakube` |
| **Git clone** fails | VCS/SSH/OAuth | Re-check Lab 16 connection |
| **Provider** download fails | No registry egress | Test network from executor pod |
| **Variable missing** | Wrong name/category | `environment` as **Terraform variable** |
| **Terraform version** error | Mismatch | Bump workspace Terraform version |
| **State** errors | API/storage/TLS | API URL, certs, `helm status terrakube -n terrakube` |

```powershell
kubectl get pods -n terrakube
kubectl logs deployment/<executor-deployment> -n terrakube
```

Replace `<executor-deployment>` with the name from `kubectl get deployments -n terrakube`.

---

## Cleanup

- Optionally **destroy** resources: run **Apply** with destroy job template if your Terrakube version supports it, or remove the `random_pet` resource in Git and apply.  
- Delete workspaces **`demo-development`** and **`demo-production`** when no longer needed.  
- Delete GitHub repo **`terrakube-remote-execution`** or keep for reuse.  
- Remove `labs/work/lab-17/` locally.

---

## Lab completion checklist

- [ ] GitHub repo **`terrakube-remote-execution`** pushed; no local `apply` used for state  
- [ ] Workspace **`demo-development`** created with VCS on `main`  
- [ ] Terraform variable **`environment`** = `development` configured  
- [ ] Remote **plan** completed; logs show provider install and plan  
- [ ] Remote **apply** completed; output `application_name` captured  
- [ ] **State** verified; second plan shows **no changes**  
- [ ] Workspace **`demo-production`** with `environment = production` and separate output  
- [ ] Git change (`length = 3`) produced **replace** plan in development  
- [ ] Understand **Terraform variable** vs **environment variable** in UI  

## Self-review questions

| Topic | Question |
|-------|----------|
| Workspace | What four things does a workspace bind together? |
| Remote execution | Who runs `terraform init` in this lab? |
| State | Why do dev and prod workspaces need separate state? |
| Variables | What breaks if `environment` is set as env var but not Terraform var? |
| Idempotency | Why does a second plan show no changes? |
| vs Lab 14 | How is apply approval similar on HCP and Terrakube? |

## Instructor takeaway

A Terrakube workspace is not a folder on disk—it is **Git + variables + executor + remote state + audit history**. Optional next course topic: Terrakube workspace with **AWS** or **Azure** and centrally stored cloud credentials.

## Previous lab

[Lab 16 — Connect Terrakube with GitHub](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md).

## Next lab

[Lab 18 — Terraform CI/CD with GitHub Actions and Terrakube](./Lab%2018%20-%20Terraform%20CI%20CD%20with%20GitHub%20Actions%20and%20Terrakube.md): Actions fmt/validate/plan, deployment environments, Terrakube as authoritative apply.

## Reference documentation

- [Creating workspaces](https://docs.terrakube.io/user-guide/workspaces/creating-workspaces)  
- [Terrakube documentation](https://docs.terrakube.io/)