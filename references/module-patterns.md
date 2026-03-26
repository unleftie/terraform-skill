# Module Development Patterns

> **Part of:** [terraform-skill](../SKILL.md)
> **Purpose:** Best practices for Terraform/OpenTofu module development

This document provides detailed guidance on creating reusable, maintainable Terraform modules. For high-level principles, see the [main skill file](../SKILL.md#core-principles).

---

## Table of Contents

1. [Architecture Principles](#architecture-principles)
2. [Module Structure](#module-structure)
3. [Variable Best Practices](#variable-best-practices)
4. [Output Best Practices](#output-best-practices)
5. [Common Patterns](#common-patterns)
6. [Anti-patterns to Avoid](#anti-patterns-to-avoid)
7. [Testing Philosophy & Patterns](#testing-philosophy--patterns)

---

## Architecture Principles

### 2. Always Use Remote State

**Why:**

- **Prevents race conditions** with multiple developers
- **Provides disaster recovery** (state versioning)
- **Enables team collaboration** (shared access)
- **Supports state locking** (prevents concurrent modifications)

**Never:**

```hcl
# ❌ BAD - Local state (default)
# State stored in local terraform.tfstate file
# Lost if computer crashes
# Can't share with team
```

**Always:**

```hcl
# ✅ GOOD - Remote state
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # State locking
    encrypt        = true                # Encryption at rest
  }
}
```

### 3. Use terraform_remote_state as Glue

**Pattern:** Connect compositions via remote state data sources

**Why:**

- Loose coupling between infrastructure components
- Teams can work independently
- Changes to one stack don't require rebuilding others
- Outputs from one stack become inputs to another

**Example:**

```hcl
# environments/prod/networking/outputs.tf
output "vpc_id" {
  description = "ID of the production VPC"
  value       = aws_vpc.this.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

# environments/prod/compute/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

module "ec2" {
  source = "../../modules/ec2"

  vpc_id     = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
}
```

**Best practices:**

- Use remote state for cross-team dependencies
- Document which outputs are consumed by other stacks
- Version outputs (don't break downstream consumers)
- Consider using data sources instead for provider-managed resources

### 4. Keep Resource Modules Simple

**Principles:**

- Don't hardcode values
- Use variables for all configurable parameters
- Use data sources for external dependencies
- Focus on single responsibility

**Example:**

```hcl
# ❌ BAD - Hardcoded values in resource module
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Hardcoded
  instance_type = "t3.large"               # Hardcoded
  subnet_id     = "subnet-12345678"        # Hardcoded

  tags = {
    Environment = "production"             # Hardcoded
  }
}

# ✅ GOOD - Parameterized resource module
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id != "" ? var.ami_id : data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = var.tags
}
```

---

## Module Structure

### Standard Layout

```
my-module/
├── README.md                # Usage documentation
├── main.tf                  # Primary resources
├── variables.tf             # Input variables with descriptions
├── outputs.tf               # Output values
├── examples/
│   └── main/                # Minimal working example
```

### Why This Structure?

- **README.md** - First thing users see, should explain module purpose
- **main.tf** - Primary resources, keep focused
- **variables.tf** - All inputs in one place with descriptions
- **outputs.tf** - All outputs documented
- **examples/** - Serve as both documentation and test fixtures. Also we can run `terraform validate` in examples to catch issues early.

---

## Variable Best Practices

### Complete Example

```hcl
variable "instance_type" {
  description = "EC2 instance type for the application server"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
  }
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}

variable "enable_monitoring" {
  description = "Enable CloudWatch detailed monitoring"
  type        = bool
  default     = true
}
```

### Variable Naming

```hcl
# ✅ Good: Context-specific
var.vpc_cidr_block          # Not just "cidr"
var.database_instance_class # Not just "instance_class"
var.application_port        # Not just "port"

# ❌ Bad: Generic names
var.name
var.type
var.value
```

---

## Output Best Practices

### Complete Example

```hcl
output "instance_id" {
  description = "ID of the created EC2 instance"
  value       = aws_instance.this.id
}

output "instance_arn" {
  description = "ARN of the created EC2 instance"
  value       = aws_instance.this.arn
}

output "private_ip" {
  description = "Private IP address of the instance"
  value       = aws_instance.this.private_ip
  sensitive   = false  # Explicitly document sensitivity
}

output "connection_info" {
  description = "Connection information for the instance"
  value = {
    id         = aws_instance.this.id
    private_ip = aws_instance.this.private_ip
    public_dns = aws_instance.this.public_dns
  }
}
```

### Key Principles

- ✅ **Always include `description`** - Explain what the output is for
- ✅ **Mark sensitive outputs** - Use `sensitive = true`
- ✅ **Return objects for related values** - Groups logically related data
- ✅ **Document intended use** - What should consumers do with this?

---

## Common Patterns

### ✅ DO: Use `for_each` for Resources

```hcl
# Good: Maintain stable resource addresses
resource "aws_instance" "server" {
  for_each = toset(["web", "api", "worker"])

  instance_type = "t3.micro"
  tags = {
    Name = each.key
  }
}
```

**Why?** When you remove an item from the middle, `for_each` doesn't reshuffle other resources.

### ❌ DON'T: Use `count` When Order Matters

```hcl
# Bad: Removing middle item reshuffles all subsequent resources
resource "aws_instance" "server" {
  count = length(var.server_names)

  tags = {
    Name = var.server_names[count.index]
  }
}
```

**Problem:** If you remove `var.server_names[1]`, Terraform will destroy and recreate all instances after it.

---

## Anti-patterns to Avoid

### ❌ DON'T: Hard-code Environment-Specific Values

```hcl
# Bad: Module is locked to production
resource "aws_instance" "app" {
  instance_type = "m5.large"  # Should be variable
  tags = {
    Environment = "prod" # Should be variable
  }
}
```

**Fix:** Make everything configurable:

```hcl
resource "aws_instance" "app" {
  instance_type = var.instance_type
  tags          = var.tags
}
```

### ❌ DON'T: Create God Modules

```hcl
# Bad: One module does everything
module "everything" {
  source = "./modules/app-infrastructure"

  # Creates VPC, EC2, RDS, S3, IAM, CloudWatch, etc.
}
```

**Problem:** Hard to test, hard to reuse, hard to maintain.

**Fix:** Break into focused modules:

```hcl
module "networking" {
  source = "./modules/vpc"
}

module "compute" {
  source = "./modules/ec2"
  vpc_id = module.networking.vpc_id
}

module "database" {
  source = "./modules/rds"
  vpc_id = module.networking.vpc_id
}
```

### ❌ DON'T: Use `count` or `for_each` in Root Modules for Different Environments

```hcl
# Bad: All environments in one root module
resource "aws_instance" "app" {
  for_each = toset(["test", "stage", "prod"])

  instance_type = each.key == "prod" ? "m5.large" : "t3.micro"
}
```

**Problem:** Can't have separate state files, blast radius is huge.

---

## Module Naming Conventions

### Public Modules

Follow the Terraform Registry convention:

```
<PROVIDER>-<NAME>-terraform-module

Examples:
aws-vpc-terraform-module
aws-eks-terraform-module
google-network-terraform-module
```

### Private Modules

Use organization-specific prefixes:

```
<ORG>-<PROVIDER>-<NAME>-terraform-module

Examples:
acme-vpc-terraform-module
acme-rds-terraform-module
```

---

## Testing Philosophy & Patterns

### What to Test in Terraform Modules

**Core testing areas:**

- **Input validation** - Variables accept valid values and reject invalid ones
- **Resource creation** - Resources are created as expected with correct attributes
- **Output correctness** - Outputs return expected values and types

### Testing Layers

**1. Syntax validation:**

```bash
terraform fmt -check -recursive
```

**2. Configuration validity:**

```bash
cd examples/main
terraform validate
```

**3. Plan preview:**

```bash
cd examples/main
terraform plan
# Review: Are expected resources being created?
# Verify: Count and types of resources match expectations
```

### Input Validation Testing

Test that variables reject invalid values:

```hcl
# In variables.tf
variable "environment" {
  description = "Environment name"
  type        = string

  validation {
    condition     = contains(["test", "stage", "prod"], var.environment)
    error_message = "Environment must be one of: test, stage, prod."
  }
}

# Test: terraform plan with invalid value should fail
# terraform plan -var="environment=invalid"
# Expected: Error message about validation failure
```

**For testing framework details, see:** [Testing Frameworks Guide](testing-frameworks.md)

---

**Back to:** [Main Skill File](../SKILL.md)
