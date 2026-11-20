# Lab 16 — Connect Terrakube with GitHub (VCS Integration)

## Description

You connect **Terrakube** to **GitHub** so Terraform configuration in Git is executed centrally: **plan**, **apply**, **remote state**, and **job history** in Terrakube workspaces. This lab focuses on **GitHub as a source repository** (Version Control workflow). **GitHub as a sign-in provider** (OIDC/OAuth for users) is noted for context only—not configured here.

You use the **`random`** provider so **no AWS or Azure credentials** are required. This lab builds on [Lab 15](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md) (Terrakube on **Minikube**) and reinforces [Lab 11](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md) (Git workflow).

**Estimated time:** 90–120 minutes

## Prerequisites

- Completed [Lab 15](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md): Terrakube running, UI at **https://terrakube-ui.minikube.net**, you can sign in and open an organization  
- [Git](https://git-scm.com/downloads) installed  
- [GitHub](https://github.com) account  
- Terrakube **executor** pods can reach `github.com` over HTTPS (outbound from the cluster)  
- Optional: [GitHub CLI](https://cli.github.com/) (`gh auth login`) to create the repository from the terminal  
- Optional: [GitHub OAuth app](https://github.com/settings/developers) for full VCS provider integration (Steps 10–12)

### Lab environment assumption (training)

This course uses **Terrakube on local Minikube** (Lab 15), not cloud Kubernetes. Implications:

| Topic | Minikube training |
|-------|-------------------|
| **OAuth callback URL** | Use the **exact callback URL** shown in Terrakube when you add GitHub (often tied to `https://terrakube-api.minikube.net` or your chart’s Dex/API URL—not the UI homepage alone) |
| **GitHub webhooks** | GitHub **cannot** POST to private laptop hostnames unless you expose a **public HTTPS** tunnel; default path is **manual runs** + demo webhook concepts |
| **Public vs private repo** | Start with a **public** repo and public Git URL or OAuth; use **SSH deploy keys** for private repos without OAuth |

---

## Topic explanation

### Two GitHub integrations

| Integration | Purpose | This lab |
|-------------|---------|----------|
| **GitHub as VCS** | Clone repo; run Terraform from branch commits; webhooks for auto-runs | **Yes — primary** |
| **GitHub as authentication** | Users log in to Terrakube UI via GitHub (IdP) | Concept only |

### Version Control workflow

```text
Developer ── push ──► GitHub (main.tf)
                          │
              webhook (optional, needs public URL)
                          ▼
                   Terrakube workspace
                          │
                    Executor job
                          │
              init → plan → apply → state in Terrakube
```

**Benefit:** Terraform execution is **centralized**, **logged**, and **decoupled** from individual laptops—similar in spirit to [Lab 14](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md) remote runs on HCP.

### Which connection method to use

| Requirement | Recommended method |
|---------------|-------------------|
| Public repo, quickest lab | **Public Git URL** in workspace (if your Terrakube version supports it) |
| Private repo, no OAuth | **SSH deploy key** (alternative section below) |
| Org-wide GitHub, listing repos | **GitHub VCS OAuth app** |
| Auto-run on every push | OAuth VCS + **webhook** + **publicly reachable** Terrakube API |
| Minikube classroom | **Public repo** + **manual** plan/apply first; OAuth second; webhooks as discussion only |

Official references: [Creating workspaces](https://docs.terrakube.io/user-guide/workspaces/creating-workspaces), [Terrakube documentation](https://docs.terrakube.io/).

---

## Use case / scenario

**Wide World Importers** stores Terraform in **GitHub**. Platform engineering requires:

1. Repository **`terrakube-demo`** with a small root module  
2. Terrakube workspace **`terrakube-demo-dev`** on branch **`main`**  
3. Successful **remote plan and apply**  
4. A Git commit that triggers a **new plan** (manual or webhook if available)

Local project root: **`labs/work/lab-16/terrakube-demo/`** (push to GitHub; do not commit secrets or state).

---

## Step-by-step instructions

> **From repository root**, create **`labs/work/lab-16/terrakube-demo/`**.

### Step 1 — Verify Terrakube and cluster egress

```powershell
kubectl get pods -n terrakube
```

Open **https://terrakube-ui.minikube.net**, sign in, and confirm you can open **Organizations**.

From a Terrakube executor pod (replace pod name):

```powershell
kubectl get pods -n terrakube
kubectl exec -it <executor-pod> -n terrakube -- sh -c "wget -q -O- https://github.com | head -c 80"
```

If egress fails, fix cluster/DNS before continuing.

### Step 2 — Create the GitHub repository

**Option A — GitHub website**

1. Open [Create a new repository](https://github.com/new).  
2. Use:

| Setting | Value |
|---------|--------|
| Repository name | `terrakube-demo` |
| Description | Terraform automation using Terrakube |
| Visibility | **Public** (recommended for first run) |
| Initialize with README | Yes |

3. Note the URL: `https://github.com/YOUR-USERNAME/terrakube-demo`

**Option B — GitHub CLI**

```powershell
gh repo create terrakube-demo --public --description "Terraform automation using Terrakube"
```

### Step 3 — Create Terraform configuration locally

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-16\terrakube-demo" -Force | Out-Null
Set-Location "labs\work\lab-16\terrakube-demo"
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
  type    = string
  default = "dev"
}

resource "random_pet" "application" {
  length    = 2
  separator = "-"
  prefix    = var.environment
}

output "application_name" {
  description = "Generated application name for lab verification"
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

Expected layout after push:

```text
terrakube-demo/
  README.md
  main.tf
  .gitignore
```

### Step 4 — Push to GitHub

Clone your empty README repo or init and push:

```powershell
git init
git remote add origin https://github.com/YOUR-USERNAME/terrakube-demo.git
git add main.tf .gitignore README.md
git commit -m "Add Terrakube demo Terraform"
git branch -M main
git push -u origin main
```

Replace `YOUR-USERNAME`. Never commit `.terraform/` or state files.

### Step 5 — Create a workspace (Version Control workflow)

In Terrakube UI → your organization:

1. **Workspaces** → **New workspace** → **Terraform** → **Version Control workflow**  
2. If prompted for VCS: choose **public Git repository URL** (fastest for Minikube class) **or** proceed to Step 10 to attach **GitHub OAuth** first.

**Workspace settings (target values):**

| Setting | Value |
|---------|--------|
| Workspace name | `terrakube-demo-dev` |
| Workflow | Version Control |
| Repository | `YOUR-USERNAME/terrakube-demo` or public clone URL |
| Branch | `main` |
| Working directory | `/` (repository root) |
| Terraform version | Compatible with `>= 1.5.0` |
| VCS push template (if available) | **Plan only** (avoids auto-apply on every commit) |

3. **Create workspace**.

Docs: [Creating workspaces](https://docs.terrakube.io/user-guide/workspaces/creating-workspaces).

### Step 6 — Run Terraform plan remotely

1. Open workspace **`terrakube-demo-dev`**.  
2. Go to **Jobs** (or equivalent runs area).  
3. Start a **Plan** (or Plan/Apply per your template).

**Expected plan excerpt:**

```text
  # random_pet.application will be created
  + resource "random_pet" "application" {
      + length = 2
      + prefix = "dev"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

In the job log, locate: **init**, provider install, **plan** output, completion status.

### Step 7 — Run apply and inspect state

If workspace policy allows, run **Apply** (or continue a combined workflow).

Then verify in the UI:

- Job status **Finished** / **Applied**  
- **Outputs** include `application_name` (e.g. `dev-happy-tiger`)  
- **State** stored in Terrakube (not on your laptop)

### Step 8 — Change code in Git and re-plan

Edit `main.tf` locally:

```hcl
resource "random_pet" "application" {
  length    = 3
  separator = "-"
  prefix    = var.environment
}
```

```powershell
git add main.tf
git commit -m "Increase random pet length to 3"
git push
```

**Minikube (manual run):** Start a **new plan** in Terrakube after push. The plan should show **replacement** of `random_pet.application` because `length` changed.

**If webhooks are configured** (production or tunnel): Terrakube may queue a job automatically; still prefer **Plan only** on push in training.

### Step 9 — Webhooks on Minikube (concept)

Terrakube can use **GitHub webhooks** to start jobs on push. GitHub must reach your Terrakube API over **public HTTPS**.

```text
GitHub ── POST webhook ──► https://<public-api-host>/...
```

On a laptop with `terrakube-api.minikube.net`, that hostname is **not** reachable from the internet. Options for advanced learners:

- **Manual runs** after `git push` (default for this course)  
- **Tunnel** (e.g. ngrok, Cloudflare Tunnel) to the Terrakube API—only with org approval and short-lived URLs  
- **Terrakube in cloud Kubernetes** with a public load balancer for real webhook labs

Configure workspace **VCS push template** to **Plan only** so pushes never auto-apply in class.

---

## Optional — GitHub VCS OAuth (organization-wide integration)

Use this when you need to **pick repositories from a list** or use **private** repos without SSH.

### Step 10 — Create a GitHub OAuth application

1. Open [GitHub Developer settings](https://github.com/settings/developers) → **OAuth Apps** → **New OAuth App**.  
2. Fill in:

| Field | Value |
|-------|--------|
| Application name | `Terrakube Training` |
| Homepage URL | `https://terrakube-ui.minikube.net` |
| Authorization callback URL | **Copy from Terrakube** when adding GitHub VCS (must match exactly) |

3. **Register application**.  
4. Copy **Client ID**; generate **Client Secret** and store it securely.

Do not use the homepage URL as the callback unless Terrakube documentation explicitly allows it.

### Step 11 — Register GitHub in Terrakube

Organization → **VCS** settings (or **Connect to a different VCS** during workspace creation):

| Field | Value |
|-------|--------|
| Name | `github-training` |
| Provider | GitHub |
| Client ID | From GitHub OAuth app |
| Client Secret | From GitHub OAuth app |

Save and complete **Authorize** with GitHub when prompted. Your Terrakube user must have permission to manage VCS settings.

### Step 12 — Recreate or update the workspace

Create (or edit) workspace **`terrakube-demo-dev`**:

| Setting | Value |
|---------|--------|
| VCS provider | `github-training` |
| Repository | `YOUR-USERNAME/terrakube-demo` |
| Branch | `main` |

Run **plan/apply** again to confirm OAuth cloning works.

---

## Optional — Private repository via SSH deploy key

Skip OAuth; suitable for **private** repos.

### Step A — Generate an SSH key (Windows)

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\terrakube_github" -C "terrakube-lab"
Get-Content "$env:USERPROFILE\.ssh\terrakube_github.pub"
```

### Step B — Add deploy key on GitHub

Repository → **Settings** → **Deploy keys** → **Add deploy key** → paste public key. **Do not** enable write access.

### Step C — Configure Terrakube

Upload the **private** key in organization **SSH keys** settings (per your Terrakube version’s UI).

### Step D — Workspace repository URL

Use SSH form:

```text
git@github.com:YOUR-USERNAME/terrakube-demo.git
```

Select the uploaded SSH key when creating the workspace. **Webhooks are separate**—SSH alone only fixes **clone**, not push triggers.

---

## Cleanup

- In Terrakube: delete workspace **`terrakube-demo-dev`** if no longer needed (destroys or orphan state per platform behavior—follow UI warnings).  
- On GitHub: delete repository **`terrakube-demo`** or make it private and remove deploy keys/OAuth app used for training.  
- Revoke **OAuth Client Secret** if the training app is discarded.  
- Remove `labs/work/lab-16/` locally.

---

## Lab completion checklist

- [ ] GitHub repo **`terrakube-demo`** contains `main.tf` and `.gitignore`  
- [ ] Terrakube workspace **`terrakube-demo-dev`** uses **Version Control** on branch `main`  
- [ ] **VCS** connected via public URL, **OAuth**, or **SSH** (one method completed)  
- [ ] Remote **plan** shows `random_pet.application` create  
- [ ] Remote **apply** succeeds; **output** and **state** visible in UI  
- [ ] Git commit (`length = 3`) followed by new **plan** showing **replace**  
- [ ] Understand why **webhooks** may not work on Minikube without a public endpoint  

## Self-review questions

| Topic | Question |
|-------|----------|
| VCS vs auth | What is the difference between GitHub-as-VCS and GitHub-as-login? |
| State | Where is state after Terrakube apply vs local `terraform apply`? |
| Webhooks | Why can’t GitHub call `terrakube-api.minikube.net` from the internet? |
| OAuth callback | Why must the GitHub callback URL match Terrakube’s setting exactly? |
| SSH | What does a deploy key enable that it does not enable? |
| vs Lab 14 | How is a Terrakube job similar to an HCP remote run? |

## Training delivery note

**Recommended order in class:** (1) public repo + manual runs (Steps 1–8), (2) GitHub OAuth (Steps 10–12), (3) discuss webhooks and tunnels (Step 9), (4) optional SSH for private repos.

## Previous lab

[Lab 15 — Install and Configure Terrakube](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md).

## Next lab

[Lab 17 — Terrakube Workspaces and Remote Execution](./Lab%2017%20-%20Terrakube%20Workspaces%20and%20Remote%20Terraform%20Execution.md): workspace variables, dual workspaces, plan/apply only on the executor, remote state.

## Reference documentation

- [Creating workspaces](https://docs.terrakube.io/user-guide/workspaces/creating-workspaces)  
- [Terrakube documentation](https://docs.terrakube.io/)  
- [GitHub OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app)