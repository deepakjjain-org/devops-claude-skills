# Reusable Terraform VPC Module for Multi-Environment Deployment

## Skill Workflow: Decision Tree Result

Following the **iac-terraform** skill's core workflow decision tree:

> **Is this reusable across environments/projects?** --> **Yes** --> **Create a Terraform module**

Since you need the same VPC setup across 3 environments (dev/staging/prod) with different CIDR ranges, this is a textbook case for a reusable Terraform module. Below is the complete guidance.

---

## Step 1: Scaffold the Module

Use the skill's `init_module.py` script to create the standard structure:

```bash
python3 scripts/init_module.py vpc --path ./modules
```

This creates the required file structure:
```
modules/vpc/
├── main.tf           # Primary resource definitions
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── versions.tf       # Version constraints
├── README.md         # Module documentation
└── examples/
    └── complete/
        ├── main.tf
        ├── versions.tf
        └── README.md
```

---

## Step 2: Module Implementation

### versions.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### variables.tf

Following the skill's best practices for variable declarations -- every variable has a `type`, `description`, and `validation` where appropriate. Sensitive naming conventions are respected. Variables are grouped by concern (network configuration, feature toggles, tagging).

```hcl
# =============================================================================
# Network Configuration
# =============================================================================

variable "name" {
  type        = string
  description = "Name prefix for all VPC resources (e.g., 'myapp-dev')"
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC (e.g., '10.0.0.0/16')"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid IPv4 CIDR block."
  }
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "az_count" {
  type        = number
  description = "Number of availability zones to use (2 for dev, 3 for prod recommended)"
  default     = 2

  validation {
    condition     = var.az_count >= 2 && var.az_count <= 4
    error_message = "AZ count must be between 2 and 4."
  }
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for public subnets (one per AZ)"

  validation {
    condition     = alltrue([for cidr in var.public_subnet_cidrs : can(cidrhost(cidr, 0))])
    error_message = "All public subnet CIDRs must be valid IPv4 CIDR blocks."
  }
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for private subnets (one per AZ)"

  validation {
    condition     = alltrue([for cidr in var.private_subnet_cidrs : can(cidrhost(cidr, 0))])
    error_message = "All private subnet CIDRs must be valid IPv4 CIDR blocks."
  }
}

# =============================================================================
# Feature Toggles
# =============================================================================

variable "enable_dns_hostnames" {
  type        = bool
  description = "Enable DNS hostnames in the VPC"
  default     = true
}

variable "enable_dns_support" {
  type        = bool
  description = "Enable DNS support in the VPC"
  default     = true
}

variable "enable_nat_gateway" {
  type        = bool
  description = "Create NAT Gateway(s) for private subnet internet access"
  default     = true
}

variable "single_nat_gateway" {
  type        = bool
  description = "Use a single NAT Gateway instead of one per AZ (cost savings for non-prod)"
  default     = false
}

variable "enable_vpn_gateway" {
  type        = bool
  description = "Create a VPN Gateway for hybrid connectivity"
  default     = false
}

variable "enable_flow_logs" {
  type        = bool
  description = "Enable VPC Flow Logs for network monitoring"
  default     = true
}

# =============================================================================
# Tagging
# =============================================================================

variable "tags" {
  type        = map(string)
  description = "A map of tags to add to all resources"
  default     = {}
}
```

### main.tf

Following the skill's module template patterns and cost optimization guidance (VPC endpoints to reduce NAT charges, single NAT gateway option for non-prod):

```hcl
# =============================================================================
# Data Sources
# =============================================================================

data "aws_availability_zones" "available" {
  state = "available"
}

# =============================================================================
# Locals
# =============================================================================

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, var.az_count)

  common_tags = merge(
    var.tags,
    {
      Module      = "vpc"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  )

  # Determine NAT Gateway count: single for cost savings, or one per AZ for HA
  nat_gateway_count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : var.az_count) : 0
}

# =============================================================================
# VPC
# =============================================================================

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-vpc"
    }
  )
}

# =============================================================================
# Internet Gateway
# =============================================================================

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-igw"
    }
  )
}

# =============================================================================
# Public Subnets
# =============================================================================

resource "aws_subnet" "public" {
  count = var.az_count

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-public-${local.azs[count.index]}"
      Tier = "public"
    }
  )
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-public-rt"
    }
  )
}

resource "aws_route_table_association" "public" {
  count = var.az_count

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# =============================================================================
# Private Subnets
# =============================================================================

resource "aws_subnet" "private" {
  count = var.az_count

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.private_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = false

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-private-${local.azs[count.index]}"
      Tier = "private"
    }
  )
}

resource "aws_route_table" "private" {
  count = var.az_count

  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-private-rt-${local.azs[count.index]}"
    }
  )
}

resource "aws_route_table_association" "private" {
  count = var.az_count

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# =============================================================================
# NAT Gateway(s)
# =============================================================================

resource "aws_eip" "nat" {
  count  = local.nat_gateway_count
  domain = "vpc"

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-nat-eip-${count.index}"
    }
  )

  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = local.nat_gateway_count

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-nat-${count.index}"
    }
  )

  depends_on = [aws_internet_gateway.main]
}

resource "aws_route" "private_nat" {
  count = var.enable_nat_gateway ? var.az_count : 0

  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = var.single_nat_gateway ? aws_nat_gateway.main[0].id : aws_nat_gateway.main[count.index].id
}

# =============================================================================
# VPC Flow Logs (Security Best Practice)
# =============================================================================

resource "aws_flow_log" "main" {
  count = var.enable_flow_logs ? 1 : 0

  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"
  log_destination_type = "cloud-watch-logs"
  log_destination      = aws_cloudwatch_log_group.flow_logs[0].arn
  iam_role_arn         = aws_iam_role.flow_logs[0].arn

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-flow-logs"
    }
  )
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name              = "/vpc/flow-logs/${var.name}"
  retention_in_days = var.environment == "prod" ? 90 : 14

  tags = local.common_tags
}

resource "aws_iam_role" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.name}-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "vpc-flow-logs.amazonaws.com"
      }
    }]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.name}-flow-logs-policy"
  role = aws_iam_role.flow_logs[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Effect   = "Allow"
      Resource = "*"
    }]
  })
}
```

### outputs.tf

Following the skill's guidance to "output useful values" with descriptions on every output:

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "public_subnet_cidrs" {
  description = "List of CIDR blocks of public subnets"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_cidrs" {
  description = "List of CIDR blocks of private subnets"
  value       = aws_subnet.private[*].cidr_block
}

output "nat_gateway_ips" {
  description = "Elastic IPs of NAT gateways (for firewall whitelisting)"
  value       = aws_eip.nat[*].public_ip
}

output "internet_gateway_id" {
  description = "The ID of the Internet Gateway"
  value       = aws_internet_gateway.main.id
}

output "public_route_table_id" {
  description = "The ID of the public route table"
  value       = aws_route_table.public.id
}

output "private_route_table_ids" {
  description = "List of IDs of private route tables"
  value       = aws_route_table.private[*].id
}

output "availability_zones" {
  description = "List of availability zones used"
  value       = local.azs
}
```

---

## Step 3: Environment Configurations

Following the skill's recommended project structure, create environment-specific configurations that reference the module:

### Directory Layout

```
terraform-project/
├── modules/
│   └── vpc/                    # The reusable module (from above)
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars
│       └── backend.tf
└── README.md
```

### environments/dev/terraform.tfvars

```hcl
environment          = "dev"
name                 = "myapp-dev"
vpc_cidr             = "10.0.0.0/16"
az_count             = 2
public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
single_nat_gateway   = true    # Cost savings: single NAT for dev
enable_flow_logs     = false   # Optional for dev
```

### environments/staging/terraform.tfvars

```hcl
environment          = "staging"
name                 = "myapp-staging"
vpc_cidr             = "10.1.0.0/16"
az_count             = 2
public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24"]
private_subnet_cidrs = ["10.1.10.0/24", "10.1.11.0/24"]
single_nat_gateway   = true    # Cost savings: single NAT for staging
enable_flow_logs     = true
```

### environments/prod/terraform.tfvars

```hcl
environment          = "prod"
name                 = "myapp-prod"
vpc_cidr             = "10.2.0.0/16"
az_count             = 3
public_subnet_cidrs  = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
private_subnet_cidrs = ["10.2.10.0/24", "10.2.11.0/24", "10.2.12.0/24"]
single_nat_gateway   = false   # HA: one NAT per AZ in prod
enable_flow_logs     = true
```

### environments/dev/main.tf (same pattern for staging/prod)

```hcl
module "vpc" {
  source = "../../modules/vpc"

  name                 = var.name
  environment          = var.environment
  vpc_cidr             = var.vpc_cidr
  az_count             = var.az_count
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  single_nat_gateway   = var.single_nat_gateway
  enable_flow_logs     = var.enable_flow_logs

  tags = {
    Project    = "myapp"
    CostCenter = "engineering"
  }
}
```

### environments/dev/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "dev/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

---

## Step 4: CIDR Range Design Rationale

The CIDR allocation follows these principles:

| Environment | VPC CIDR       | Public Subnets                        | Private Subnets                          | AZs |
|-------------|----------------|---------------------------------------|------------------------------------------|-----|
| dev         | 10.0.0.0/16    | 10.0.1.0/24, 10.0.2.0/24             | 10.0.10.0/24, 10.0.11.0/24              | 2   |
| staging     | 10.1.0.0/16    | 10.1.1.0/24, 10.1.2.0/24             | 10.1.10.0/24, 10.1.11.0/24              | 2   |
| prod        | 10.2.0.0/16    | 10.2.1.0/24, 10.2.2.0/24, 10.2.3.0/24| 10.2.10.0/24, 10.2.11.0/24, 10.2.12.0/24| 3   |

Key design decisions:
- **Non-overlapping CIDRs** -- enables VPC peering between environments if needed later
- **Second octet differentiates environments** -- easy to identify at a glance
- **/16 VPCs** -- room for growth (65,536 IPs each)
- **/24 subnets** -- 251 usable IPs per subnet, sufficient for most workloads
- **Separate public/private ranges** -- public in 10.x.1-9.0/24, private in 10.x.10-19.0/24 for clarity
- **3 AZs for prod** -- higher availability; 2 AZs for dev/staging saves cost

---

## Step 5: Cost Optimization per Environment

Following the skill's cost optimization reference (`references/cost_optimization.md`), here are the key cost differences by environment:

| Feature            | Dev                  | Staging              | Prod                      |
|--------------------|----------------------|----------------------|---------------------------|
| NAT Gateway        | Single (save ~$32/mo)| Single (save ~$32/mo)| One per AZ (HA)           |
| Flow Logs          | Disabled             | Enabled (14-day retention) | Enabled (90-day retention)|
| AZ Count           | 2                    | 2                    | 3                         |
| Estimated VPC cost | ~$35/mo              | ~$70/mo              | ~$130/mo                  |

Additional cost optimization to consider (from the cost optimization reference):
- Add **VPC endpoints for S3 and DynamoDB** (gateway endpoints are free) to reduce NAT data transfer charges
- Add **interface endpoints** for ECR, STS, etc. if running containers
- Use **scheduled shutdown** for dev resources during non-business hours

---

## Step 6: Validate the Module

After implementing, run the skill's validation script:

```bash
python3 scripts/validate_module.py modules/vpc
```

This checks:
- All required files present (main.tf, variables.tf, outputs.tf)
- All variables have descriptions and type constraints
- All outputs have descriptions
- Sensitive values properly marked
- README exists with Usage, Inputs, and Outputs sections
- Version constraints specified in versions.tf
- Examples directory exists with working configurations
- Naming conventions (snake_case for resources and files)
- No hardcoded values (like hardcoded regions)

---

## Step 7: Test the Module

```bash
# Test with the dev environment first (as per skill's "test in non-production first" principle)
cd environments/dev
terraform init
terraform validate
terraform plan -out=tfplan

# Review the plan carefully, then apply
terraform apply tfplan

# Verify outputs
terraform output
```

---

## Step 8: Quality Checklist

From the skill's code review checklist:

- [x] All variables have descriptions
- [x] Sensitive values marked as sensitive (none needed for VPC)
- [x] Outputs have descriptions
- [x] Resources follow naming conventions (`${var.name}-resource-type`)
- [x] No hardcoded values (all configurable via variables)
- [x] README is complete and current
- [x] Examples directory exists and works
- [x] Version constraints specified (Terraform >= 1.5.0, AWS provider ~> 5.0)
- [x] Security best practices followed (private subnets, flow logs, no public IPs on private subnets)

---

## Step 9: Generate Documentation

```bash
cd modules/vpc
terraform-docs markdown . > README.md
```

---

## Optional: Terragrunt Alternative

If your team uses Terragrunt, follow the skill's Terragrunt patterns section. The structure would be:

```
terragrunt-project/
├── terragrunt.hcl              # Root config (provider, remote state)
├── account.hcl
├── region.hcl
└── environments/
    ├── dev/
    │   ├── env.hcl             # environment = "dev"
    │   └── us-east-1/
    │       └── vpc/
    │           └── terragrunt.hcl  # References the VPC module with dev inputs
    ├── staging/
    │   ├── env.hcl
    │   └── us-east-1/
    │       └── vpc/
    │           └── terragrunt.hcl
    └── prod/
        ├── env.hcl
        └── us-east-1/
            └── vpc/
                └── terragrunt.hcl
```

Each environment's `terragrunt.hcl` would reference the shared VPC module with environment-specific inputs, inheriting provider and backend configuration from the root `terragrunt.hcl`.

---

## Summary

This approach follows every principle from the iac-terraform skill:

1. **Module-first**: Reusable VPC module with single responsibility
2. **Sensible defaults**: DNS enabled, flow logs on, works out of the box
3. **Environment parity**: Same module, different parameters per environment
4. **Security by default**: Private subnets without public IPs, VPC flow logs, encryption
5. **Cost-conscious**: Single NAT for non-prod, optional flow logs, fewer AZs in dev
6. **Fully documented**: Descriptions on all variables and outputs, README, examples
7. **Validated**: Can be checked with `validate_module.py` before use
8. **Remote state**: Separate state files per environment with S3 backend and DynamoDB locking
