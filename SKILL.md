---
name: terraform-skill
description: Use when working with Terraform or OpenTofu - creating modules, reviewing configurations, or making infrastructure-as-code architecture decisions
license: Apache-2.0
---

# Terraform Skill for Claude

Comprehensive Terraform and OpenTofu guidance covering testing, modules, CI/CD, and production patterns. Based on terraform-best-practices.com and enterprise experience.

## When to Use This Skill

**Activate this skill when:**

- Creating new Terraform or OpenTofu configurations or modules
- Setting up testing infrastructure for IaC code
- Deciding between testing approaches (validate, plan, frameworks)
- Reviewing or refactoring existing Terraform/OpenTofu projects

**Don't use this skill for:**

- Basic Terraform/OpenTofu syntax questions (Claude knows this)
- Provider-specific API reference (link to docs instead)
- Cloud platform questions unrelated to Terraform/OpenTofu

## Core Principles

### 1. Code Structure Philosophy

**Module Hierarchy:**

| Type                      | When to Use                                  | Scope                                           |
| ------------------------- | -------------------------------------------- | ----------------------------------------------- |
| **Resource Module**       | Single logical group of connected resources  | VPC + subnets, Security group + rules           |
| **Infrastructure Module** | Collection of resource modules for a purpose | Multiple resource modules in one region/account |
| **Composition**           | Complete infrastructure                      | Spans multiple regions/accounts                 |

**Hierarchy:** Resource → Resource Module → Infrastructure Module → Composition

**Directory Structure:**

```
environments/        # Environment-specific configurations
├── prod/
├── stage/
└── test/

modules/            # Reusable modules
├── networking/
├── compute/
└── data/

examples/           # Module usage examples (also serve as tests)
├── complete/
└── minimal/
```

**Key principle from terraform-best-practices.com:**

- Separate **environments** (prod, stage) from **modules** (reusable components)
- Use **examples/** as both documentation and integration test fixtures
- Keep modules small and focused (single responsibility)

**For detailed module architecture, see:** [Code Patterns: Module Types & Hierarchy](references/code-patterns.md)

### 2. Naming Conventions

**Resources:**

```hcl
# Good: Descriptive, contextual
resource "aws_instance" "web_server" { }
resource "aws_s3_bucket" "application_logs" { }

# Good: "this" for singleton resources (only one of that type)
resource "aws_vpc" "this" { }
resource "aws_security_group" "this" { }

# Avoid: Generic names for non-singletons
resource "aws_instance" "main" { }
resource "aws_s3_bucket" "bucket" { }
```

**Singleton Resources:**

Use `"this"` when your module creates only one resource of that type:

✅ DO:

```hcl
resource "aws_vpc" "this" {}           # Module creates one VPC
resource "aws_security_group" "this" {}  # Module creates one SG
```

❌ DON'T use "this" for multiple resources:

```hcl
resource "aws_subnet" "this" {}  # If creating multiple subnets
```

Use descriptive names when creating multiple resources of the same type.

**Variables:**

```hcl
# Prefix with context when needed
var.vpc_cidr_block          # Not just "cidr"
var.database_instance_class # Not just "instance_class"
```

**Files:**

- `main.tf` - Primary resources
- `variables.tf` - Input variables
- `outputs.tf` - Output values
- `versions.tf` - Provider versions
- `data.tf` - Data sources (optional)

## Code Structure Standards

## Count vs For_Each: When to Use Each

### Quick Decision Guide

| Scenario                            | Use                         | Why                                 |
| ----------------------------------- | --------------------------- | ----------------------------------- |
| Boolean condition (create or don't) | `count = condition ? 1 : 0` | Simple on/off toggle                |
| Simple numeric replication          | `count = 3`                 | Fixed number of identical resources |
| Items may be reordered/removed      | `for_each = toset(list)`    | Stable resource addresses           |
| Reference by key                    | `for_each = map`            | Named access to resources           |
| Multiple named resources            | `for_each`                  | Better maintainability              |

### Common Patterns

**Boolean conditions:**

```hcl
# ✅ GOOD - Boolean condition
resource "aws_nat_gateway" "this" {
  count = var.create_nat_gateway ? 1 : 0
  # ...
}
```

**Stable addressing with for_each:**

```hcl
# ✅ GOOD - Removing "us-east-1b" only affects that subnet
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  availability_zone = each.key
  # ...
}

# ❌ BAD - Removing middle AZ recreates all subsequent subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  availability_zone = var.availability_zones[count.index]
  # ...
}
```

**For migration guides and detailed examples, see:** [Code Patterns: Count vs For_Each](references/code-patterns.md#count-vs-for_each-deep-dive)

## Locals for Dependency Management

**Use locals to ensure correct resource deletion order:**

```hcl
# Problem: Subnets might be deleted after CIDR blocks, causing errors
# Solution: Use try() in locals to hint deletion order

locals {
  # References secondary CIDR first, falling back to VPC
  # Forces Terraform to delete subnets before CIDR association
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
  vpc_id     = local.vpc_id  # Uses local, not direct reference
  cidr_block = "10.1.0.0/24"
}
```

**Why this matters:**

- Prevents deletion errors when destroying infrastructure
- Ensures correct dependency order without explicit `depends_on`
- Particularly useful for VPC configurations with secondary CIDR blocks

**For detailed examples, see:** [Code Patterns: Locals for Dependency Management](references/code-patterns.md#locals-for-dependency-management)

## Module Development

### Best Practices Summary

**Variables:**

- ✅ Always include `description`
- ✅ Use explicit `type` constraints
- ✅ Provide sensible `default` values where appropriate
- ✅ Add `validation` blocks for complex constraints
- ✅ Use `sensitive = true` for secrets

**Outputs:**

- ✅ Always include `description`
- ✅ Mark sensitive outputs with `sensitive = true`
- ✅ Consider returning objects for related values
- ✅ Document what consumers should do with each output

**For detailed module patterns, see:**

- **[Module Patterns Guide](references/module-patterns.md)** - Variable best practices, output design, ✅ DO vs ❌ DON'T patterns

## Version Management

### Version Constraint Syntax

```hcl
version = "5.0.0"      # Exact (avoid - inflexible)
version = "~> 5.0"     # Recommended: 5.0.x only
version = ">= 5.0"     # Minimum (risky - breaking changes)
```

### Strategy by Component

| Component     | Strategy          | Example                       |
| ------------- | ----------------- | ----------------------------- |
| **Terraform** | Pin minor version | `required_version = "~> 1.9"` |
| **Providers** | Pin major version | `version = "~> 5.0"`          |
| **Modules**   | Pin exact version | `version = "5.1.2"`           |

## Modern Terraform Features (1.0+)

### Feature Availability by Version

| Feature                    | Version | Use Case                                     |
| -------------------------- | ------- | -------------------------------------------- |
| `try()` function           | 0.13+   | Safe fallbacks, replaces `element(concat())` |
| `nullable = false`         | 1.1+    | Prevent null values in variables             |
| `moved` blocks             | 1.1+    | Refactor without destroy/recreate            |
| `optional()` with defaults | 1.3+    | Optional object attributes                   |
| Provider functions         | 1.8+    | Provider-specific data transformation        |
| Cross-variable validation  | 1.9+    | Validate relationships between variables     |
| Write-only arguments       | 1.11+   | Secrets never stored in state                |

### Quick Examples

```hcl
# try() - Safe fallbacks (0.13+)
output "sg_id" {
  value = try(aws_security_group.this[0].id, "")
}

# optional() - Optional attributes with defaults (1.3+)
variable "config" {
  type = object({
    name    = string
    timeout = optional(number, 300)  # Default: 300
  })
}

# Cross-variable validation (1.9+)
variable "environment" { type = string }
variable "backup_days" {
  type = number
  validation {
    condition     = var.environment == "prod" ? var.backup_days >= 7 : true
    error_message = "Production requires backup_days >= 7"
  }
}
```

**For complete patterns and examples, see:** [Code Patterns: Modern Terraform Features](references/code-patterns.md#modern-terraform-features-10)

## Detailed Guides

This skill uses **progressive disclosure** - essential information is in this main file, detailed guides are available when needed:

📚 **Reference Files:**

- **[Testing Frameworks](references/testing-frameworks.md)** - In-depth guide to static analysis, native tests, and Terratest
- **[Module Patterns](references/module-patterns.md)** - Module structure, variable/output best practices, ✅ DO vs ❌ DON'T patterns
- **[Code Patterns](references/code-patterns.md)** - Count vs For_Each, modern features, refactoring patterns, locals for dependencies

**How to use:** When you need detailed information on a topic, reference the appropriate guide. Claude will load it on demand to provide comprehensive guidance.

## License

This skill is licensed under the **Apache License 2.0**. See the LICENSE file for full terms.

**Copyright © 2026 Anton Babenko**
