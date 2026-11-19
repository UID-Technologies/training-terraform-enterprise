# Lab 03 — Linux Virtual Machine with VNet Integration

## Description

You deploy an **Ubuntu Linux** virtual machine attached to the **web subnet** from Lab 02’s design pattern: VNet, subnet, NSG, public IP on the NIC, network interface, and Linux VM with SSH key authentication—all defined in Terraform.

## Prerequisites

- Completed [Lab 02](./Lab 02 - Virtual Network, Subnet, NSG, and Public IP.md) or understand VNet/subnet/NSG concepts
- Azure CLI logged in; subscription with quota for at least one `Standard_B1s` (or similar) VM
- An **SSH key pair** on your workstation (see Step 1)

## Use case / scenario

Adventure Works needs a jump host / dev sandbox for container tooling on Ubuntu. Requirements:

- VM lives in `snet-web` with private IP from that subnet  
- Admin access via **SSH (22)** only from your IP (not open to `0.0.0.0/0`)  
- Public IP on the NIC for outbound and SSH (simpler than Bastion for this lab)  
- No password auth; **SSH public key** only  

## Topic explanation

### Network interface (NIC)

- Connects a VM to a subnet  
- Can have dynamic or static private IP  
- Can reference a public IP for inbound connectivity  

### Linux VM resources

| Resource | Role |
|----------|------|
| `azurerm_linux_virtual_machine` | OS disk, size, admin user, SSH keys |
| `azurerm_network_interface` | Subnet membership |
| `azurerm_public_ip` | Optional inbound path |

### NSG for this lab

Lab 02 denied SSH at subnet level for the generic design. **This lab’s configuration** uses a dedicated NSG that **allows SSH from your public IP** so you can connect. In production, prefer Azure Bastion or private-only NICs with VPN/ExpressRoute.

### Sizing and cost

Default size `Standard_B1s` is low cost; still **run destroy** when finished.

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-03/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Generate SSH key (if needed)

**PowerShell (Windows):**

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\id_ed25519_azure_lab -C "azure-lab"
```

or

```bash
ssh-keygen -t rsa -b 4096 -f "$env:USERPROFILE\.ssh\azure_terraform_rsa"
```

Note the path to the **`.pub`** file.

### Step 2 — Create Terraform files

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-03" -Force | Out-Null
Set-Location "labs\work\lab-03"
```


Create these four files in your working directory (see note above). Provider pin matches [Lab 01](./Lab 01 - Create an Azure Resource Group with Terraform.md).

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
  default = "rg-terraform-lab-03-demo"
}

variable "vnet_name" {
  type    = string
  default = "vnet-lab-03"
}

variable "subnet_name" {
  type    = string
  default = "snet-web"
}

variable "vm_name" {
  type    = string
  default = "vm-linux-lab03"
}

variable "vm_size" {
  type    = string
  default = "Standard_B1s"
}

variable "admin_username" {
  type    = string
  default = "azureuser"
}

variable "ssh_public_key" {
  type        = string
  description = "SSH public key content (single line)"
  default     = ""
}

variable "allowed_ssh_cidr" {
  type        = string
  description = "CIDR allowed to SSH to the VM (use your public IP/32)"
  default     = "0.0.0.0/0"
}
```

#### Create `main.tf`

```hcl
locals {
  tags = {
    environment = "lab"
    lab         = "03-linux-vm"
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

resource "azurerm_subnet" "web" {
  name                 = var.subnet_name
  resource_group_name  = azurerm_resource_group.lab.name
  virtual_network_name = azurerm_virtual_network.lab.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_security_group" "vm" {
  name                = "nsg-${var.vm_name}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  tags                = local.tags
}

resource "azurerm_network_security_rule" "ssh" {
  name                        = "AllowSSHFromAdmin"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = var.allowed_ssh_cidr
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.lab.name
  network_security_group_name = azurerm_network_security_group.vm.name
}

resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
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
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.vm.id
  }

  tags = local.tags
}

resource "azurerm_linux_virtual_machine" "lab" {
  name                = var.vm_name
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  size                = var.vm_size
  admin_username      = var.admin_username

  network_interface_ids = [
    azurerm_network_interface.vm.id,
  ]

  disable_password_authentication = true

  admin_ssh_key {
    username   = var.admin_username
    public_key = var.ssh_public_key
  }

  os_disk {
    name                 = "${var.vm_name}-osdisk"
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  tags = local.tags
}
```

#### Create `outputs.tf`

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.lab.name
}

output "vm_name" {
  value = azurerm_linux_virtual_machine.lab.name
}

output "nic_name" {
  value = azurerm_network_interface.vm.name
}

output "private_ip_address" {
  value = azurerm_network_interface.vm.private_ip_address
}

output "public_ip_address" {
  value = azurerm_public_ip.vm.ip_address
}

output "ssh_command" {
  value = "ssh ${var.admin_username}@${azurerm_public_ip.vm.ip_address}"
}
```

**Dependency chain:**

`resource group` → `vnet` → `subnet` → `nsg` (+ SSH rule) → `public ip` → `nic` → `linux vm`

The SSH allow rule uses `var.allowed_ssh_cidr`.

### Step 3 — Create `terraform.tfvars`

Create `terraform.tfvars` in the same folder. Use these **example** values, then replace with your real data:

```hcl
resource_group_name = "rg-terraform-lab-03-yourinitials"
ssh_public_key      = "ssh-ed25519 AAAA... your-comment"
allowed_ssh_cidr    = "203.0.113.10/32"
```

| Setting | What to change |
|---------|----------------|
| `resource_group_name` | Unique name (e.g. include your initials) |
| `ssh_public_key` | Paste the full single-line contents of your `.pub` file from Step 1 |
| `allowed_ssh_cidr` | Your public IP with `/32` (find it: `curl -s ifconfig.me` or search “what is my IP”) |

**Do not commit `terraform.tfvars`.** A `.gitignore` in `labs/` excludes it if present.

### Step 4 — Deploy

```powershell
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Apply may take **5–10 minutes** (VM provisioning).

### Step 5 — Connect via SSH

```powershell
$ip = terraform output -raw public_ip_address
ssh -i $env:USERPROFILE\.ssh\id_ed25519_azure_lab azureuser@$ip
```

Run `uname -a` and `ip addr` on the VM to confirm OS and private IP (`10.0.1.x`).

### Step 6 — Verify networking

```powershell
$rg = terraform output -raw resource_group_name
az vm show -d -g $rg -n $(terraform output -raw vm_name) --query "publicIps" -o tsv
az network nic show -g $rg -n $(terraform output -raw nic_name) --query "ipConfigurations[0].privateIPAddress" -o tsv
```

### Step 7 — Optional exercises

1. Change VM `size` and run `plan` — note **resize may require deallocation** (provider may replace or update in place depending on SKU change).  
2. Add a `custom_data` blob to install nginx (see Terraform docs for `custom_data` base64).  
3. Remove public IP and discuss how you would access the VM privately.

---

## Cleanup

```powershell
terraform destroy
```

Confirm `yes`. Verify the resource group is gone:

```powershell
az group exists --name <your-rg-name>
```

Delete local SSH key only if you created it solely for this lab.

## Lab completion checklist

- [ ] SSH works from your IP only (test fails from mobile hotspot if IP differs — expected)
- [ ] Private IP is in `10.0.1.0/24`
- [ ] `terraform destroy` removed VM and disks

## Next lab

[Lab 04 — Windows virtual machine with VNet](./Lab 04 - Windows Virtual Machine with VNet Integration.md).
