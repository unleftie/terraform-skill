# Testing Frameworks - Detailed Guide

> **Part of:** [terraform-skill](../SKILL.md)
> **Purpose:** Detailed guides for Terraform/OpenTofu testing frameworks

This document provides in-depth guidance on testing frameworks for Infrastructure as Code. For the decision matrix and high-level overview, see the [main skill file](../SKILL.md#testing-strategy-framework).

---

## Table of Contents

1. [Static Analysis](#static-analysis)

---

## Static Analysis

### Pre-commit Hooks

```bash
[[ ! -f .git/hooks/pre-commit ]] && pre-commit install --config ~/.config/pre-commit/.pre-commit-config.yaml
pre-commit run --config ~/.config/pre-commit/.pre-commit-config.yaml -a
```

### What pre-commit hooks checks

- **`terraform fmt`** - Code formatting consistency
- **`terraform validate`** - Syntax and internal consistency
- **`TFLint`** - Best practices, provider-specific rules
- **`trivy` / `checkov`** - Security vulnerabilities

### When to Use

Every commit, always. Zero cost, catches 40%+ of issues.

**Back to:** [Main Skill File](../SKILL.md)
