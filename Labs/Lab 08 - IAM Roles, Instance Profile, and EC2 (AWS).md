# Lab 08 — IAM Roles, Instance Profile, and EC2 (AWS)

## Description

You implement **least-privilege access for EC2** using **IAM roles** instead of long-lived keys on the instance. Terraform creates an **IAM role** with a trust policy for EC2, attaches the **AmazonSSMManagedInstanceCore** managed policy, publishes an **instance profile**, and launches an EC2 instance in your VPC with that profile—ready for **AWS Systems Manager Session Manager**.

The lab includes the same VPC/subnet/IGW/route/SG foundation as Labs 06–07 so you have a **complete AWS infrastructure** story.

## Prerequisites

- Completed [Lab 07](./Lab%2007%20-%20EC2%20Instance%20with%20VPC,%20Security%20Group,%20and%20Elastic%20IP%20(AWS).md) or comfortable with EC2 + VPC
- [Labs README](./README.md) — AWS credentials and CLI (Labs 06–08)
- `aws sts get-caller-identity` succeeds with credentials that can create IAM roles and instance profiles (`iam:CreateRole`, etc.)
- Optional: SSH key for fallback access (`public_key` in `terraform.tfvars`)

## Use case / scenario

Adventure Works security policy forbids storing **AWS access keys** on build VMs. Operations wants:

- EC2 instances to call AWS APIs with **temporary credentials** from instance metadata  
- Shell access via **Session Manager** where possible (no inbound SSH rule required)  
- A dedicated role **per workload** (`for_each` over a small map of role definitions)  

This lab uses one primary workload role and demonstrates **`for_each`** on IAM policies attachments where multiple roles are defined.

## Topic explanation

### IAM role trust policy

- **Trust policy**: Who can assume the role (`ec2.amazonaws.com` for EC2).  
- **Permission policies**: What the role can do (SSM core for Session Manager).

### Instance profile

- Container AWS exposes to EC2 as `iam_instance_profile`.  
- One profile maps to one role in simple cases (`aws_iam_instance_profile` + `role`).

### Instance metadata (IMDSv2)

- Lab enables `http_tokens = "required"` (IMDSv2) on the instance.  
- Apps and agents use metadata for **temporary** `AccessKeyId`, `SecretAccessKey`, `Token`.

### `for_each` on IAM attachments

- `var.instance_roles` map defines multiple logical roles; each gets `aws_iam_role`, attachment, and profile—pattern for web vs worker tiers.

---

## Step-by-step instructions
> **Lab root folder:** From your **repository root** (the folder that contains `labs`), create and use **`labs/work/lab-08/`** for all Terraform files. Run every `terraform` command from that folder.


### Step 1 — Verify AWS access

Confirm the AWS CLI uses the account and identity you intend **before** you create files or run `terraform apply`:

```powershell
aws sts get-caller-identity
$env:AWS_REGION = "us-east-1"
```

Note the **Account** and **Arn** (typically an IAM user or assumed role). Deploy will fail if credentials are missing or pointed at the wrong account.

### Step 2 — Create Terraform files

```powershell
New-Item -ItemType Directory -Path "labs\work\lab-08" -Force | Out-Null
Set-Location "labs\work\lab-08"
```


Create these files in your working directory (see note above).

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
      lab         = "08-iam"
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
  type    = string
  default = "lab-07-demo"
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
  description = "Optional SSH public key; leave empty to disable key pair resource"
  default     = ""
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "enable_ssh_ingress" {
  type    = bool
  default = true
}

variable "instance_roles" {
  type = map(object({
    description = string
  }))
  default = {
    app = {
      description = "Application EC2 role for SSM and lab demos"
    }
  }
}
```

#### Create `locals.tf`

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

  ssh_ingress_rule = {
    description = "SSH from admin"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.allowed_ssh_cidr]
  }

  ingress_rules = var.enable_ssh_ingress ? [local.ssh_ingress_rule] : []

  egress_rules = [
    {
      description = "All outbound"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    },
  ]

  instance_roles = var.instance_roles
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

resource "aws_route_table_association" "public" {
  for_each = local.public_subnet_keys

  subnet_id      = aws_subnet.lab[each.key].id
  route_table_id = aws_route_table.public.id
}
```

#### Create `security_groups.tf`

```hcl
resource "aws_security_group" "app" {
  name        = "${var.name_prefix}-sg-app"
  description = "App instance security group (SSM-first; optional SSH)"
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
    Name = "${var.name_prefix}-sg-app"
  }
}
```

#### Create `iam.tf`

```hcl
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "ec2" {
  for_each = local.instance_roles

  name               = "${var.name_prefix}-role-${each.key}"
  description        = each.value.description
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
}

resource "aws_iam_role_policy_attachment" "ssm_core" {
  for_each = aws_iam_role.ec2

  role       = each.value.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "ec2" {
  for_each = aws_iam_role.ec2

  name = "${var.name_prefix}-profile-${each.key}"
  role = each.value.name
}
```

#### Create `ec2.tf`

```hcl
resource "aws_key_pair" "lab" {
  count = var.public_key != "" ? 1 : 0

  key_name   = "${var.name_prefix}-key"
  public_key = var.public_key
}

resource "aws_instance" "app" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.lab["public-1a"].id
  vpc_security_group_ids = [aws_security_group.app.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2["app"].name

  key_name = var.public_key != "" ? aws_key_pair.lab[0].key_name : null

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  tags = {
    Name = "${var.name_prefix}-ec2-app"
  }
}
```

#### Create `elastic_ip.tf`

```hcl
resource "aws_eip" "app" {
  domain = "vpc"

  tags = {
    Name = "${var.name_prefix}-eip"
  }

  depends_on = [aws_internet_gateway.lab]
}

resource "aws_eip_association" "app" {
  allocation_id = aws_eip.app.id
  instance_id   = aws_instance.app.id
}
```

#### Create `outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.lab.id
}

output "instance_id" {
  value = aws_instance.app.id
}

output "iam_role_arn" {
  value = aws_iam_role.ec2["app"].arn
}

output "instance_profile_arn" {
  value = aws_iam_instance_profile.ec2["app"].arn
}

output "elastic_ip" {
  value = aws_eip.app.public_ip
}

output "ssm_start_session_command" {
  value = "aws ssm start-session --target ${aws_instance.app.id} --region ${var.aws_region}"
}
```

**File map:**

| File | Purpose |
|------|---------|
| `network.tf` | VPC, IGW, subnets, routes (`for_each`) |
| `security_groups.tf` | SG with optional SSH via `dynamic` ingress |
| `iam.tf` | Roles, SSM policy attachment, instance profiles (`for_each`) |
| `ec2.tf` | Instance with `iam_instance_profile` and IMDSv2 |
| `elastic_ip.tf` | EIP + association |

### Step 3 — Review `iam.tf`

After you create the files above, confirm:

- `aws_iam_role` per entry in `local.instance_roles` (`for_each`)  
- `aws_iam_role_policy_attachment` → `AmazonSSMManagedInstanceCore`  
- `aws_iam_instance_profile` linked to each role  

`ec2.tf` sets `iam_instance_profile = aws_iam_instance_profile.ec2["app"].name`.

### Step 4 — Create `terraform.tfvars`

Create `terraform.tfvars` in the same folder (or copy from the example below). **Do not commit `terraform.tfvars`.** A `.gitignore` in `labs/` excludes it if present.

**Example `terraform.tfvars`:**

```hcl
name_prefix        = "lab-07-yourinitials"
allowed_ssh_cidr   = "203.0.113.10/32"
enable_ssh_ingress = false
# public_key       = ""
```

| Setting | What to change |
|---------|----------------|
| `name_prefix` | Unique prefix (e.g. include your initials) |
| `allowed_ssh_cidr` | Your public IP with `/32` if you enable SSH |
| `enable_ssh_ingress` | Set `false` to rely on SSM only (recommended) |
| `public_key` | Optional one-line `.pub` contents if you want SSH fallback |

### Step 5 — Deploy

```powershell
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

### Step 6 — Session Manager (preferred)

1. Ensure the SSM agent is present (pre-installed on Amazon Linux 2023).  
2. Wait 2–5 minutes after instance launch.  
3. AWS Console → **Systems Manager** → **Session Manager** → **Start session** → select your instance.

Or CLI:

```powershell
aws ssm start-session --target $(terraform output -raw instance_id)
```

### Step 7 — Prove role credentials (on instance via SSM)

```bash
aws sts get-caller-identity
```

Should show the **assumed role ARN**, not an IAM user.

### Step 8 — Optional SSH

If you provided `public_key`, you may SSH using the EIP output. Compare: SSH uses **network** access; SSM uses **IAM**.

### Step 9 — Optional exercises

1. Add a second map entry in `var.instance_roles` and run `plan`—observe **for_each** growth.  
2. Attach a custom inline policy granting `s3:ListBucket` on one bucket—discuss scope.  
3. Remove SSH ingress rule from `locals.ingress_rules` and rely on SSM only.

---

## Cleanup

```powershell
terraform destroy
```

IAM roles and profiles are removed when destroy completes (no dependencies from other accounts).

**Note:** If destroy fails on IAM, ensure no other EC2 instances still reference the instance profile.

## Lab completion checklist

- [ ] `aws sts get-caller-identity` succeeded locally before deploy  
- [ ] Instance profile attached to EC2  
- [ ] `aws sts get-caller-identity` on instance shows role  
- [ ] Session Manager session works  
- [ ] Full stack destroyed  

## Course wrap-up (Labs 06–08)

You built on AWS:

1. **Lab 06** — VPC, subnets, IGW, routes, SG (`dynamic`, `for_each`, `count`), EIPs  
2. **Lab 07** — EC2 with SG and Elastic IP  
3. **Lab 08** — IAM roles and instance profiles for secure AWS API access  

## Next lab

[Lab 09 — Terraform Modules](./Lab%2009%20-%20Terraform%20Modules%20Local%20Registry%20Versioning%20and%20Design.md): local modules, registry modules, versioning, private modules, and design standards.