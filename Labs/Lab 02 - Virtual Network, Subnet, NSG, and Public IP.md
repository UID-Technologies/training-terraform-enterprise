# Lab 02 — Virtual Network, Subnet, NSG, and Public IP

## Description

You build core **Azure networking** with Terraform: a virtual network (VNet), a subnet, a Network Security Group (NSG) with rules, and a **standalone public IP** (used by later VM labs). You learn how address space, subnet delegation, and security rules fit together before attaching compute.

## Prerequisites

- Completed [Lab 01](./Lab 01 - Create an Azure Resource Group with Terraform.md) (concepts) or equivalent Terraform + Azure CLI setup
- `az login` with rights to create networking resources
- Familiarity with CIDR notation (e.g. `10.0.0.0/16`, `10.0.1.0/24`)

## Use case / scenario

Fabrikam is preparing a small “DMZ + app” segment in Azure for a pilot API. Network architects require:

- One VNet with space for future subnets
- A **web** subnet (`10.0.1.0/24`) for upcoming VMs
- An NSG allowing **HTTPS (443)** from the internet and **denying** RDP/SSH from the internet at the subnet edge (VM-specific rules come in Labs 03–04)
- A **Standard** static public IP reserved for a future load balancer or single VM

You will codify this design in Terraform.

## Topic explanation

### Virtual network (VNet)

- Private network in Azure; isolated from other customers
- `address_space` defines overall IP range; subnets must fit inside it

### Subnet

- Segment of the VNet where NICs are placed
- Each subnet can be associated with one NSG

### Network Security Group (NSG)

- Stateful-ish filter on traffic (allow/deny) by protocol, port, source, destination
- Applied at subnet and/or NIC; effective rules are combination of both

### Public IP

- **Standard** SKU recommended for production-style labs (zones-capable, used with Standard load balancers/NICs)
- `allocation_method = "Static"` keeps the address stable across reboots

### Terraform dependencies

Terraform infers that the subnet depends on the VNet. Explicit `azurerm_subnet_network_security_group_association` links NSG to subnet.

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-02/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Navigate to the lab folder and set naming variables

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-02" -Force | Out-Null
Set-Location "labs\work\lab-02"
$env:TF_VAR_resource_group_name = "rg-terraform-lab-02-yourinitials"
```

Default names match the `variables.tf` you create in Step 2; override the resource group name above for uniqueness in shared subscriptions.

### Step 2 — Create Terraform files

Create these four files in your working directory (see note above). Provider pin matches Lab 01. `main.tf` defines the resource group, VNet, subnet, NSG, rules, public IP, and subnet–NSG association. `outputs.tf` exposes IDs and IPs for Labs 03–04.

**Network design in this lab:**

| Item | Value |
|------|--------|
| VNet | `10.0.0.0/16` |
| Subnet `snet-web` | `10.0.1.0/24` |
| NSG | Allow 443 inbound from Internet; deny 22 and 3389 from Internet |

#### Create `providers.tf`

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

#### Create `variables.tf`

```hcl
variable "location" {
  type        = string
  description = "Azure region"
  default     = "eastus"
}

variable "resource_group_name" {
  type        = string
  description = "Resource group for networking lab resources"
  default     = "rg-terraform-lab-02-demo"
}

variable "vnet_name" {
  type    = string
  default = "vnet-lab-02"
}

variable "vnet_address_space" {
  type    = list(string)
  default = ["10.0.0.0/16"]
}

variable "subnet_name" {
  type    = string
  default = "snet-web"
}

variable "subnet_prefix" {
  type    = string
  default = "10.0.1.0/24"
}

variable "nsg_name" {
  type    = string
  default = "nsg-snet-web"
}

variable "public_ip_name" {
  type    = string
  default = "pip-lab-02-standby"
}
```

#### Create `main.tf`

```hcl
resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location

  tags = {
    environment = "lab"
    lab         = "02-virtual-network"
    managed_by  = "terraform"
  }
}

resource "azurerm_virtual_network" "lab" {
  name                = var.vnet_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  address_space       = var.vnet_address_space

  tags = azurerm_resource_group.lab.tags
}

resource "azurerm_subnet" "web" {
  name                 = var.subnet_name
  resource_group_name  = azurerm_resource_group.lab.name
  virtual_network_name = azurerm_virtual_network.lab.name
  address_prefixes     = [var.subnet_prefix]
}

resource "azurerm_network_security_group" "web" {
  name                = var.nsg_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  tags = azurerm_resource_group.lab.tags
}

resource "azurerm_network_security_rule" "allow_https_inbound" {
  name                        = "AllowHTTPSInbound"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "443"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.lab.name
  network_security_group_name = azurerm_network_security_group.web.name
}

resource "azurerm_network_security_rule" "deny_ssh_inbound" {
  name                        = "DenySSHInbound"
  priority                    = 200
  direction                   = "Inbound"
  access                      = "Deny"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.lab.name
  network_security_group_name = azurerm_network_security_group.web.name
}

resource "azurerm_network_security_rule" "deny_rdp_inbound" {
  name                        = "DenyRDPInbound"
  priority                    = 210
  direction                   = "Inbound"
  access                      = "Deny"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "3389"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.lab.name
  network_security_group_name = azurerm_network_security_group.web.name
}

resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}

resource "azurerm_public_ip" "standby" {
  name                = var.public_ip_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  allocation_method   = "Static"
  sku                 = "Standard"

  tags = azurerm_resource_group.lab.tags
}
```

#### Create `outputs.tf`

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.lab.name
}

output "vnet_name" {
  value = azurerm_virtual_network.lab.name
}

output "vnet_id" {
  value = azurerm_virtual_network.lab.id
}

output "subnet_id" {
  value = azurerm_subnet.web.id
}

output "subnet_name" {
  value = azurerm_subnet.web.name
}

output "nsg_id" {
  value = azurerm_network_security_group.web.id
}

output "public_ip_name" {
  value = azurerm_public_ip.standby.name
}

output "public_ip_address" {
  value = azurerm_public_ip.standby.ip_address
}
```

### Step 3 — Initialize

```powershell
terraform init
terraform fmt
terraform validate
```

### Step 4 — Plan and apply

```powershell
terraform plan -out=tfplan
terraform apply tfplan
```

Expect **several resources to add** (resource group, vnet, subnet, nsg, rules, association, public ip).

### Step 5 — Verify in Azure CLI

```powershell
$rg = terraform output -raw resource_group_name
az network vnet list --resource-group $rg --output table
az network vnet subnet list --resource-group $rg --vnet-name $(terraform output -raw vnet_name) --output table
az network nsg list --resource-group $rg --output table
az network public-ip show --resource-group $rg --name $(terraform output -raw public_ip_name) --query ipAddress -o tsv
```

### Step 6 — Portal verification

In the Portal: Resource group → Virtual network → Subnets → check `snet-web` and NSG association. Open NSG → **Inbound security rules** and confirm allow 443 and deny 22/3389.

### Step 7 — Optional exercise

1. Add an **Allow** rule for your client IP on port `443` only (source `IPAddresses` = your `/32`).  
2. Run `terraform plan` and discuss why a narrow source is preferable to `*` for admin access.  
3. Revert or keep per instructor guidance.

---

## Cleanup

```powershell
terraform destroy
```

Type `yes`. Public IP and VNet are removed with the resource group.

**Tip:** If destroy fails due to a lock, check `az lock list -g <rg>`.

## Lab completion checklist

- [ ] Subnet address is inside VNet range
- [ ] NSG associated with subnet
- [ ] Public IP shows in outputs
- [ ] Resources destroyed after lab

## Next lab

[Lab 03 — Linux virtual machine in the VNet](./Lab 03 - Linux Virtual Machine with VNet Integration.md).