# Reusable Terraform VPC Module for Multi-Environment Setup

Here is a complete reusable Terraform module for your VPC setup supporting dev, staging, and prod environments with different CIDR ranges.

## Module Structure

```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
    locals.tf
environments/
  dev/
    main.tf
    terraform.tfvars
    backend.tf
  staging/
    main.tf
    terraform.tfvars
    backend.tf
  prod/
    main.tf
    terraform.tfvars
    backend.tf
```

---

## Module Code

### `modules/vpc/variables.tf`

```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr must be a valid CIDR block."
  }
}

variable "availability_zones" {
  description = "List of availability zones to use"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets (one per AZ)"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets (one per AZ)"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Whether to create NAT Gateway(s) for private subnet internet access"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use a single NAT Gateway instead of one per AZ (cost saving for non-prod)"
  type        = bool
  default     = false
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames in the VPC"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Enable DNS support in the VPC"
  type        = bool
  default     = true
}

variable "enable_flow_logs" {
  description = "Enable VPC Flow Logs"
  type        = bool
  default     = false
}

variable "flow_logs_retention_days" {
  description = "Number of days to retain VPC Flow Logs in CloudWatch"
  type        = number
  default     = 30
}

variable "tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

### `modules/vpc/locals.tf`

```hcl
locals {
  common_tags = merge(
    {
      Environment = var.environment
      ManagedBy   = "terraform"
      Module      = "vpc"
    },
    var.tags
  )

  # Determine how many NAT Gateways to create
  nat_gateway_count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.availability_zones)) : 0
}
```

### `modules/vpc/main.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# =============================================================================
# VPC
# =============================================================================
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc"
  })
}

# =============================================================================
# Internet Gateway
# =============================================================================
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-igw"
  })
}

# =============================================================================
# Public Subnets
# =============================================================================
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.environment}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  })
}

# =============================================================================
# Private Subnets
# =============================================================================
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  })
}

# =============================================================================
# Elastic IPs for NAT Gateways
# =============================================================================
resource "aws_eip" "nat" {
  count  = local.nat_gateway_count
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.this]
}

# =============================================================================
# NAT Gateways
# =============================================================================
resource "aws_nat_gateway" "this" {
  count = local.nat_gateway_count

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-nat-${count.index + 1}"
  })

  depends_on = [aws_internet_gateway.this]
}

# =============================================================================
# Public Route Table
# =============================================================================
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-public-rt"
  })
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this.id
}

resource "aws_route_table_association" "public" {
  count = length(var.public_subnet_cidrs)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# =============================================================================
# Private Route Tables (one per AZ for HA, or shared if single NAT)
# =============================================================================
resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.availability_zones)) : 1
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-private-rt-${count.index + 1}"
  })
}

resource "aws_route" "private_nat" {
  count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.availability_zones)) : 0

  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.this[var.single_nat_gateway ? 0 : count.index].id
}

resource "aws_route_table_association" "private" {
  count = length(var.private_subnet_cidrs)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[var.single_nat_gateway ? 0 : count.index].id
}

# =============================================================================
# VPC Flow Logs (optional)
# =============================================================================
resource "aws_cloudwatch_log_group" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name              = "/aws/vpc/flow-logs/${var.environment}"
  retention_in_days = var.flow_logs_retention_days

  tags = local.common_tags
}

resource "aws_iam_role" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.environment}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.environment}-vpc-flow-logs-policy"
  role = aws_iam_role.flow_logs[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}

resource "aws_flow_log" "this" {
  count = var.enable_flow_logs ? 1 : 0

  vpc_id                   = aws_vpc.this.id
  traffic_type             = "ALL"
  iam_role_arn             = aws_iam_role.flow_logs[0].arn
  log_destination          = aws_cloudwatch_log_group.flow_logs[0].arn
  max_aggregation_interval = 60

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc-flow-log"
  })
}

# =============================================================================
# Default Security Group (restrict all traffic)
# =============================================================================
resource "aws_default_security_group" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-default-sg-restricted"
  })
}

# =============================================================================
# Default Network ACL
# =============================================================================
resource "aws_default_network_acl" "this" {
  default_network_acl_id = aws_vpc.this.default_network_acl_id

  ingress {
    protocol   = -1
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  egress {
    protocol   = -1
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-default-nacl"
  })

  lifecycle {
    ignore_changes = [subnet_ids]
  }
}
```

### `modules/vpc/outputs.tf`

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}

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

output "nat_gateway_ips" {
  description = "List of NAT Gateway public IPs"
  value       = aws_eip.nat[*].public_ip
}

output "internet_gateway_id" {
  description = "The ID of the Internet Gateway"
  value       = aws_internet_gateway.this.id
}

output "public_route_table_id" {
  description = "The ID of the public route table"
  value       = aws_route_table.public.id
}

output "private_route_table_ids" {
  description = "List of private route table IDs"
  value       = aws_route_table.private[*].id
}

output "default_security_group_id" {
  description = "The ID of the default security group (restricted)"
  value       = aws_default_security_group.this.id
}
```

---

## Environment Configurations

### `environments/dev/terraform.tfvars`

```hcl
environment = "dev"
vpc_cidr    = "10.0.0.0/16"

availability_zones   = ["us-east-1a", "us-east-1b"]
public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]

# Cost savings: single NAT Gateway is fine for dev
enable_nat_gateway = true
single_nat_gateway = true

# Flow logs not needed in dev
enable_flow_logs = false

tags = {
  Team = "engineering"
}
```

### `environments/staging/terraform.tfvars`

```hcl
environment = "staging"
vpc_cidr    = "10.1.0.0/16"

availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
private_subnet_cidrs = ["10.1.10.0/24", "10.1.11.0/24", "10.1.12.0/24"]

# Single NAT for staging to save cost, but 3 AZs to mirror prod topology
enable_nat_gateway = true
single_nat_gateway = true

enable_flow_logs         = true
flow_logs_retention_days = 14

tags = {
  Team = "engineering"
}
```

### `environments/prod/terraform.tfvars`

```hcl
environment = "prod"
vpc_cidr    = "10.2.0.0/16"

availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
public_subnet_cidrs  = ["10.2.1.0/24", "10.2.2.0/24", "10.2.3.0/24"]
private_subnet_cidrs = ["10.2.10.0/24", "10.2.11.0/24", "10.2.12.0/24"]

# HA: one NAT Gateway per AZ in production
enable_nat_gateway = true
single_nat_gateway = false

enable_flow_logs         = true
flow_logs_retention_days = 90

tags = {
  Team        = "engineering"
  Compliance  = "required"
}
```

### `environments/dev/main.tf` (same pattern for staging/prod)

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

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Project   = "my-project"
      ManagedBy = "terraform"
    }
  }
}

module "vpc" {
  source = "../../modules/vpc"

  environment          = var.environment
  vpc_cidr             = var.vpc_cidr
  availability_zones   = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  enable_nat_gateway   = var.enable_nat_gateway
  single_nat_gateway   = var.single_nat_gateway
  enable_dns_hostnames = true
  enable_dns_support   = true
  enable_flow_logs     = var.enable_flow_logs
  flow_logs_retention_days = var.flow_logs_retention_days
  tags                 = var.tags
}

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnet_ids" {
  value = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  value = module.vpc.private_subnet_ids
}

output "nat_gateway_ips" {
  value = module.vpc.nat_gateway_ips
}
```

### `environments/dev/variables.tf` (same for staging/prod)

```hcl
variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type = string
}

variable "availability_zones" {
  type = list(string)
}

variable "public_subnet_cidrs" {
  type = list(string)
}

variable "private_subnet_cidrs" {
  type = list(string)
}

variable "enable_nat_gateway" {
  type    = bool
  default = true
}

variable "single_nat_gateway" {
  type    = bool
  default = false
}

variable "enable_flow_logs" {
  type    = bool
  default = false
}

variable "flow_logs_retention_days" {
  type    = number
  default = 30
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

### `environments/dev/backend.tf` (adjust bucket/key per environment)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-project-terraform-state"
    key            = "dev/vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

For staging and prod, change the `key` value accordingly:
- Staging: `"staging/vpc/terraform.tfstate"`
- Prod: `"prod/vpc/terraform.tfstate"`

---

## CIDR Allocation Summary

| Environment | VPC CIDR       | Public Subnets                        | Private Subnets                         | AZs | NAT GWs |
|-------------|---------------|----------------------------------------|-----------------------------------------|-----|---------|
| dev         | 10.0.0.0/16   | 10.0.1.0/24, 10.0.2.0/24             | 10.0.10.0/24, 10.0.11.0/24             | 2   | 1       |
| staging     | 10.1.0.0/16   | 10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24| 10.1.10.0/24, 10.1.11.0/24, 10.1.12.0/24| 3   | 1       |
| prod        | 10.2.0.0/16   | 10.2.1.0/24, 10.2.2.0/24, 10.2.3.0/24| 10.2.10.0/24, 10.2.11.0/24, 10.2.12.0/24| 3   | 3       |

Each /16 VPC gives you 65,536 IP addresses. Each /24 subnet provides 251 usable IPs (256 minus 5 AWS-reserved addresses). The non-overlapping second octets (10.0, 10.1, 10.2) ensure the VPCs can be peered if needed.

---

## Key Design Decisions

### 1. Cost Optimization per Environment
- **Dev**: 2 AZs, single NAT Gateway (~$32/month instead of ~$64 for two). No flow logs.
- **Staging**: 3 AZs to match prod topology, but single NAT Gateway to save cost. Flow logs with short retention.
- **Prod**: 3 AZs, one NAT Gateway per AZ for high availability (if one AZ fails, private subnets in other AZs retain internet access). Flow logs with 90-day retention.

### 2. Security Hardening
- **Default security group is locked down**: The `aws_default_security_group` resource is imported with no rules, preventing accidental use of the default SG with open access.
- **VPC Flow Logs**: Enabled in staging and prod for network traffic auditing and troubleshooting.
- **DNS support enabled**: Required for private hosted zones and service discovery.

### 3. Non-Overlapping CIDRs
The CIDR ranges use `10.0.0.0/16`, `10.1.0.0/16`, and `10.2.0.0/16` so that:
- VPC peering between environments is straightforward (no overlapping routes).
- Future VPN or Transit Gateway connectivity does not require re-addressing.
- There is room for additional subnets (database tier, cache tier, etc.) within each /16.

### 4. Extensibility
To add more subnet tiers (e.g., a database tier), you would add another variable like `database_subnet_cidrs` and another `aws_subnet` resource block following the same pattern. The module is structured to make this straightforward.

---

## Usage

```bash
# Initialize and apply dev environment
cd environments/dev
terraform init
terraform plan
terraform apply

# Same for staging and prod
cd ../staging
terraform init
terraform plan
terraform apply

cd ../prod
terraform init
terraform plan
terraform apply
```

---

## Optional Enhancements to Consider

1. **Database subnets**: Add a third subnet tier with a DB subnet group for RDS.
2. **VPC endpoints**: Add S3 and DynamoDB gateway endpoints (free) and interface endpoints for ECR, CloudWatch, etc. to reduce NAT Gateway data processing costs.
3. **Transit Gateway**: If you need connectivity between environments, add a Transit Gateway attachment output.
4. **IPv6**: Add `assign_generated_ipv6_cidr_block = true` to the VPC and dual-stack subnets if needed.
5. **Terragrunt**: If you want to reduce the boilerplate across environments, wrap this module with Terragrunt using a `terragrunt.hcl` per environment that sets the variables and backend config automatically.
