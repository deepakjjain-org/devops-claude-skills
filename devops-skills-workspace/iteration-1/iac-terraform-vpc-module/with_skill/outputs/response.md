# Reusable Terraform VPC Module for Multi-Environment Deployment

## Decision Tree Result

Following the skill's workflow decision tree:

> *Is this reusable across environments/projects?* **Yes** -- You need the same VPC pattern across dev, staging, and prod with different CIDR ranges.

This means we create a **reusable Terraform module** and then consume it from per-environment configurations.

---

## Step 1: Scaffold the Module

Use the skill's scaffolding script to generate the standard structure:

```bash
python3 scripts/init_module.py vpc --path ./modules
```

This creates:

```
modules/vpc/
├── main.tf                    # Primary resources
├── variables.tf               # Input variables
├── outputs.tf                 # Output values
├── versions.tf                # Version constraints
├── README.md                  # Documentation
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

Following the skill's best practices -- every variable has a description, a type, and validation where appropriate. Sensitive-named variables are marked `sensitive = true`. Sensible defaults are provided for optional parameters.

```hcl
# ------------------------------------
# Required Variables
# ------------------------------------

variable "name" {
  type        = string
  description = "Name prefix for all VPC resources (e.g., 'myapp')"
}

variable "environment" {
  type        = string
  description = "Environment name (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC (e.g., 10.0.0.0/16)"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid IPv4 CIDR block."
  }
}

# ------------------------------------
# Network Configuration
# ------------------------------------

variable "az_count" {
  type        = number
  description = "Number of Availability Zones to use (2 or 3 recommended)"
  default     = 3

  validation {
    condition     = var.az_count >= 2 && var.az_count <= 4
    error_message = "AZ count must be between 2 and 4."
  }
}

variable "public_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for public subnets (one per AZ). If empty, calculated automatically from vpc_cidr."
  default     = []

  validation {
    condition     = alltrue([for cidr in var.public_subnet_cidrs : can(cidrhost(cidr, 0))])
    error_message = "All entries must be valid IPv4 CIDR blocks."
  }
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "CIDR blocks for private subnets (one per AZ). If empty, calculated automatically from vpc_cidr."
  default     = []

  validation {
    condition     = alltrue([for cidr in var.private_subnet_cidrs : can(cidrhost(cidr, 0))])
    error_message = "All entries must be valid IPv4 CIDR blocks."
  }
}

# ------------------------------------
# Feature Flags
# ------------------------------------

variable "enable_nat_gateway" {
  type        = bool
  description = "Enable NAT Gateway for private subnets (set to false for dev to save costs)"
  default     = true
}

variable "single_nat_gateway" {
  type        = bool
  description = "Use a single NAT Gateway instead of one per AZ (cheaper, less HA)"
  default     = false
}

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

variable "enable_flow_logs" {
  type        = bool
  description = "Enable VPC Flow Logs (recommended for prod)"
  default     = false
}

variable "flow_log_retention_days" {
  type        = number
  description = "Number of days to retain VPC flow logs"
  default     = 30
}

variable "enable_vpn_gateway" {
  type        = bool
  description = "Create a VPN Gateway for hybrid connectivity"
  default     = false
}

# ------------------------------------
# Tagging
# ------------------------------------

variable "tags" {
  type        = map(string)
  description = "Common tags to apply to all resources"
  default     = {}
}
```

### main.tf

```hcl
# ------------------------------------
# Data Sources
# ------------------------------------

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_region" "current" {}

# ------------------------------------
# Locals
# ------------------------------------

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, var.az_count)

  # Auto-calculate subnet CIDRs if not provided
  # Public subnets get the first N /20 blocks, private subnets get the next N
  public_subnet_cidrs = length(var.public_subnet_cidrs) > 0 ? var.public_subnet_cidrs : [
    for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 4, i)
  ]

  private_subnet_cidrs = length(var.private_subnet_cidrs) > 0 ? var.private_subnet_cidrs : [
    for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 4, i + var.az_count)
  ]

  # Determine NAT Gateway count
  nat_gateway_count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : var.az_count) : 0

  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Module      = "vpc"
    }
  )
}

# ------------------------------------
# VPC
# ------------------------------------

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-vpc"
    }
  )
}

# ------------------------------------
# Internet Gateway
# ------------------------------------

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-igw"
    }
  )
}

# ------------------------------------
# Public Subnets
# ------------------------------------

resource "aws_subnet" "public" {
  count = var.az_count

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    local.common_tags,
    {
      Name                     = "${var.name}-${var.environment}-public-${local.azs[count.index]}"
      Tier                     = "public"
      "kubernetes.io/role/elb" = "1"
    }
  )
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-public-rt"
    }
  )
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}

resource "aws_route_table_association" "public" {
  count = var.az_count

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ------------------------------------
# Private Subnets
# ------------------------------------

resource "aws_subnet" "private" {
  count = var.az_count

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.private_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = false

  tags = merge(
    local.common_tags,
    {
      Name                              = "${var.name}-${var.environment}-private-${local.azs[count.index]}"
      Tier                              = "private"
      "kubernetes.io/role/internal-elb" = "1"
    }
  )
}

# ------------------------------------
# NAT Gateway (conditional)
# ------------------------------------

resource "aws_eip" "nat" {
  count  = local.nat_gateway_count
  domain = "vpc"

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-nat-eip-${count.index + 1}"
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
      Name = "${var.name}-${var.environment}-nat-${count.index + 1}"
    }
  )

  depends_on = [aws_internet_gateway.main]
}

resource "aws_route_table" "private" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : var.az_count) : 1

  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-private-rt-${count.index + 1}"
    }
  )
}

resource "aws_route" "private_nat" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : var.az_count) : 0

  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = var.single_nat_gateway ? aws_nat_gateway.main[0].id : aws_nat_gateway.main[count.index].id
}

resource "aws_route_table_association" "private" {
  count = var.az_count

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = var.single_nat_gateway || !var.enable_nat_gateway ? aws_route_table.private[0].id : aws_route_table.private[count.index].id
}

# ------------------------------------
# VPC Flow Logs (conditional)
# ------------------------------------

resource "aws_flow_log" "main" {
  count = var.enable_flow_logs ? 1 : 0

  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"
  iam_role_arn         = aws_iam_role.flow_log[0].arn
  log_destination      = aws_cloudwatch_log_group.flow_log[0].arn
  log_destination_type = "cloud-watch-logs"
  max_aggregation_interval = 60

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-flow-log"
    }
  )
}

resource "aws_cloudwatch_log_group" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0

  name              = "/aws/vpc-flow-log/${var.name}-${var.environment}"
  retention_in_days = var.flow_log_retention_days

  tags = local.common_tags
}

resource "aws_iam_role" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.name}-${var.environment}-flow-log-role"

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

resource "aws_iam_role_policy" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.name}-${var.environment}-flow-log-policy"
  role = aws_iam_role.flow_log[0].id

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

# ------------------------------------
# VPN Gateway (conditional)
# ------------------------------------

resource "aws_vpn_gateway" "main" {
  count = var.enable_vpn_gateway ? 1 : 0

  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-${var.environment}-vgw"
    }
  )
}
```

### outputs.tf

Following the skill's guidance to output useful values with complete descriptions:

```hcl
# ------------------------------------
# VPC
# ------------------------------------

output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "vpc_arn" {
  description = "The ARN of the VPC"
  value       = aws_vpc.main.arn
}

# ------------------------------------
# Subnets
# ------------------------------------

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnet_cidrs" {
  description = "List of public subnet CIDR blocks"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_cidrs" {
  description = "List of private subnet CIDR blocks"
  value       = aws_subnet.private[*].cidr_block
}

output "availability_zones" {
  description = "List of Availability Zones used"
  value       = local.azs
}

# ------------------------------------
# Gateways
# ------------------------------------

output "internet_gateway_id" {
  description = "The ID of the Internet Gateway"
  value       = aws_internet_gateway.main.id
}

output "nat_gateway_ids" {
  description = "List of NAT Gateway IDs"
  value       = aws_nat_gateway.main[*].id
}

output "nat_gateway_public_ips" {
  description = "Elastic IPs of NAT Gateways (useful for firewall whitelisting)"
  value       = aws_eip.nat[*].public_ip
}

# ------------------------------------
# Route Tables
# ------------------------------------

output "public_route_table_id" {
  description = "ID of the public route table"
  value       = aws_route_table.public.id
}

output "private_route_table_ids" {
  description = "List of private route table IDs"
  value       = aws_route_table.private[*].id
}

# ------------------------------------
# VPN Gateway
# ------------------------------------

output "vpn_gateway_id" {
  description = "The ID of the VPN Gateway (null if not enabled)"
  value       = var.enable_vpn_gateway ? aws_vpn_gateway.main[0].id : null
}
```

---

## Step 3: Environment Configurations

Following the skill's recommended project structure, create per-environment directories that consume the module:

```
terraform-project/
├── modules/
│   └── vpc/                      # The module from Step 2
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
name        = "myapp"
environment = "dev"
vpc_cidr    = "10.0.0.0/16"

# Cost optimization: single NAT gateway in dev
enable_nat_gateway = true
single_nat_gateway = true

# Dev only needs 2 AZs
az_count = 2

# No flow logs in dev
enable_flow_logs = false

tags = {
  Project      = "MyApp"
  CostCenter   = "engineering"
  AutoShutdown = "enabled"
}
```

### environments/staging/terraform.tfvars

```hcl
name        = "myapp"
environment = "staging"
vpc_cidr    = "10.1.0.0/16"

# Staging uses single NAT for cost savings
enable_nat_gateway = true
single_nat_gateway = true

az_count = 3

# Flow logs enabled for testing observability
enable_flow_logs        = true
flow_log_retention_days = 14

tags = {
  Project    = "MyApp"
  CostCenter = "engineering"
}
```

### environments/prod/terraform.tfvars

```hcl
name        = "myapp"
environment = "prod"
vpc_cidr    = "10.2.0.0/16"

# Production: full HA -- one NAT per AZ
enable_nat_gateway = true
single_nat_gateway = false

az_count = 3

# Full flow logs for compliance and security
enable_flow_logs        = true
flow_log_retention_days = 90

tags = {
  Project    = "MyApp"
  CostCenter = "engineering"
}
```

### environments/dev/main.tf (same pattern for staging/prod)

```hcl
module "vpc" {
  source = "../../modules/vpc"

  name        = var.name
  environment = var.environment
  vpc_cidr    = var.vpc_cidr

  az_count           = var.az_count
  enable_nat_gateway = var.enable_nat_gateway
  single_nat_gateway = var.single_nat_gateway
  enable_flow_logs   = var.enable_flow_logs

  flow_log_retention_days = var.flow_log_retention_days

  tags = var.tags
}
```

### environments/dev/backend.tf

```hcl
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "dev/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "myapp-terraform-lock"
  }
}
```

Each environment gets its own state key (`dev/vpc/`, `staging/vpc/`, `prod/vpc/`) to prevent accidental cross-environment changes.

---

## Step 4: CIDR Allocation Summary

The three environments use non-overlapping /16 ranges, allowing VPC peering if needed later:

| Environment | VPC CIDR       | Public Subnets                    | Private Subnets                   | NAT Strategy     |
|-------------|----------------|-----------------------------------|-----------------------------------|------------------|
| dev         | 10.0.0.0/16    | 10.0.0.0/20, 10.0.16.0/20        | 10.0.32.0/20, 10.0.48.0/20       | Single NAT       |
| staging     | 10.1.0.0/16    | 10.1.0.0/20, 10.1.16.0/20, 10.1.32.0/20 | 10.1.48.0/20, 10.1.64.0/20, 10.1.80.0/20 | Single NAT |
| prod        | 10.2.0.0/16    | 10.2.0.0/20, 10.2.16.0/20, 10.2.32.0/20 | 10.2.48.0/20, 10.2.64.0/20, 10.2.80.0/20 | One NAT per AZ |

Each /20 subnet provides 4,091 usable IP addresses -- more than sufficient for most workloads.

---

## Step 5: Validate the Module

Run the skill's validation script to check the module against best practices:

```bash
python3 scripts/validate_module.py modules/vpc
```

This verifies:
- All required files are present (main.tf, variables.tf, outputs.tf)
- Every variable has a description and type
- Every output has a description
- README.md exists with Usage, Inputs, and Outputs sections
- versions.tf specifies required_version and required_providers
- examples/ directory exists with working examples
- Naming conventions are followed (snake_case)
- Sensitive-looking variables are marked as sensitive

---

## Step 6: Cost Optimization Notes

Following the skill's cost optimization reference (`references/cost_optimization.md`):

1. **NAT Gateway costs**: NAT Gateways cost ~$32/month each plus data processing charges. The module defaults to a single NAT in dev/staging (`single_nat_gateway = true`) and one-per-AZ in prod. In dev, you could even set `enable_nat_gateway = false` and use VPC endpoints for AWS service access if workloads do not need general internet.

2. **VPC Endpoints**: Consider adding VPC endpoints for S3, DynamoDB, ECR, and other frequently used AWS services to reduce NAT Gateway data processing charges. The module outputs the VPC ID and route table IDs so you can attach endpoints at the environment level.

3. **Flow Logs**: Disabled in dev by default. For prod, the CloudWatch Logs destination is used -- for higher volume environments, consider switching to S3 with lifecycle policies to reduce long-term storage costs.

---

## Step 7: Deployment Workflow

Following the skill's standard Terraform workflow:

```bash
# 1. Deploy dev first
cd environments/dev
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan

# 2. Verify dev, then deploy staging
cd ../staging
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# 3. Finally, deploy production with extra review
cd ../prod
terraform init
terraform plan -out=tfplan
# Have another team member review the plan output
terraform apply tfplan
```

---

## Step 8: Next Steps

1. **Generate documentation**: Run `terraform-docs markdown modules/vpc > modules/vpc/README.md` after finalizing variables and outputs.

2. **Security scan**: Run `checkov -d modules/vpc` to catch security misconfigurations.

3. **Version the module**: Once stable, tag with `git tag v1.0.0` and reference by version in environments:
   ```hcl
   source = "git::https://github.com/your-org/terraform-modules.git//vpc?ref=v1.0.0"
   ```

4. **Add VPC endpoints**: Create a companion module or add endpoints at the environment level for S3/DynamoDB/ECR.

5. **Consider Terragrunt**: If you have many environments or accounts, use Terragrunt to DRY up the backend/provider configuration. The skill's MODULE_TEMPLATE.md includes complete Terragrunt configuration examples for root and module-level `terragrunt.hcl` files.

6. **CI/CD integration**: Use the skill's provided GitHub Actions or GitLab CI templates in `assets/workflows/` to automate plan-on-PR and apply-on-merge workflows.
