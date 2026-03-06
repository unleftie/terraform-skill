# Terraform Skill for Claude

[![Claude Skill](https://img.shields.io/badge/Claude-Skill-5865F2)](https://docs.claude.ai/docs/agent-skills)
[![Terraform](https://img.shields.io/badge/Terraform-1.0+-623CE4)](https://www.terraform.io/)
[![OpenTofu](https://img.shields.io/badge/OpenTofu-1.6+-FFD814)](https://opentofu.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

Comprehensive Terraform and OpenTofu best practices skill for Claude Code. Get instant guidance on testing strategies, module patterns, CI/CD workflows, and production-ready infrastructure code.

## What This Skill Provides

📦 **Module Development**

- Structure and naming conventions
- Versioning strategies
- Public vs private module patterns

🔒 **Security & Compliance**

- Trivy, Checkov integration
- Policy-as-code patterns
- Compliance scanning workflows

## Installation

```bash
git clone https://github.com/unleftie/terraform-skill ~/.claude/skills/terraform-skill
```

### Verify Installation

After installation, try:

```
"Create a Terraform module with testing for an S3 bucket"
```

Claude will automatically use the skill when working with Terraform/OpenTofu code.

## Quick Start Examples

**Create a module with tests:**

> "Create a Terraform module for AWS VPC with native tests"

**Review existing code:**

> "Review this Terraform configuration following best practices"

**Generate CI/CD workflow:**

> "Create a GitHub Actions workflow for Terraform with cost estimation"

**Testing strategy:**

> "Help me choose between native tests and Terratest for my modules"

## What It Covers

### Module Development Patterns

- Naming conventions (`terraform-<PROVIDER>-<NAME>`)
- Directory structure best practices
- Input variable organization
- Output value design
- Version constraint patterns
- Documentation standards

### Common Patterns & Anti-patterns

Side-by-side ✅ DO vs ❌ DON'T examples for:

- Variable naming
- Resource naming
- Module composition
- State management
- Provider configuration

## Requirements

- **Claude Code** or other Claude environment supporting skills
- **Terraform** 1.0+ or **OpenTofu** 1.6+
- Optional: MCP Terraform server for enhanced registry integration

## Releases

Releases are automated based on conventional commits in commit messages:

| Commit Type                    | Version Bump    | Example       |
| ------------------------------ | --------------- | ------------- |
| `feat!:` or `BREAKING CHANGE:` | Major           | 1.2.3 → 2.0.0 |
| `feat:`                        | Minor           | 1.2.3 → 1.3.0 |
| `fix:`                         | Patch           | 1.2.3 → 1.2.4 |
| Other commits                  | Patch (default) | 1.2.3 → 1.2.4 |

Releases are created automatically when changes are pushed to master.

## Related Resources

### Official Documentation

- [Terraform Language](https://developer.hashicorp.com/terraform/docs) - HashiCorp official docs
- [Terraform Testing](https://developer.hashicorp.com/terraform/language/tests) - Native test framework
- [OpenTofu Documentation](https://opentofu.org/docs/) - OpenTofu official docs
- [HashiCorp Best Practices](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices) - Cloud best practices

### Community Resources

- [Awesome Terraform](https://github.com/shuaibiyy/awesome-tf)
- [Terraform Best Practices](https://terraform-best-practices.com) - Comprehensive guide (base for this skill)
- [terraform-aws-modules](https://github.com/terraform-aws-modules) - Production-grade AWS modules
- [Terratest](https://terratest.gruntwork.io/docs/) - Go testing framework for Terraform
- [Google Cloud Best Practices](https://docs.cloud.google.com/docs/terraform/best-practices/general-style-structure)
- [AWS Terraform Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/introduction.html)

### Development Tools

- [pre-commit-terraform](https://github.com/antonbabenko/pre-commit-terraform) - Pre-commit hooks for Terraform
- [terraform-docs](https://terraform-docs.io/) - Generate documentation from Terraform modules
- [terraform-switcher](https://github.com/warrensbox/terraform-switcher) - Terraform version manager
- [TFLint](https://github.com/terraform-linters/tflint) - Terraform linter
- [Trivy](https://github.com/aquasecurity/trivy) - Security scanner for IaC

## License & Attribution

**License:** Apache 2.0 - see [LICENSE](LICENSE)

```
Based on terraform-skill by Anton Babenko
https://github.com/antonbabenko/terraform-skill
terraform-best-practices.com | Compliance.tf
```
