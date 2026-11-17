# Lab 06 — VPC, Subnets, Internet Gateway, Route Tables, Security Groups, Elastic IP (AWS)

## Description

You build a **complete base network** in AWS using Terraform: VPC, multiple subnets across availability zones, internet gateway, public (and optional private) route tables, **security groups** defined with **`dynamic`** ingress/egress blocks, and **Elastic IPs** provisioned with **`count`**. Subnets and route table associations use **`for_each`**.

## Prerequisites

- Completed [Lab 05](./Lab%2005%20-%20Deploy%20a%20Container%20with%20Azure%20Container%20Instances.md) (Azure) and [Labs README](./README.md) AWS prerequisites
- AWS Labs **06–08**: credentials and CLI
- `aws sts get-caller-identity` succeeds
- Basic CIDR knowledge (`10.0.0.0/16`, `/24` subnets)

## Use case / scenario

Northwind Traders is standardizing a **hub pattern** for small workloads in `us-east-1`. Network engineering requires:

- One VPC with room to grow  
- **Two public subnets** (different AZs) for future load-balanced front ends  
- One **private** subnet slice for app tier (no NAT gateway in this lab—to limit cost)  
- Internet egress for public subnets via an **internet gateway**  
- A reusable **web** security group (HTTP/HTTPS from the internet, SSH from admin CIDR only)  
- Two **unassociated Elastic IPs** reserved for bastion/NLB demos (`count`)

## Topic explanation

### VPC and subnets

- **VPC**: Isolated network; you define CIDR (`10.0.0.0/16`).  
- **Subnet**: AZ-scoped segment; public subnets have a route to an **internet gateway**.  
- **`for_each`**: Creates one `aws_subnet` per map key—ideal for named tiers (`public-1a`, `private-1a`).

### Internet gateway and route tables

- **IGW**: Allows bidirectional internet access for resources with public IPs.  
- **Route table**: List of routes; `0.0.0.0/0` → IGW on the **public** table.  
- **`aws_route_table_association`**: Links each public subnet to the public route table (`for_each`).

### Security groups vs Azure NSG

- **Security group**: Stateful firewall attached to ENIs (instances, LB, etc.).  
- **`dynamic` blocks**: Loop over a list of rule objects to avoid copy-paste `ingress` stanzas.

### Elastic IP

- Static public IPv4 in your account.  
- **`count`**: Provisions `var.eip_count` standalone EIPs (not yet attached—that is Lab 07).

### `count` vs `for_each` in this lab

| Meta-argument | Used for |
|---------------|----------|
| `for_each` | Subnets, public route associations, optional tagging maps |
| `count` | Optional private route table (`count = var.enable_private_route_table ? 1 : 0`), Elastic IPs |
| `dynamic` | Security group `ingress` / `egress` from `local.ingress_rules` / `local.egress_rules` (defaults from `locals.tf`, overridable via `var.ingress_rules` / `var.egress_rules`) |

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-06/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Verify AWS access

```powershell
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
```

### Step 2 — Create Terraform files

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-06" -Force | Out-Null
Set-Location "labs\work\lab-06"
```


Create these files in your working directory (see note above).

**Address design:**

| Subnet key | CIDR | Tier |
|------------|------|------|
| `public-1a` | `10.0.1.0/24` | Public |
| `public-1b` | `10.0.2.0/24` | Public |
| `private-1a` | `10.0.11.0/24` | Private (local routes only) |

#### Create `providers.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.70"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      environment = "lab"
      managed_by  = "terraform"
      lab         = "06-vpc"
    }
  }
}
```

#### Create `variables.tf`

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "name_prefix" {
  type        = string
  description = "Prefix for resource names — make unique per learner"
  default     = "lab-05-demo"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "allowed_ssh_cidr" {
  type        = string
  description = "Source CIDR for SSH to future EC2 (used in SG rule)"
  default     = "0.0.0.0/0"
}

variable "enable_private_route_table" {
  type        = bool
  description = "When true, creates one private route table (count demo)"
  default     = true
}

variable "eip_count" {
  type        = number
  description = "Number of standalone Elastic IPs to allocate (count demo)"
  default     = 2
}

variable "ingress_rules" {
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}

variable "egress_rules" {
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = []
}
```

#### Create `locals.tf`

```hcl
locals {
  subnet_definitions = {
    public-1a = {
      cidr              = "10.0.1.0/24"
      availability_zone = "${var.aws_region}a"
      tier              = "public"
    }
    public-1b = {
      cidr              = "10.0.2.0/24"
      availability_zone = "${var.aws_region}b"
      tier              = "public"
    }
    private-1a = {
      cidr              = "10.0.11.0/24"
      availability_zone = "${var.aws_region}a"
      tier              = "private"
    }
  }

  public_subnet_keys = {
    for name, cfg in local.subnet_definitions : name => cfg if cfg.tier == "public"
  }

  private_subnet_keys = {
    for name, cfg in local.subnet_definitions : name => cfg if cfg.tier == "private"
  }

  default_ingress_rules = [
    {
      description = "HTTPS from internet"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTP from internet"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "SSH from admin"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [var.allowed_ssh_cidr]
    },
  ]

  default_egress_rules = [
    {
      description = "All outbound"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    },
  ]

  ingress_rules = length(var.ingress_rules) > 0 ? var.ingress_rules : local.default_ingress_rules
  egress_rules  = length(var.egress_rules) > 0 ? var.egress_rules : local.default_egress_rules
}
```

#### Create `network.tf`

```hcl
resource "aws_vpc" "lab" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.name_prefix}-vpc"
  }
}

resource "aws_internet_gateway" "lab" {
  vpc_id = aws_vpc.lab.id

  tags = {
    Name = "${var.name_prefix}-igw"
  }
}

resource "aws_subnet" "lab" {
  for_each = local.subnet_definitions

  vpc_id                  = aws_vpc.lab.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = each.value.tier == "public"

  tags = {
    Name = "${var.name_prefix}-${each.key}"
    Tier = each.value.tier
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.lab.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.lab.id
  }

  tags = {
    Name = "${var.name_prefix}-rt-public"
  }
}

resource "aws_route_table" "private" {
  count = var.enable_private_route_table ? 1 : 0

  vpc_id = aws_vpc.lab.id

  tags = {
    Name = "${var.name_prefix}-rt-private"
  }
}

resource "aws_route_table_association" "public" {
  for_each = local.public_subnet_keys

  subnet_id      = aws_subnet.lab[each.key].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  for_each = var.enable_private_route_table ? local.private_subnet_keys : {}

  subnet_id      = aws_subnet.lab[each.key].id
  route_table_id = aws_route_table.private[0].id
}
```

#### Create `security_groups.tf`

```hcl
resource "aws_security_group" "web" {
  name        = "${var.name_prefix}-sg-web"
  description = "Web tier security group (Lab 06 dynamic rules)"
  vpc_id      = aws_vpc.lab.id

  dynamic "ingress" {
    for_each = local.ingress_rules
    content {
      description = ingress.value.description
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  dynamic "egress" {
    for_each = local.egress_rules
    content {
      description = egress.value.description
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }

  tags = {
    Name = "${var.name_prefix}-sg-web"
  }
}
```

#### Create `elastic_ip.tf`

```hcl
resource "aws_eip" "standby" {
  count = var.eip_count

  domain = "vpc"

  tags = {
    Name = "${var.name_prefix}-eip-${count.index}"
  }

  depends_on = [aws_internet_gateway.lab]
}
```

#### Create `outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.lab.id
}

output "public_subnet_ids" {
  value = { for k, s in aws_subnet.lab : k => s.id if local.subnet_definitions[k].tier == "public" }
}

output "private_subnet_ids" {
  value = { for k, s in aws_subnet.lab : k => s.id if local.subnet_definitions[k].tier == "private" }
}

output "internet_gateway_id" {
  value = aws_internet_gateway.lab.id
}

output "web_security_group_id" {
  value = aws_security_group.web.id
}

output "elastic_ip_addresses" {
  value = aws_eip.standby[*].public_ip
}

output "elastic_ip_allocation_ids" {
  value = aws_eip.standby[*].allocation_id
}
```

### Step 3 — Create `terraform.tfvars`

Create `terraform.tfvars` in the same folder. Use these **example** values, then replace with your data:

```hcl
name_prefix                = "lab-05-yourinitials"
allowed_ssh_cidr           = "203.0.113.10/32"
enable_private_route_table = true
eip_count                  = 2
```

| Setting | What to change |
|---------|----------------|
| `name_prefix` | Unique prefix (e.g. include your initials) |
| `allowed_ssh_cidr` | Your public IP with `/32` for SSH in the web security group |
| `enable_private_route_table` | Set `false` to demo `count = 0` on the private route table |
| `eip_count` | Number of standalone EIPs (`count`) |

Alternatively, set values without a file:

```powershell
$env:TF_VAR_name_prefix = "lab05-jd"
$env:TF_VAR_allowed_ssh_cidr = "203.0.113.10/32"
```

**Do not commit `terraform.tfvars`.** A `.gitignore` in `labs/` excludes it if present.

### Step 4 — Initialize and deploy

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

Expect **multiple resources**: 1 VPC, 3 subnets, 1 IGW, 1–2 route tables, associations, 1 SG, 2 EIPs (default).

### Step 5 — Verify with AWS CLI

```powershell
$vpc = terraform output -raw vpc_id
aws ec2 describe-vpcs --vpc-ids $vpc --query "Vpcs[0].CidrBlock" --output text
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpc" --query "Subnets[].{Id:SubnetId,Cidr:CidrBlock,Az:AvailabilityZone}" --output table
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$vpc" --output table
terraform output elastic_ip_addresses
```

### Step 6 — Portal check

VPC console → your VPC → **Resource map** → confirm IGW attached and public subnets route to it.

### Step 7 — Optional exercises

1. Add an ingress rule object to `var.ingress_rules` for port `8080` and run `plan` again.  
2. Set `enable_private_route_table = false` and observe **`count = 0`** removes the private route table.  
3. Change `eip_count` to `1` and note **count index** changes (careful: may force EIP recreation).

---

## Cleanup

```powershell
terraform destroy
```

Confirm `yes`. Unattached EIPs are released on destroy.

## Lab completion checklist

- [ ] Three subnets in two AZs  
- [ ] Public subnets use route to IGW  
- [ ] Security group rules driven by variables + `dynamic`  
- [ ] Elastic IP outputs shown  
- [ ] All resources destroyed  

## Next lab

[Lab 07 — EC2 with VPC and Elastic IP](./Lab%2007%20-%20EC2%20Instance%20with%20VPC,%20Security%20Group,%20and%20Elastic%20IP%20(AWS).md).md).