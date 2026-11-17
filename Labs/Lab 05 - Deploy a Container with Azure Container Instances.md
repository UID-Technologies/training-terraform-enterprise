# Lab 05 — Deploy a Container with Azure Container Instances

## Description

You run a **containerized** workload on Azure without managing VMs, using **Azure Container Instances (ACI)**. ACI is the managed, serverless container platform on Azure today (the legacy **Azure Container Service** product line evolved into **Azure Kubernetes Service**; this lab uses ACI for a focused Terraform exercise before the AWS track).

You define a resource group and an `azurerm_container_group` with a public HTTP endpoint, then verify the app in a browser.

## Prerequisites

- Completed [Lab 04](./Lab%2004%20-%20Windows%20Virtual%20Machine%20with%20VNet%20Integration.md) or equivalent Azure + Terraform experience
- `az login` with permission to create container groups and public DNS labels in your subscription
- Subscription quota for Container Instances in your region

## Use case / scenario

Tailwind Traders wants a **short-lived demo API** for a conference booth: one Linux container, public port 80, no cluster operations. Platform engineering chooses **ACI** plus Terraform so the demo can be created and torn down in minutes.

## Topic explanation

### Azure Container Instances

- **Pods-on-demand**: Azure schedules containers on shared infrastructure; you pay per vCPU/memory-second while the group runs.
- **Public IP**: With `ip_address_type = "Public"` and a `dns_name_label`, Azure exposes the container on the internet (suitable for labs only).
- **`azurerm_container_group`**: One Terraform resource describes OS type, containers, CPU/memory, ports, and optional registry credentials.

### Compared to VMs (Labs 03–04)

| | VM labs | This lab |
|--|---------|----------|
| Unit of deploy | OS + disks | Container image |
| Patch model | Guest OS | Image refresh |
| Typical use | RDP/SSH admin | Stateless HTTP services |

---

## Step-by-step instructions

> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-05/`** for all Terraform files. Run every `terraform` command from that folder.

### Step 1 — Sign in to Azure

```powershell
az login
az account set --subscription "<your-subscription-name-or-id>"
az account show --output table
```

### Step 2 — Create the lab root folder

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-05" -Force | Out-Null
Set-Location "labs\work\lab-05"
```

### Step 3 — Create Terraform files

Create these files in `labs/work/lab-05/`.

#### `providers.tf`

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

#### `variables.tf`

```hcl
variable "location" {
  type    = string
  default = "eastus"
}

variable "resource_group_name" {
  type    = string
  default = "rg-terraform-lab-05-demo"
}

variable "container_group_name" {
  type    = string
  default = "aci-lab-05-demo"
}

variable "dns_name_label" {
  type        = string
  description = "Globally unique DNS label for the container FQDN (lowercase, no spaces)"
  default     = "acilab05demo"
}
```

Set `dns_name_label` to a **unique** value (e.g. `acilab05-yourinitials`).

#### `main.tf`

```hcl
resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment = "lab"
    lab         = "05-container-instances"
    managed_by  = "terraform"
  }
}

resource "azurerm_container_group" "lab" {
  name                = var.container_group_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  os_type             = "Linux"
  ip_address_type     = "Public"
  dns_name_label      = var.dns_name_label

  container {
    name   = "hello"
    image  = "mcr.microsoft.com/azuredocs/aci-helloworld"
    cpu    = "0.5"
    memory = "1.5"

    ports {
      port     = 80
      protocol = "TCP"
    }
  }

  tags = azurerm_resource_group.lab.tags
}
```

#### `outputs.tf`

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.lab.name
}

output "fqdn" {
  value = azurerm_container_group.lab.fqdn
}

output "ip_address" {
  value = azurerm_container_group.lab.ip_address
}

output "url" {
  value = "http://${azurerm_container_group.lab.fqdn}"
}
```

### Step 4 — Initialize and apply

```powershell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Type `yes` when prompted. Provisioning usually completes in **1–3 minutes**.

### Step 5 — Verify

Open the `url` output in a browser, or:

```powershell
terraform output -raw url
Invoke-WebRequest -Uri (terraform output -raw url) -UseBasicParsing | Select-Object StatusCode, Content
```

```powershell
$rg = terraform output -raw resource_group_name
az container list --resource-group $rg --query "[].{name:name,state:instanceView.state,fqdn:ipAddress.fqdn}" -o table
```

### Step 6 — Optional exercises

1. Change the image to `nginx:latest` and add port `443` (discuss limits of single-container groups).
2. Set `ip_address_type = "Private"` and outline what else you would need (VNet integration, Application Gateway).
3. Add a second container block (sidecar) and observe `plan` changes.

---

## Cleanup

```powershell
terraform destroy
```

Confirm `yes`. Verify the resource group is removed in the Portal or:

```powershell
az group exists --name <your-resource-group-name>
```

## Lab completion checklist

- [ ] Container group shows **Running** in Portal  
- [ ] HTTP URL returns the sample hello page  
- [ ] `terraform destroy` removed the group and resource group  

## Next lab

Start the AWS track: [Lab 06 - VPC, Subnets, Internet Gateway, Route Tables, Security Groups, Elastic IP (AWS)](./Lab%2006%20-%20VPC,%20Subnets,%20Internet%20Gateway,%20Route%20Tables,%20Security%20Groups,%20Elastic%20IP%20(AWS).md).