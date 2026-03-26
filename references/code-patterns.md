# Code Patterns & Structure

> **Part of:** [terraform-skill](../SKILL.md)
> **Purpose:** Comprehensive patterns for Terraform/OpenTofu code structure and modern features

This document provides detailed code patterns, structure guidelines, and modern Terraform features. For high-level principles, see the [main skill file](../SKILL.md).

---

## Table of Contents

1. [Count vs For_Each Deep Dive](#count-vs-for_each-deep-dive)
2. [Modern Terraform Features (1.0+)](#modern-terraform-features-10)
3. [Version Management](#version-management)
4. [Refactoring Patterns](#refactoring-patterns)
5. [Locals for Dependency Management](#locals-for-dependency-management)

---

## Count vs For_Each Deep Dive

### When to use count

✓ **Simple numeric replication:**

```hcl
resource "aws_subnet" "public" {
  count = 3

  cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index)
}
```

✓ **Boolean conditions (create or don't):**

```hcl
# ✅ GOOD - Boolean condition
resource "aws_nat_gateway" "this" {
  count = var.create_nat_gateway ? 1 : 0
}

# Less preferred - length check
resource "aws_nat_gateway" "this" {
  count = length(var.public_subnets) > 0 ? 1 : 0
}
```

✓ **When order doesn't matter and items won't change**

### When to use for_each

✓ **Reference resources by key:**

```hcl
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, index(var.availability_zones, each.key))
}

# Reference by key: aws_subnet.private["us-east-1a"]
```

✓ **Items may be added/removed from middle:**

```hcl
# ❌ BAD with count - removing middle item recreates all subsequent resources
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  availability_zone = var.availability_zones[count.index]
  # If var.availability_zones[1] removed, all resources after recreated!
}

# ✅ GOOD with for_each - removal only affects that one resource
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  availability_zone = each.key
  # Removing one AZ only destroys that subnet
}
```

✓ **Creating multiple named resources:**

```hcl
variable "environments" {
  default = {
    test = {
      instance_type = "t3.micro"
      instance_count = 1
    }
    prod = {
      instance_type = "t3.large"
      instance_count = 3
    }
  }
}

resource "aws_instance" "app" {
  for_each = var.environments

  instance_type = each.value.instance_type
  count         = each.value.instance_count

  tags = {
    Environment = each.key  # "test" or "prod"
  }
}
```

### Count to For_Each Migration

**When to migrate:** When you need stable resource addressing or items might be added/removed from middle of list.

**Migration steps:**

1. Add `for_each` to resource
2. Use `moved` blocks to preserve existing resources
3. Remove `count` after verifying with `terraform plan`

**Complete example:**

```hcl
# Before (using count)
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-${var.availability_zones[count.index]}"
  }
}

# Reference: aws_subnet.private[0].id

# After (using for_each)
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.key))
  availability_zone = each.key

  tags = {
    Name = "private-${each.key}"
  }
}

# Reference: aws_subnet.private["us-east-1a"].id

# Migration blocks (prevents resource recreation)
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}

moved {
  from = aws_subnet.private[1]
  to   = aws_subnet.private["us-east-1b"]
}

moved {
  from = aws_subnet.private[2]
  to   = aws_subnet.private["us-east-1c"]
}

# Verify migration:
# terraform plan should show "moved" operations, not destroy/create
```

**Benefits after migration:**

- Removing "us-east-1b" only destroys that subnet (not c)
- Adding new AZ doesn't affect existing subnets
- Resources have stable addresses by AZ name

---

## Modern Terraform Features (1.0+)

### try() Function (Terraform 0.13+)

**Use try() instead of element(concat()):**

```hcl
# ✅ GOOD - Modern try() function
output "security_group_id" {
  description = "The ID of the security group"
  value       = try(aws_security_group.this[0].id, "")
}

output "first_subnet_id" {
  description = "ID of first subnet with multiple fallbacks"
  value       = try(
    aws_subnet.public[0].id,
    aws_subnet.private[0].id,
    ""
  )
}

# ❌ BAD - Legacy pattern
output "security_group_id" {
  value = element(concat(aws_security_group.this.*.id, [""]), 0)
}
```

### nullable = false (Terraform 1.1+)

**Set nullable = false for non-null variables:**

```hcl
# ✅ GOOD (Terraform 1.1+)
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  nullable    = false  # Passing null uses default, not null
  default     = "10.0.0.0/16"
}
```

### Moved Blocks (Terraform 1.1+)

**Rename resources without destroy/recreate:**

```hcl
# Rename a resource
moved {
  from = aws_instance.web_server
  to   = aws_instance.web
}

# Rename a module
moved {
  from = module.old_module_name
  to   = module.new_module_name
}

# Move resource into for_each
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}
```

### Provider-Defined Functions (Terraform 1.8+)

**Use provider-specific functions for data transformation:**

```hcl
# AWS provider function example
data "aws_region" "current" {}

locals {
  # Provider function (Terraform 1.8+)
  bucket_name = provider::aws::arn_build("s3", "my-bucket", data.aws_region.current.name)
}

# Check provider documentation for available functions
# Common providers adding functions: AWS, Azure, Google Cloud
```

### Cross-Variable Validation (Terraform 1.9+)

**Reference other variables in validation blocks:**

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
}

variable "storage_size" {
  description = "Storage size in GB"
  type        = number

  validation {
    # Can reference var.instance_type in Terraform 1.9+
    condition = !(
      var.instance_type == "db.t3.micro" &&
      var.storage_size > 1000
    )
    error_message = "Micro instances cannot have storage > 1000 GB"
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "backup_retention" {
  description = "Backup retention period in days"
  type        = number

  validation {
    # Production requires longer retention
    condition = (
      var.environment == "prod" ? var.backup_retention >= 7 : true
    )
    error_message = "Production environment requires backup_retention >= 7 days"
  }
}
```

### Write-Only Arguments (Terraform 1.11+)

**Always use write-only arguments or external secret management:**

```hcl
# ✅ GOOD - External secret with write-only argument
data "aws_secretsmanager_secret" "db_password" {
  name = "prod-database-password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "this" {
  engine         = "mysql"
  instance_class = "db.t3.micro"
  username       = "admin"

  # write-only: Terraform sends to AWS then forgets it (not in state)
  password_wo = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# ❌ BAD - Secret ends up in state file
resource "random_password" "db" {
  length = 16
}

resource "aws_db_instance" "this" {
  password = random_password.db.result  # Stored in state!
}

# ❌ BAD - Variable secret stored in state
resource "aws_db_instance" "this" {
  password = var.db_password  # Ends up in state file
}
```

---

## Version Management

### Version Constraint Syntax

```hcl
# Exact version (avoid unless necessary - inflexible)
version = "5.0.0"

# Pessimistic constraint (recommended for stability)
# Allows patch updates only
version = "~> 5.0"      # Allows 5.0.x (any x), but not 5.1.0
version = "~> 5.0.1"    # Allows 5.0.x where x >= 1, but not 5.1.0

# Range constraints
version = ">= 5.0, < 6.0"     # Any 5.x version
version = ">= 5.0.0, < 5.1.0" # Specific minor version range

# Minimum version
version = ">= 5.0"  # Any version 5.0 or higher (risky - breaking changes)

# Latest (avoid in production - unpredictable)
# No version specified = always use latest available
```

### Versioning Strategy by Component

**Terraform itself:**

```hcl
# versions.tf
terraform {
  # Pin to minor version, allow patch updates
  required_version = "~> 1.9"  # Allows 1.9.x
}
```

**Providers:**

```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Pin major version, allow minor/patch updates
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
```

**Modules:**

```hcl
# Production - pin exact version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"  # Exact version for production stability
}

# Development - allow flexibility
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"  # Allow patch updates in test
}
```

---

## Refactoring Patterns

### Secrets Remediation

**Pattern:** Move secrets out of Terraform state into external secret management.

#### Before - Secrets in State

```hcl
# ❌ BAD - Secret generated and stored in state
resource "random_password" "db" {
  length  = 16
  special = true
}

resource "aws_db_instance" "this" {
  engine   = "mysql"
  username = "admin"
  password = random_password.db.result  # In state!
}

# OR

# ❌ BAD - Secret passed via variable and stored in state
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # Marked sensitive but still in state!
}

resource "aws_db_instance" "this" {
  password = var.db_password  # In state!
}
```

#### After - External Secret Management

**Option 1: Write-only arguments (Terraform 1.11+)**

```hcl
# ✅ GOOD - Fetch from AWS Secrets Manager
data "aws_secretsmanager_secret" "db_password" {
  name = "prod-database-password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "this" {
  engine   = "mysql"
  username = "admin"

  # write-only: Sent to AWS, not stored in state
  password_wo = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

**Option 2: Separate secret creation (if Terraform 1.11+ not available)**

```hcl
# ✅ GOOD - Reference pre-existing secret
# Secret created outside Terraform (manually or separate process)

data "aws_secretsmanager_secret" "db_password" {
  name = "prod-database-password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

# Note: Without write-only, you may need to handle secret rotation
# outside Terraform or accept that the secret value appears in state
# during initial creation but not after rotation
```

**Migration steps:**

1. Create secret in AWS Secrets Manager (outside Terraform)
2. Update Terraform to use data sources
3. Use write-only argument (if Terraform 1.11+)
4. Remove `random_password` resource or variable
5. Run `terraform apply` to update
6. Verify secret not in state: `terraform show` should not display password

---

## Locals for Dependency Management

**Use locals to hint explicit resource deletion order:**

```hcl
# ✅ GOOD - Forces correct deletion order
# Ensures subnets deleted before secondary CIDR blocks

locals {
  # References secondary CIDR first, falling back to VPC
  # This forces Terraform to delete subnets before CIDR association
  vpc_id = try(
    aws_vpc_ipv4_cidr_block_association.this[0].vpc_id,
    aws_vpc.this.id,
    ""
  )
}

resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc_ipv4_cidr_block_association" "this" {
  count = var.add_secondary_cidr ? 1 : 0

  vpc_id     = aws_vpc.this.id
  cidr_block = "10.1.0.0/16"
}

resource "aws_subnet" "public" {
  # Uses local instead of direct reference
  # Creates implicit dependency on CIDR association
  vpc_id     = local.vpc_id
  cidr_block = "10.1.0.0/24"
}

# Without local: Terraform might try to delete CIDR before subnets → ERROR
# With local: Subnets deleted first, then CIDR association, then VPC ✓
```

**Why this matters:**

- Prevents deletion errors when destroying infrastructure
- Ensures correct dependency order without explicit `depends_on`
- Particularly useful for complex VPC configurations with secondary CIDR blocks

**Common use cases:**

- VPC with secondary CIDR blocks
- Resources that depend on optional configurations
- Complex deletion order requirements

---

**Back to:** [Main Skill File](../SKILL.md)
