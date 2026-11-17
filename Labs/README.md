# Cloud Infrastructure with Terraform — Hands-On Labs

Step-by-step exercises for **Azure** (Labs 01–05) and **AWS** (Labs 06–08) with [Terraform](https://www.terraform.io/).

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

Complete **Labs 01–05 (Azure)** before starting **AWS Labs 06–08** (or have equivalent Terraform experience).

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

## Conventions

- Use a **unique name** per lab run (resource group, `name_prefix`, etc.).
- **Cost**: VMs, EC2, containers, and Elastic IPs can incur charges — run **`terraform destroy`** after each lab.
- **Lab root folder**: `labs/work/lab-NN/` relative to repository root (gitignored via `work/`).
- **State**: Local `terraform.tfstate` and `.terraform/` live in your lab root folder; see [`.gitignore`](./.gitignore).

## Getting help

- [Terraform Azure tutorials](https://developer.hashicorp.com/terraform/tutorials/azure-get-started) · [AzureRM provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform AWS tutorials](https://developer.hashicorp.com/terraform/tutorials/aws-get-started) · [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)