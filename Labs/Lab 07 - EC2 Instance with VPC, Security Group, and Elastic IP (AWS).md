# Lab 07 — EC2 Instance with VPC, Security Group, and Elastic IP (AWS)

## Description

You deploy a **complete AWS stack** end to end: VPC networking (subnets, internet gateway, route tables) plus an **Amazon Linux 2023 EC2** instance in a public subnet, attached to the **web security group**, with an **Elastic IP** associated for stable SSH access.

This lab reinforces Lab 06 networking and focuses on **compute**: AMI selection, key pairs, user data (optional), and `aws_eip_association`.

## Prerequisites

- Completed [Lab 06](./Lab%2006%20-%20VPC,%20Subnets,%20Internet%20Gateway,%20Route%20Tables,%20Security%20Groups,%20Elastic%20IP%20(AWS).md) or equivalent understanding of VPC + security groups
- AWS credentials configured; quota for `t3.micro` (Free Tier eligible in many accounts)
- **SSH key pair** on your workstation (Step 2)

## Use case / scenario

Contoso’s DevOps team needs a disposable **build agent** host in `us-east-1` for Terraform pipeline experiments. Requirements:

- Host must sit in a **public subnet** with controlled **SSH** access  
- Public address must **not change** when the instance stops/starts → **Elastic IP**  
- Host must use the shared **web** security group pattern from Lab 06  

## Topic explanation

### EC2 and ENI

- Each instance has a primary **elastic network interface** in one subnet.  
- `vpc_security_group_ids` attaches security groups at the ENI.  
- `associate_public_ip_address` can be set on subnet; this lab also uses EIP for a fixed address.

### Key pairs

- AWS stores the **public** key; you keep the **private** key.  
- Terraform resource: `aws_key_pair`.

### Elastic IP association

- `aws_eip` + `aws_eip_association` (or `instance` argument on newer patterns) binds a public IP to the instance.  
- Releasing the association (or destroy) returns the EIP to your pool.

### Relationship to Lab 05

| Lab 06 concept | Lab 06 usage |
|----------------|--------------|
| `for_each` subnets | Instance placed in `public-1a` |
| `dynamic` SG rules | Same pattern; instance uses `aws_security_group.web` |
| `count` on EIP | Lab 06 uses a **single** EIP bound to EC2 (simpler than Lab 06 standby EIPs) |

---

## Terraform configuration

Create each file in `labs/work/lab-07/` with the contents below.

### `providers.tf`

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
      lab         = "07-ec2"
    }
  }
}
```

### `variables.tf`

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "name_prefix" {
  type    = string
  default = "lab-06-demo"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "allowed_ssh_cidr" {
  type    = string
  default = "0.0.0.0/0"
}

variable "public_key" {
  type        = string
  description = "SSH public key for EC2 access"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "instance_subnet_key" {
  type        = string
  description = "Key from local.subnet_definitions (for_each)"
  default     = "public-1a"
}
```

### `locals.tf`

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

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

  ingress_rules = [
    {
      description = "SSH from admin"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [var.allowed_ssh_cidr]
    },
    {
      description = "HTTP for test"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
  ]

  egress_rules = [
    {
      description = "All outbound"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    },
  ]
}
```

### `network.tf`

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
  for_each = local.private_subnet_keys

  subnet_id      = aws_subnet.lab[each.key].id
  route_table_id = aws_route_table.private.id
}
```

### `security_groups.tf`

```hcl
resource "aws_security_group" "web" {
  name        = "${var.name_prefix}-sg-web"
  description = "Web/SSH access for EC2 lab"
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

### `ec2.tf`

```hcl
resource "aws_key_pair" "lab" {
  key_name   = "${var.name_prefix}-key"
  public_key = var.public_key
}

resource "aws_instance" "lab" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.lab[var.instance_subnet_key].id
  vpc_security_group_ids      = [aws_security_group.web.id]
  key_name                    = aws_key_pair.lab.key_name
  associate_public_ip_address = true

  user_data = <<-EOF
              #!/bin/bash
              dnf update -y
              EOF

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  tags = {
    Name = "${var.name_prefix}-ec2"
  }

  lifecycle {
    ignore_changes = [user_data]
  }
}
```

### `elastic_ip.tf`

```hcl
resource "aws_eip" "lab" {
  domain = "vpc"

  tags = {
    Name = "${var.name_prefix}-eip"
  }

  depends_on = [aws_internet_gateway.lab]
}

resource "aws_eip_association" "lab" {
  allocation_id = aws_eip.lab.id
  instance_id   = aws_instance.lab.id
}
```

### `outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.lab.id
}

output "instance_id" {
  value = aws_instance.lab.id
}

output "private_ip" {
  value = aws_instance.lab.private_ip
}

output "elastic_ip" {
  value = aws_eip.lab.public_ip
}

output "security_group_id" {
  value = aws_security_group.web.id
}

output "ssh_command" {
  value = "ssh -i <private-key-file> ec2-user@${aws_eip.lab.public_ip}"
}
```

### `terraform.tfvars` (example)

Create `terraform.tfvars` (do not commit secrets). Example:

```hcl
name_prefix      = "lab-06-yourinitials"
allowed_ssh_cidr = "203.0.113.10/32"
public_key       = "ssh-ed25519 AAAA... aws-lab06"
```

Copy from `terraform.tfvars.example` if you prefer:

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

- `name_prefix` — unique  
- `public_key` — contents of `id_ed25519_aws_lab06.pub` (one line)  
- `allowed_ssh_cidr` — your public IP `/32`  

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-07/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Create Terraform files

Create the eight `.tf` files listed in [Terraform configuration](#terraform-configuration) in this directory.

### Step 2 — Create SSH key

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\id_ed25519_aws_lab06 -C "aws-lab06"
```

### Step 3 — Configure variables

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-07" -Force | Out-Null
Set-Location "labs\work\lab-07"
```
Copy-Item terraform.tfvars.example terraform.tfvars
```

See [terraform.tfvars (example)](#terraformtfvars-example) for required values.

### Step 4 — Review configuration

| File | Purpose |
|------|---------|
| `providers.tf` | Terraform and AWS provider constraints, default tags |
| `variables.tf` | Region, CIDR, SSH CIDR, public key, instance options |
| `locals.tf` | AMI data source, subnet map, SG rule lists |
| `network.tf` | VPC, IGW, subnets, routes (`for_each`) |
| `security_groups.tf` | SG with `dynamic` rules |
| `ec2.tf` | Key pair, instance, user data snippet |
| `elastic_ip.tf` | EIP + association to instance |
| `outputs.tf` | VPC, instance, EIP, SSH helper |

Instance defaults: `t3.micro`, Amazon Linux 2023.

### Step 5 — Deploy

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Wait until the instance passes **status checks** (a few minutes).

### Step 6 — SSH to the instance

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519_aws_lab06 ec2-user@$(terraform output -raw elastic_ip)
```

Run:

```bash
curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone
sudo dnf install -y httpd && sudo systemctl enable --now httpd
```

### Step 7 — Verify

```powershell
aws ec2 describe-instances --instance-ids $(terraform output -raw instance_id) `
  --query "Reservations[0].Instances[0].{State:State.Name,PublicIp:PublicIpAddress,Subnet:SubnetId,Sg:SecurityGroups[0].GroupId}" `
  --output table
```

From a browser, try `http://<elastic-ip>/` if you installed httpd.

### Step 8 — Optional exercises

1. Change `instance_subnet_key` to `public-1b` and apply—instances may replace (AZ change).  
2. Tighten SG by removing HTTP/HTTPS rules and confirm only SSH works.  
3. Add `aws_volume_attachment` for an extra EBS volume.

---

## Cleanup

```powershell
terraform destroy
```

Confirm `yes`. Verify no instances or EIPs remain:

```powershell
aws ec2 describe-addresses --query "Addresses[?Domain=='vpc'].PublicIp" --output table
```

## Lab completion checklist

- [ ] EC2 in public subnet with correct SG  
- [ ] SSH works from `allowed_ssh_cidr`  
- [ ] Elastic IP output matches SSH target  
- [ ] Destroy completed  

## Next lab

[Lab 08 — IAM roles and instance profile for EC2](./Lab%2008%20-%20IAM%20Roles,%20Instance%20Profile,%20and%20EC2%20(AWS).md).md).