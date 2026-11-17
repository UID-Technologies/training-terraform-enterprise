# Lab 01 — Create an Azure Resource Group with Terraform

## Description

In this lab you configure Terraform for Azure, define required providers, and create your first resource: an **Azure Resource Group**. A resource group is a logical container for Azure resources that share lifecycle, permissions, and billing tags.

## Prerequisites

- Completed [labs README](./README.md) global prerequisites
- Successful `az login` and `az account show` pointing at the intended subscription
- Empty or dedicated folder for this lab (this directory)

## Use case / scenario

Contoso’s platform team wants all “dev” networking experiments isolated in one named container so cost and cleanup are predictable. You will provision `rg-terraform-lab-01` (with your own suffix) in `eastus` using Infrastructure as Code so the same definition can be reviewed in git and reapplied in other environments.

## Topic explanation

### What is a resource group?

- Scope for RBAC, policies, and cost analysis
- Resources belong to exactly one resource group (most resource types)
- Deleting a resource group deletes contained resources (use carefully)

### Terraform concepts used

| Concept | Purpose in this lab |
|--------|---------------------|
| `terraform` block | Pins Terraform and provider versions |
| `provider "azurerm"` | Configures Azure API access |
| `resource "azurerm_resource_group"` | Declares the RG you want Azure to match |
| `terraform init` | Downloads provider plugins |
| `terraform plan` | Shows planned changes without applying |
| `terraform apply` | Creates or updates real infrastructure |

### Provider authentication

The provider uses your Azure CLI login. No secrets are stored in `.tf` files for this lab.

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-01/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Sign in to Azure

```powershell
az login
az account set --subscription "<your-subscription-name-or-id>"
az account show --output table
```

### Step 2 — Create Terraform files in this lab folder

**`providers.tf`**

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.110"
    }
  }
}

provider "azurerm" {
  features {}
}
```

**`variables.tf`**

```hcl
variable "location" {
  type        = string
  description = "Azure region for the resource group"
  default     = "eastus"
}

variable "resource_group_name" {
  type        = string
  description = "Name of the resource group (must be globally unique in your subscription context)"
  default     = "rg-terraform-lab-01-demo"
}
```

**`main.tf`**

```hcl
resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment = "lab"
    lab         = "01-resource-group"
    managed_by  = "terraform"
  }
}
```

**`outputs.tf`**

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.lab.name
}

output "resource_group_id" {
  value = azurerm_resource_group.lab.id
}

output "location" {
  value = azurerm_resource_group.lab.location
}
```

**Action:** Edit `resource_group_name` in `variables.tf` or pass `-var` so the name is unique for your team (e.g. `rg-terraform-lab-01-jd`).

### Step 3 — Initialize Terraform

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-01" -Force | Out-Null
Set-Location "labs\work\lab-01"
terraform init
```

Expected: `Terraform has been successfully initialized!`

### Step 4 — Validate and format

```powershell
terraform fmt
terraform validate
```

### Step 5 — Plan

```powershell
terraform plan
```

Review: **1 to add**, `azurerm_resource_group.lab`.

### Step 6 — Apply

```powershell
terraform apply
```

Type `yes` when prompted. Note the outputs.

### Step 7 — Verify in Azure

```powershell
az group show --name <your-resource-group-name> --output table
```

Or open the Azure Portal → Resource groups → find your group and tags.

### Step 8 — Optional: change tags

Add a tag in `main.tf`, run `terraform plan` again, and apply. Observe **in-place update** (no recreation).

---

## Cleanup

Destroy only what this lab created:

```powershell
terraform destroy
```

Confirm with `yes`. Verify:

```powershell
az group exists --name <your-resource-group-name>
```

Should return `false`.

**Important:** Keep this lab’s patterns (`providers.tf`, tagging, variables) — Lab 02 reuses the same provider setup.

## Lab completion checklist

- [ ] `terraform init` / `plan` / `apply` succeeded
- [ ] Resource group visible in Portal with expected tags
- [ ] `terraform destroy` completed and group is gone

## Next lab

Continue to [Lab 02 — Virtual Network, Subnet, NSG, and Public IP](./Lab 02 - Virtual Network, Subnet, NSG, and Public IP.md).