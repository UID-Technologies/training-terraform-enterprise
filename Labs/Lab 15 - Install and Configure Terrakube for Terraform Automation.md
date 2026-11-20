# Lab 15 — Install and Configure Terrakube for Terraform Automation (Local Kubernetes)

## Description

You install **Terrakube** on a **local Kubernetes** cluster (Minikube), configure **HTTPS**, **authentication**, and **storage**, access the **web UI**, and run **centralized Terraform plan/apply** from workspaces. The lab uses the **random** provider for hands-on Terraform work—**no AWS or Azure subscription** required. This complements [Lab 13](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md) and [Lab 14](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md) (HCP Terraform) by showing a **self-hosted** automation platform pattern.

**Estimated time:** 3–4 hours | **Level:** intermediate | **Host:** Windows 11 + Docker Desktop

## Prerequisites

- Completed [Lab 11](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md) (Git basics) and **Lab 13–14** or equivalent understanding of **workspaces** and **remote runs**  
- [Terraform](https://developer.hashicorp.com/terraform/install) 1.5+  
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with Kubernetes support  
- Administrator rights (for `hosts` file and optional `minikube tunnel`)  
- **16 GB RAM** and **4 CPUs** recommended (8 GB may be tight)  
- Optional: GitHub account for **Step 24** (push demo configuration to Git)

---

## Topic explanation

### Terrakube architecture

Terrakube runs Terraform operations on a **platform** instead of only on individual laptops.

```text
Developers / CI ──► Terrakube UI & API ──► Executor ──► Terraform CLI
                            │                    │
                            ├─ Auth (OIDC/Dex)     ├─ Providers
                            ├─ Database (metadata) └─ Run logs
                            └─ Object storage (state & artifacts)
                                      │
                               Kubernetes (Minikube)
```

| Component | Responsibility |
|-----------|----------------|
| **Terrakube UI** | Dashboard for orgs, workspaces, modules, access |
| **Terrakube API** | Organizations, workspaces, jobs, configuration |
| **Executor** | Runs Terraform init/plan/apply in isolated jobs |
| **Authentication** | Users and groups (Dex/OIDC in production; sample local auth in lab) |
| **Database** | Platform metadata |
| **State storage** | Terraform state and related artifacts (HTTPS endpoints) |
| **Kubernetes** | Hosts all platform services |

### How this lab differs from production

| Lab (Minikube) | Production |
|----------------|------------|
| mkcert TLS | Enterprise CA or managed certificates |
| Embedded DB/storage in chart defaults | External PostgreSQL and S3-compatible storage |
| Sample `admin@example.com` user | Corporate IdP (Azure AD, Okta, etc.) |
| Single-node cluster | Highly available Kubernetes |

Official reference: [Terrakube Minikube + HTTPS](https://docs.terrakube.io/getting-started/deployment/minikube-+-https) and the [Helm chart repository](https://github.com/terrakube-io/terrakube-helm-chart).

---

## Use case / scenario

**Adventure Works Training** wants a **self-hosted** Terraform automation option alongside HCP Terraform. Platform engineers must:

1. Install Terrakube with **Helm** on **Minikube**  
2. Expose UI/API/registry on **`*.minikube.net`** with **HTTPS**  
3. Run the **sample workspace**, then a **custom Git-backed** workspace  
4. Compare **local state** vs **Terrakube-managed state**

Your lab roots are **`labs/work/lab-15/terrakube-lab/`** (cluster install assets) and **`labs/work/lab-15/demo-project/`** (Terraform code).

---

## Step-by-step instructions

> **From repository root** (folder that contains `labs`), run all path commands below relative to that root. Lab work lives under **`labs/work/lab-15/`** (gitignored via `work/`).

### Step 1 — Create lab folders

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-15\terrakube-lab" -Force | Out-Null
New-Item -ItemType Directory -Path "labs\work\lab-15\demo-project" -Force | Out-Null
```

```text
labs/work/lab-15/
  terrakube-lab/     # certs, Helm values, rendered manifests
  demo-project/      # Terraform for Git + remote runs
```

### Step 2 — Install Docker Desktop

Install [Docker Desktop](https://www.docker.com/products/docker-desktop/). Enable the **WSL 2** backend if prompted. Start Docker Desktop, then verify:

```powershell
docker --version
docker info
```

**Expected:** version prints and `docker info` reports a running server without errors.

### Step 3 — Install Minikube

```powershell
winget install Kubernetes.minikube
```

Restart PowerShell, then:

```powershell
minikube version
```

### Step 4 — Install kubectl

```powershell
winget install Kubernetes.kubectl
kubectl version --client
```

`kubectl` talks to the Kubernetes API (create pods, inspect Ingress, troubleshoot).

### Step 5 — Install Helm

```powershell
winget install Helm.Helm
helm version
```

Helm installs the **Terrakube** chart and dependencies.

### Step 6 — Install mkcert (HTTPS)

```powershell
winget install FiloSottile.mkcert
mkcert -version
mkcert -install
```

`mkcert -install` trusts a **local development CA** on your machine only. Do not share the CA private key.

### Step 7 — Start Minikube

Ensure Docker Desktop is running:

```powershell
minikube start --driver=docker --cpus=4 --memory=8192
minikube status
kubectl get nodes
```

**Expected:** node status **Ready**. Lower `--cpus` / `--memory` only if the host cannot allocate 4/8192.

### Step 8 — Enable Kubernetes addons

```powershell
minikube addons enable ingress
minikube addons enable storage-provisioner
minikube addons enable default-storageclass
kubectl get pods -n ingress-nginx
kubectl get storageclass
```

**Ingress** routes hostnames (`terrakube-ui.minikube.net`) to services instead of publishing many NodePorts.

### Step 9 — Create the Terrakube namespace

```powershell
kubectl create namespace terrakube
kubectl get namespaces
```

Confirm `terrakube` is **Active**.

### Step 10 — Generate TLS certificates

```powershell
Set-Location "labs\work\lab-15\terrakube-lab"
mkcert -key-file key.pem -cert-file cert.pem minikube.net "*.minikube.net"
```

**Expected files:** `cert.pem`, `key.pem` in `labs/work/lab-15/terrakube-lab/`.

### Step 11 — Create the Kubernetes TLS secret

```powershell
kubectl create secret tls terrakube-tls --cert=cert.pem --key=key.pem -n terrakube
kubectl get secret terrakube-tls -n terrakube
```

### Step 12 — Configure Windows hostnames

```powershell
minikube ip
```

Note the IP (example: `192.168.49.2`). Edit **`C:\Windows\System32\drivers\etc\hosts`** as Administrator and add (replace with your Minikube IP):

```text
192.168.49.2 terrakube-ui.minikube.net
192.168.49.2 terrakube-api.minikube.net
192.168.49.2 terrakube-reg.minikube.net
```

```powershell
ping terrakube-ui.minikube.net
```

**Windows + Docker driver note:** if the Minikube IP is not reachable from Windows, run **`minikube tunnel`** in a separate **Administrator** PowerShell session and test HTTPS again. Do not assume hosts file alone fixes routing.

### Step 13 — Add the Terrakube Helm repository

```powershell
helm repo add terrakube-repo https://terrakube-io.github.io/terrakube-helm-chart
helm repo update
helm search repo terrakube-repo/terrakube
```

### Step 14 — Download chart defaults for reference

```powershell
helm show values terrakube-repo/terrakube > values-reference.yaml
helm show chart terrakube-repo/terrakube
```

Open `values-reference.yaml` and note chart **version**—you will match the official Minikube example to that version.

### Step 15 — Prepare `values.yaml` (critical step)

Do **not** install with empty defaults and expect HTTPS, auth, and remote state to work.

1. Open [Terrakube Minikube + HTTPS](https://docs.terrakube.io/getting-started/deployment/minikube-+-https).  
2. From the [terrakube-helm-chart](https://github.com/terrakube-io/terrakube-helm-chart) repository, copy the **Minikube HTTPS** `values` example for your chart version.  
3. Save as **`labs/work/lab-15/terrakube-lab/values.yaml`**.

Your file must align at minimum with:

| Setting | Lab value |
|---------|-----------|
| UI hostname | `terrakube-ui.minikube.net` |
| API hostname | `terrakube-api.minikube.net` |
| Registry hostname | `terrakube-reg.minikube.net` |
| TLS secret name | `terrakube-tls` (namespace `terrakube`) |
| Auth / Dex | Per official sample (enables sample admin user) |
| Database & storage | Per official Minikube example |

Instructors: verify this step on the chart version **before** class; chart keys change between releases.

### Step 16 — Lint and render the chart

```powershell
helm lint terrakube-repo/terrakube -f .\values.yaml
helm template terrakube terrakube-repo/terrakube -n terrakube -f .\values.yaml > rendered.yaml
```

Review `rendered.yaml` to see Ingress, Deployments, and PVCs Helm will create.

### Step 17 — Install Terrakube

```powershell
helm install terrakube terrakube-repo/terrakube `
  --namespace terrakube `
  --values .\values.yaml `
  --wait `
  --timeout 15m
helm list -n terrakube
```

### Step 18 — Verify Kubernetes resources

```powershell
kubectl get pods -n terrakube
kubectl get svc -n terrakube
kubectl get ingress -n terrakube
kubectl get pvc -n terrakube
```

Wait until application pods are **Running** / **Ready** and PVCs are **Bound** where used.

If a pod fails:

```powershell
kubectl describe pod <pod-name> -n terrakube
kubectl logs <pod-name> -n terrakube
```

### Step 19 — Access the Terrakube UI

Open: **https://terrakube-ui.minikube.net**

If the official Minikube sample auth is deployed, documented sample credentials are often:

| Field | Value |
|-------|-------|
| Username | `admin@example.com` |
| Password | `admin` |

These apply **only** when the sample Dex/static user configuration from the official Minikube guide is present—not for every Terrakube install.

**Checkpoint:** HTTPS loads, sign-in works, you can open **Organizations** and **Workspaces**.

### Step 20 — Explore the sample workspace

In the UI:

1. Organization **`simple`** → workspace **`sample_simple`**  
2. Review Terraform configuration, variables, execution settings, and **job history**  
3. Start a **new run** (plan/apply per workspace policy)  

Observe the job lifecycle: **queued → init → plan → (approval) → apply → finished**.

In the job log, locate: provider download, plan output, apply result, final status.

### Step 21 — Create the demo Terraform project locally

```powershell
Set-Location "labs\work\lab-15\demo-project"
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
  default = "development"
}

resource "random_pet" "server" {
  length    = 2
  separator = "-"
  prefix    = var.environment
}

output "server_name" {
  value = random_pet.server.id
}
```

### Step 22 — Validate locally (before Terrakube)

```powershell
terraform init
terraform validate
terraform plan
terraform apply
terraform output
```

Example output shape: `server_name = "development-happy-tiger"` (name will vary).

This creates **local state** on your laptop—separate from Terrakube state in later steps.

### Step 23 — Clean local state before remote runs (recommended)

After you understand local behavior, remove local state so it does not confuse the remote workflow:

```powershell
terraform destroy
Remove-Item -Recurse -Force .terraform -ErrorAction SilentlyContinue
Remove-Item terraform.tfstate* -ErrorAction SilentlyContinue
```

Add **`labs/work/lab-15/demo-project/.gitignore`** when using Git:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
.terraform.lock.hcl
```

### Step 24 — Push demo configuration to Git

Create an empty GitHub repository (e.g. `terrakube-training-demo`), then from `demo-project`:

```powershell
git init
git add main.tf .gitignore
git commit -m "Initial Terrakube demo"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/terrakube-training-demo.git
git push -u origin main
```

Replace `YOUR-USERNAME` with your GitHub account. Never commit secrets or state files.

### Step 25 — Connect Git to Terrakube

In the Terrakube UI:

1. Open your training organization (or create one).  
2. Configure a **Git provider** connection if missing.  
3. Create workspace **`terraform-demo-dev`**.  
4. Select the repository, branch **`main`**, working directory **`.`** (repo root).  
5. Choose a Terraform version compatible with `>= 1.5.0`.  
6. Save the workspace.

Exact menus depend on your Terrakube version.

### Step 26 — Remote plan and apply

From workspace **`terraform-demo-dev`**:

1. Start a **plan** run.  
2. Confirm plan creates `random_pet.server`.  
3. **Apply** if the workspace allows it.  
4. Inspect **remote state** and **outputs** in the UI.

The pet name may differ from your earlier local apply—Terrakube holds a **new state file**.

### Step 27 — Git-driven change and second run

Update `main.tf`:

```hcl
resource "random_pet" "server" {
  length    = 3
  separator = "-"
  prefix    = var.environment
}
```

```powershell
git add main.tf
git commit -m "Change server naming format"
git push
```

Trigger another Terrakube run. The plan should show **replacement** of `random_pet.server` because `length` changed.

### Step 28 — Troubleshooting reference

| Problem | What to check |
|---------|----------------|
| Pod **Pending** | CPU/memory requests, PVC binding, `kubectl describe pod` |
| **ImagePullBackOff** | Image name, registry access, network |
| UI not reachable | Ingress, hosts file, `minikube tunnel` |
| HTTPS certificate error | mkcert SANs (`*.minikube.net`), secret name/namespace |
| Authentication fails | Dex config in `values.yaml`, sample user enabled |
| Terraform job fails | Executor logs, workspace variables, provider access |
| Remote state errors | API/registry URLs, TLS trust, storage config |

```powershell
kubectl get events -n terrakube --sort-by=.lastTimestamp
kubectl describe ingress -n terrakube
helm status terrakube -n terrakube
```

---

## Cleanup

```powershell
Set-Location "labs\work\lab-15\terrakube-lab"
helm uninstall terrakube -n terrakube
```

Review remaining PVCs and secrets; delete the namespace if appropriate:

```powershell
kubectl delete namespace terrakube
```

Optional full cluster teardown:

```powershell
minikube delete
```

Remove `terrakube-*.minikube.net` entries from the Windows `hosts` file.

Delete `labs/work/lab-15/` when finished.

---

## Lab completion checklist

- [ ] Minikube cluster **Ready**; Ingress and storage addons enabled  
- [ ] TLS secret `terrakube-tls` created in namespace `terrakube`  
- [ ] **`values.yaml`** from official Minikube HTTPS guide (version-matched)  
- [ ] Terrakube installed with Helm; pods **Ready**  
- [ ] UI opens at **https://terrakube-ui.minikube.net**; authentication works  
- [ ] Sample workspace **`sample_simple`** run completed; logs reviewed  
- [ ] Demo project in `labs/work/lab-15/demo-project/` validated locally  
- [ ] Git repository connected; workspace **`terraform-demo-dev`** created  
- [ ] Remote **plan/apply** and **state** inspected in UI  
- [ ] Git push triggered a second run with **replace** in plan  

## Self-review questions

| Topic | Question |
|-------|----------|
| Architecture | Which component runs `terraform plan`? |
| vs HCP (Lab 13–14) | What does Terrakube self-host that HCP hosts for you? |
| HTTPS | Why does Terrakube require HTTPS for remote state/registry in many setups? |
| State | Why did the pet name differ between local apply and Terrakube apply? |
| Helm | Why must `values.yaml` match chart version? |
| Production | Name three external dependencies you would add for production |

## Instructor delivery schedule (optional)

| Session | Topic | Duration |
|---------|--------|----------|
| 1 | Architecture and prerequisites | 30 min |
| 2 | Minikube, Ingress, HTTPS (Steps 7–12) | 45 min |
| 3 | Helm install and troubleshooting (Steps 13–18) | 45 min |
| 4 | UI, auth, sample workspace (Steps 19–20) | 30 min |
| 5 | Git integration and remote runs (Steps 21–27) | 45 min |
| 6 | State, troubleshooting, cleanup | 25 min |
| **Total** | | **~3 h 40 min** |

**Instructor note:** Step 15 (`values.yaml`) is the highest-failure step. Pre-validate the official Minikube example against the chart version installed in class.

## Previous lab

[Lab 14 — HCP Terraform Remote Execution](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md).

## Next lab

[Lab 16 — Connect Terrakube with GitHub](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md): Version Control workspaces, GitHub OAuth or SSH, remote plan/apply, and webhooks on Minikube.

## Reference documentation

- [Terrakube Minikube + HTTPS](https://docs.terrakube.io/getting-started/deployment/minikube-+-https)  
- [terrakube-helm-chart](https://github.com/terrakube-io/terrakube-helm-chart)  
- [Terrakube documentation](https://docs.terrakube.io/)