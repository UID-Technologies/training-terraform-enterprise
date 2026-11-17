# Lab 04 — Windows Virtual Machine with VNet Integration

## Description

You deploy **Windows Server** on Azure using Terraform: resource group, VNet, subnet, NSG with **RDP (3389)** restricted to your IP, public IP, NIC, and `azurerm_windows_virtual_machine` with password authentication (typical for Windows admin in lab environments).

## Prerequisites

- Completed [Lab 03](./Lab 03 - Linux Virtual Machine with VNet Integration.md) or equivalent VM networking knowledge
- Azure subscription quota for a small Windows VM (`Standard_B2s` default — Windows often needs slightly more headroom than Linux)
- Windows client with **Remote Desktop Connection** (`mstsc.exe`)
- Strong password meeting Azure complexity rules (see Step 2)

## Use case / scenario

Tailspin Traders runs a legacy .NET utility on Windows Server. For a migration spike, architects want a disposable Windows VM:

- Joined logically to the same style **web subnet** (`10.0.2.0/24` in this lab to distinguish from Linux lab segments)  
- Reachable via **RDP** only from the administrator’s office IP  
- Fully defined in Terraform for repeatable teardown after the spike  

## Topic explanation

### Windows VM resource

`azurerm_windows_virtual_machine` manages:

- OS image (Windows Server 2022 Datacenter Azure Edition)  
- VM size, hostname, admin username/password  
- OS disk type and caching  

Passwords are **sensitive**; mark variables `sensitive = true` and pass via `terraform.tfvars` (never commit real passwords).

### RDP security

- NSG allows `3389` from `allowed_rdp_cidr` only  
- Public IP exposes the VM — acceptable for training, not for production without hardening (JIT access, Bastion, etc.)

### Licensing

Windows Server images include Azure Hybrid Benefit eligibility in enterprise scenarios; this lab uses default pay-as-you-go gallery image.

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-04/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Navigate to the lab folder

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-04" -Force | Out-Null
Set-Location "labs\work\lab-04"
```


### Step 2 — Create Terraform files

Create these five files in your working directory (see note above). The stack mirrors Lab 03 but uses subnet `10.0.2.0/24`, `azurerm_windows_virtual_machine`, and an NSG rule for **RDP (3389)**.

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
  type    = string
  default = "eastus"
}

variable "resource_group_name" {
  type    = string
  default = "rg-terraform-lab-04-demo"
}

variable "vnet_name" {
  type    = string
  default = "vnet-lab-04"
}

variable "subnet_name" {
  type    = string
  default = "snet-windows"
}

variable "vm_name" {
  type    = string
  default = "vm-win-lab04"
}

variable "vm_size" {
  type    = string
  default = "Standard_B2s"
}

variable "admin_username" {
  type    = string
  default = "azureadmin"
}

variable "admin_password" {
  type        = string
  sensitive   = true
  description = "Local admin password — set in terraform.tfvars"
}

variable "allowed_rdp_cidr" {
  type        = string
  description = "CIDR allowed for RDP (your public IP/32)"
  default     = "0.0.0.0/0"
}
```

#### Create `main.tf`

```hcl
locals {
  tags = {
    environment = "lab"
    lab         = "04-windows-vm"
    managed_by  = "terraform"
  }
}

resource "azurerm_resource_group" "lab" {
  name     = var.resource_group_name
  location = var.location
  tags     = local.tags
}

resource "azurerm_virtual_network" "lab" {
  name                = var.vnet_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  address_space       = ["10.0.0.0/16"]
  tags                = local.tags
}

resource "azurerm_subnet" "windows" {
  name                 = var.subnet_name
  resource_group_name  = azurerm_resource_group.lab.name
  virtual_network_name = azurerm_virtual_network.lab.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_network_security_group" "vm" {
  name                = "nsg-${var.vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  tags                = local.tags
}

resource "azurerm_network_security_rule" "rdp" {
  name                        = "AllowRDPFromAdmin"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "3389"
  source_address_prefix       = var.allowed_rdp_cidr
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.lab.name
  network_security_group_name = azurerm_network_security_group.vm.name
}

resource "azurerm_subnet_network_security_group_association" "windows" {
  subnet_id                 = azurerm_subnet.windows.id
  network_security_group_id = azurerm_network_security_group.vm.id
}

resource "azurerm_public_ip" "vm" {
  name                = "pip-${var.vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = local.tags
}

resource "azurerm_network_interface" "vm" {
  name                = "nic-${var.vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.windows.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.vm.id
  }

  tags = local.tags
}

resource "azurerm_windows_virtual_machine" "lab" {
  name                = var.vm_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  size                = var.vm_size
  admin_username      = var.admin_username
  admin_password      = var.admin_password

  network_interface_ids = [
    azurerm_network_interface.vm.id,
  ]

  os_disk {
    name                 = "${var.vm_name}-osdisk"
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-datacenter-azure-edition"
    version   = "latest"
  }

  vm_agent_platform_updates_enabled = true

  tags = local.tags
}
```

#### Create `outputs.tf`

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.lab.name
}

output "vm_name" {
  value = azurerm_windows_virtual_machine.lab.name
}

output "admin_username" {
  value = var.admin_username
}

output "private_ip_address" {
  value = azurerm_network_interface.vm.private_ip_address
}

output "public_ip_address" {
  value = azurerm_public_ip.vm.ip_address
}

output "rdp_hint" {
  value = "mstsc /v:${azurerm_public_ip.vm.ip_address}"
}
```

#### Create `terraform.tfvars.example`

```hcl
resource_group_name = "rg-terraform-lab-04-yourinitials"
admin_password      = "ChangeMe-Lab-Strong-P@ssw0rd"
allowed_rdp_cidr    = "203.0.113.10/32"
```

Copy the example to your working variables file and edit secrets and naming:

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

- Set unique `resource_group_name`  
- Set `admin_password` (min 12 chars, upper, lower, digit, special — Azure enforces complexity)  
- Set `allowed_rdp_cidr` to your public IP `/32` (find your IP: `curl -s ifconfig.me` or search “what is my IP”)  

**Do not commit `terraform.tfvars`.** A `.gitignore` in `labs/` excludes it.

### Step 3 — Deploy

```powershell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Provisioning often takes **10–15 minutes** for Windows.

### Step 4 — Connect with RDP

```powershell
mstsc /v:$(terraform output -raw public_ip_address)
```

- Username: `.\<admin_username>` (default `azureadmin`) or `azureadmin` as shown in output  
- Password: value from your `terraform.tfvars`  

Accept certificate warning for lab VMs if prompted.

### Step 5 — On the VM

1. Open Server Manager — confirm OS edition  
2. Run `ipconfig` — private address should be `10.0.2.x`  
3. Optional: install IIS via Server Manager for a visible “success” marker  

### Step 6 — Verify with Azure CLI

```powershell
$rg = terraform output -raw resource_group_name
az vm get-instance-view -g $rg -n $(terraform output -raw vm_name) --query "instanceView.statuses" -o table
```

---

## Cleanup

```powershell
terraform destroy
```

Type `yes`. Remove `terraform.tfvars` from disk if it contains your password, or rotate the password if you reused one elsewhere.

```powershell
Remove-Item terraform.tfvars -ErrorAction SilentlyContinue
```

## Lab completion checklist

- [ ] RDP works from allowed IP only  
- [ ] VM private IP in `10.0.2.0/24`  
- [ ] Destroy completed; no orphaned disks in Portal  

## Azure track progress (Labs 01–04)

You have provisioned resource groups, VNet/NSG, Linux VM, and Windows VM. **Lab 05** adds containers on Azure before the AWS track.

## Next lab

[Lab 05 — Deploy a Container with Azure Container Instances](./Lab%2005%20-%20Deploy%20a%20Container%20with%20Azure%20Container%20Instances.md).
