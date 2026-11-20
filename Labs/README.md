# Cloud Infrastructure with Terraform — Hands-On Labs

Step-by-step exercises for **Azure** (Labs 01–05), **AWS** (Labs 06–10), **modules** (Lab 09), **remote state** (Lab 10), **Git integration** (Lab 11), **enterprise practices** (Lab 12), **HCP Terraform** (Labs 13–14), **Terrakube** (Labs 15–17), **GitHub Actions CI/CD** (Lab 18), **secrets security** (Lab 19), and **Terrakube private registry + S3** (Lab 20) with [Terraform](https://www.terraform.io/).

## How to use these labs

1. Open the lab guide (`.md` file) listed below—all guides live **directly in this `labs` folder**.
2. From **repository root** (the folder that contains `labs`), create each lab’s root folder: **`labs/work/lab-NN/`** as described in the guide.
3. Copy or type each Terraform file from the lab guide into that root folder.
4. Run `terraform init`, `plan`, and `apply` from the lab root folder; **`terraform destroy`** when you finish.

## Lab sequence

| Lab | Guide | Cloud | Topic | Est. time |
|-----|--------|-------|--------|-----------|
| 01 | [Lab 01 - Create an Azure Resource Group with Terraform](./Lab%2001%20-%20Create%20an%20Azure%20Resource%20Group%20with%20Terraform.md) | Azure | Provider setup, `azurerm_resource_group` | 30–45 min |
| 02 | [Lab 02 - Virtual Network, Subnet, NSG, and Public IP](./Lab%2002%20-%20Virtual%20Network,%20Subnet,%20NSG,%20and%20Public%20IP.md) | Azure | VNet, subnet, NSG, public IP | 60–90 min |
| 03 | [Lab 03 - Linux Virtual Machine with VNet Integration](./Lab%2003%20-%20Linux%20Virtual%20Machine%20with%20VNet%20Integration.md) | Azure | Linux VM, SSH, VNet | 90–120 min |
| 04 | [Lab 04 - Windows Virtual Machine with VNet Integration](./Lab%2004%20-%20Windows%20Virtual%20Machine%20with%20VNet%20Integration.md) | Azure | Windows VM, RDP, VNet | 90–120 min |
| 05 | [Lab 05 - Deploy a Container with Azure Container Instances](./Lab%2005%20-%20Deploy%20a%20Container%20with%20Azure%20Container%20Instances.md) | Azure | ACI, container group | 60–90 min |
| 06 | [Lab 06 - VPC, Subnets, Internet Gateway, Route Tables, Security Groups, Elastic IP (AWS)](./Lab%2006%20-%20VPC,%20Subnets,%20Internet%20Gateway,%20Route%20Tables,%20Security%20Groups,%20Elastic%20IP%20(AWS).md) | AWS | VPC, IGW, routes, SG, EIP | 90–120 min |
| 07 | [Lab 07 - EC2 Instance with VPC, Security Group, and Elastic IP (AWS)](./Lab%2007%20-%20EC2%20Instance%20with%20VPC,%20Security%20Group,%20and%20Elastic%20IP%20(AWS).md) | AWS | EC2, EIP | 90–120 min |
| 08 | [Lab 08 - IAM Roles, Instance Profile, and EC2 (AWS)](./Lab%2008%20-%20IAM%20Roles,%20Instance%20Profile,%20and%20EC2%20(AWS).md) | AWS | IAM role, SSM-ready EC2 | 90–120 min |
| 09 | [Lab 09 - Terraform Modules Local Registry Versioning and Design](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md) | AWS / Terraform | Local & registry modules, versioning, design standards | 90–120 min |
| 10 | [Lab 10 - Remote State with S3 Backend (AWS)](./Lab%2010%20-%20Remote%20State%20with%20S3%20Backend%20(AWS).md) | AWS | S3 backend, DynamoDB locking, state migration | 90–120 min |
| 11 | [Lab 11 - Git Integration Branching Pull Requests and Versioning](./Lab%2011%20-%20Git%20Integration%20Branching%20Pull%20Requests%20and%20Versioning.md) | Git / Terraform | Branching, PRs, repo layout, semver tags | 60–90 min |
| 12 | [Lab 12 - Enterprise Best Practices AWS Modules and Multi Environment](./Lab%2012%20-%20Enterprise%20Best%20Practices%20AWS%20Modules%20and%20Multi%20Environment.md) | AWS | Folder structure, naming/tags, SSM secrets, Dev–Production modules | 120–150 min |
| 13 | [Lab 13 - HCP Terraform Workspaces Variables and Variable Sets](./Lab%2013%20-%20HCP%20Terraform%20Workspaces%20Variables%20and%20Variable%20Sets.md) | HCP / AWS | Workspace architecture, variables, variable sets, sensitive & env vars | 90–120 min |
| 14 | [Lab 14 - HCP Terraform Remote Execution Plans Approval and Cost](./Lab%2014%20-%20HCP%20Terraform%20Remote%20Execution%20Plans%20Approval%20and%20Cost.md) | HCP / AWS | Remote runs, speculative plans, apply approval, cost estimate, run history | 90–120 min |
| 15 | [Lab 15 - Install and Configure Terrakube for Terraform Automation](./Lab%2015%20-%20Install%20and%20Configure%20Terrakube%20for%20Terraform%20Automation.md) | Kubernetes / Terrakube | Minikube, Helm, HTTPS, workspaces, remote Terraform | 3–4 hr |
| 16 | [Lab 16 - Connect Terrakube with GitHub](./Lab%2016%20-%20Connect%20Terrakube%20with%20GitHub.md) | Terrakube / GitHub | VCS workflow, OAuth, SSH, plan/apply from Git | 90–120 min |
| 17 | [Lab 17 - Terrakube Workspaces and Remote Terraform Execution](./Lab%2017%20-%20Terrakube%20Workspaces%20and%20Remote%20Terraform%20Execution.md) | Terrakube | Workspaces, variables, remote plan/apply, dual env state | 60–90 min |
| 18 | [Lab 18 - Terraform CI CD with GitHub Actions and Terrakube](./Lab%2018%20-%20Terraform%20CI%20CD%20with%20GitHub%20Actions%20and%20Terrakube.md) | GitHub / Terrakube | Actions CI, env approvals, Terrakube authoritative apply | 2.5–3 hr |
| 19 | [Lab 19 - Terraform Security and AWS Secrets Manager](./Lab%2019%20-%20Terraform%20Security%20and%20AWS%20Secrets%20Manager.md) | AWS | Secrets Manager, IAM, state security, optional Actions OIDC | 90–120 min |
| 20 | [Lab 20 - Terrakube Private Module Registry and AWS S3 Bucket](./Lab%2020%20-%20Terrakube%20Private%20Module%20Registry%20and%20AWS%20S3%20Bucket.md) | Terrakube / AWS | Private registry module, workspace, AWS creds in env vars, S3 apply | 120–150 min |

Complete **Labs 01–05 (Azure)** before **AWS Labs 06–10**. **Labs 09–20** build through enterprise workflow, HCP, Terrakube, CI/CD, security, and private modules on AWS.

## Global prerequisites

**All labs**

- [Terraform](https://developer.hashicorp.com/terraform/install) 1.5+ (`terraform version`)
- Text editor or IDE (VS Code recommended)

**Azure Labs 01–05**

- Azure subscription; [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (`az login`)
- Default region: `eastus`

**AWS Labs 06–08**

- AWS account; [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) (`aws sts get-caller-identity`)
- Credentials via `aws configure` or `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
- Default region: `us-east-1`

**Labs 09–10 (AWS)**

- AWS credentials; **`aws sts get-caller-identity`**  
- Lab 09: S3 create/delete; Terraform Registry access  
- Lab 10: S3 state bucket + DynamoDB table + `terraform init -migrate-state`

**Lab 11 (Git)**

- **Git** installed; optional **GitHub** account and **`gh`** CLI for a live pull request  
- Lab 11 uses `terraform validate` only unless you optionally apply from `environments/dev`

**Lab 12 (Enterprise AWS)**

- AWS credentials; S3 and SSM Parameter Store access  
- Apply in **`environments/dev`** only; **`plan`** in qa, uat, production  
- Optional: S3 remote backend from Lab 10 (one state key per environment)

**Lab 13 (HCP Terraform)**

- Free [HCP Terraform](https://app.terraform.io) organization; **`terraform login`**  
- AWS credentials configured as **workspace environment variables** for remote runs  
- S3 create/delete in workspace **`northwind-audit-dev`**

**Lab 14 (Remote execution)**

- Same HCP org as Lab 13; workspace **`northwind-audit-execution`** (or reuse dev workspace)  
- **Manual apply** enabled; optional **GitHub VCS** for speculative plans  
- Optional org **cost estimation** (admin)

**Lab 15 (Terrakube)**

- Windows 11, **Docker Desktop**, **Minikube**, **kubectl**, **Helm**, **mkcert**  
- **16 GB RAM** recommended; no cloud subscription (uses `random` provider)  
- Official [Minikube + HTTPS](https://docs.terrakube.io/getting-started/deployment/minikube-+-https) `values.yaml` required

**Lab 16 (Terrakube + GitHub)**

- Completed **Lab 15**; Terrakube UI reachable  
- GitHub account; optional **OAuth app** for VCS provider integration  
- **Manual** Terrakube runs on Minikube; webhooks require a public API endpoint (conceptual)

**Lab 17 (Terrakube workspaces)**

- Completed **Labs 15–16**; GitHub VCS working  
- Repo **`terrakube-remote-execution`**; workspaces **`demo-development`** and **`demo-production`**  
- **No local `terraform apply`**—remote executor only for plan/apply state

**Lab 18 (GitHub Actions + Terrakube)**

- GitHub repo **`terraform-github-actions-lab`**; Actions enabled  
- Completed **Labs 15–17**; workspace **`github-actions-development`**  
- Understand **stateless** Actions plan vs **Terrakube** remote state

**Lab 19 (Secrets Manager)**

- AWS credentials; **`secretsmanager`**, **`iam:CreatePolicy`**  
- Unique **`lab_suffix`**; region **`us-east-1`** (course default)  
- Optional: GitHub OIDC role for **DescribeSecret** workflow

**Lab 20 (Terrakube registry + S3)**

- **Labs 15–16** Terrakube + GitHub VCS; **Lab 09** module patterns  
- IAM user keys only in Terrakube workspace **environment variables** (sensitive)  
- S3 create/delete; private module tag **`v1.0.0`** on GitHub

## Conventions

- Use a **unique name** per lab run (resource group, `name_prefix`, etc.).
- **Cost**: VMs, EC2, containers, and Elastic IPs can incur charges — run **`terraform destroy`** after each lab.
- **Lab root folder**: `labs/work/lab-NN/` relative to repository root (gitignored via `work/`).
- **State**: Local `terraform.tfstate` and `.terraform/` live in your lab root folder; see [`.gitignore`](./.gitignore).

## Getting help

- [Terraform Azure tutorials](https://developer.hashicorp.com/terraform/tutorials/azure-get-started) · [AzureRM provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform AWS tutorials](https://developer.hashicorp.com/terraform/tutorials/aws-get-started) · [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)