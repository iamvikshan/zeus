---
name: terraform-patterns
description: >-
  Terraform best practices for file structure, naming, variables, outputs,
  resource ordering, version pinning, remote state, secrets, module composition,
  and count vs for_each. Use when writing, reviewing, or refactoring Terraform
  configurations, modules, or compositions.
---

<!-- See NOTICE.md for attribution (antonbabenko/terraform-best-practices Apache 2.0) -->

# Terraform Patterns

Canonical patterns for Terraform configurations. Apply these rules when writing
new modules, reviewing infrastructure code, or refactoring existing HCL.

---

## 1. File Structure Convention

Every Terraform root module or child module follows this layout:

```
module/
  main.tf           # Resources and data sources
  variables.tf      # All input variables
  outputs.tf        # All outputs
  versions.tf       # terraform {} block with required_version and required_providers
  locals.tf         # Local values (optional, only if non-trivial)
  README.md         # Module documentation
```

- `terraform.tfvars` belongs ONLY in compositions (environment-specific root
  modules), never in reusable modules.
- One resource type per file is acceptable for large modules (e.g.,
  `iam.tf`, `networking.tf`), but keep the four core files present.

**Incorrect:**

```hcl
# Everything dumped in a single main.tf with no separation
variable "name" {}
resource "aws_instance" "web" { ... }
output "ip" { value = aws_instance.web.public_ip }
terraform { required_version = ">= 1.0" }
```

**Correct:**

```hcl
# versions.tf
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# variables.tf
variable "instance_type" {
  type        = string
  description = "EC2 instance type."
  default     = "t3.micro"
}

# main.tf
resource "aws_instance" "this" {
  instance_type = var.instance_type
  ami           = data.aws_ami.ubuntu.id
}

# outputs.tf
output "instance_public_ip" {
  description = "Public IP address of the instance."
  value       = aws_instance.this.public_ip
}
```

---

## 2. Naming Rules

- Use `_` (underscore) in all Terraform identifiers: resource names, data
  source names, variable names, output names, locals.
- Use `-` (hyphen) only in human-facing string values: DNS names, S3 bucket
  names, tag values.
- Use **singular nouns** for resource and module names.
- Do NOT repeat the resource type in the resource name.
- Use `this` as the name if only one resource of a given type exists in the
  module.

**Incorrect:**

```hcl
resource "aws_security_group" "aws-security-group-web" {
  name = "web_sg"  # hyphen in identifier, underscore in human name
}

resource "aws_instance" "instance" {
  # "instance" repeats the resource type
}
```

**Correct:**

```hcl
resource "aws_security_group" "web" {
  name = "web-sg"  # underscore in identifier, hyphen in human name
}

resource "aws_instance" "this" {
  # only one instance in this module -- use "this"
}
```

---

## 3. Variable Structure

Variables follow a consistent field ordering. Every variable MUST have a
`description`.

**Field order:** `type` -> `description` -> `default` -> `sensitive` -> `validation`

Use plural form for collection variables.

**Incorrect:**

```hcl
variable "sg" {
  default = ["0.0.0.0/0"]
}
```

**Correct:**

```hcl
variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "CIDR blocks permitted to reach the load balancer."
  default     = []

  validation {
    condition     = alltrue([for cidr in var.allowed_cidr_blocks : can(cidrhost(cidr, 0))])
    error_message = "Each element must be a valid CIDR block."
  }
}

variable "database_password" {
  type        = string
  description = "Master password for the RDS instance."
  sensitive   = true
}
```

---

## 4. Output Naming Pattern

Outputs follow the pattern: `{resource_name}_{resource_type}_{attribute}`.
Every output MUST have a `description`.

**Incorrect:**

```hcl
output "ip" {
  value = aws_instance.this.public_ip
}
```

**Correct:**

```hcl
output "this_instance_public_ip" {
  description = "Public IP address of the EC2 instance."
  value       = aws_instance.this.public_ip
}

output "this_security_group_id" {
  description = "ID of the security group."
  value       = aws_security_group.this.id
}
```

---

## 5. Resource Argument Order

Arguments within a resource or data source follow a consistent order:

1. **Meta-arguments first** (`count` or `for_each`), followed by a blank line
2. **Required arguments** (no default, must be supplied)
3. **Optional arguments** (have defaults or are nullable)
4. **Tags** (always last before lifecycle blocks)
5. **depends_on**, **lifecycle** (at the very end, after a blank line)

**Incorrect:**

```hcl
resource "aws_instance" "this" {
  tags          = { Name = "web" }
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  depends_on = [aws_security_group.this]
  count      = var.create ? 1 : 0
}
```

**Correct:**

```hcl
resource "aws_instance" "this" {
  count = var.create ? 1 : 0

  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = {
    Name = "web"
  }

  depends_on = [aws_security_group.this]

  lifecycle {
    create_before_destroy = true
  }
}
```

---

## 6. Version Pinning

Pin ALL version constraints. Never use a provider or module without a version.

- **Terraform core**: Use `~>` pessimistic constraint (e.g., `~> 1.9`).
- **Providers**: Pin to minor version (e.g., `~> 5.0`).
- **Modules**: Pin to exact or minor version. Never omit `version`.

**Incorrect:**

```hcl
terraform {
  required_version = ">= 1.0"
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # no version constraint -- will pull latest
}
```

**Correct:**

```hcl
terraform {
  required_version = "~> 1.9"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

---

## 7. Remote State

- **Never** commit `*.tfstate` or `*.tfstate.backup` to version control.
  Add both to `.gitignore`.
- Use remote state (S3 + DynamoDB, GCS, Terraform Cloud, etc.) from day 1.
- Prefer **data sources** over `terraform_remote_state` for cross-stack
  references. `terraform_remote_state` exposes the entire state, while data
  sources query only what is needed.

**Incorrect:**

```hcl
# Reading full state of another stack
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "this" {
  subnet_id = data.terraform_remote_state.network.outputs.subnet_id
}
```

**Correct:**

```hcl
# Query only the specific resource needed
data "aws_subnet" "selected" {
  filter {
    name   = "tag:Name"
    values = ["web-subnet"]
  }
}

resource "aws_instance" "this" {
  subnet_id = data.aws_subnet.selected.id
}
```

---

## 8. Secrets Discipline

- **Never** hardcode secrets in `.tf` files, `terraform.tfvars`, or variable
  defaults.
- Use dynamic provider credentials: OIDC federation, HashiCorp Vault, cloud
  IAM roles.
- Mark secret variables with `sensitive = true`.
- Add `*.tfvars` to `.gitignore` in compositions (or use a secrets manager
  to inject at runtime).

**Incorrect:**

```hcl
variable "db_password" {
  type    = string
  default = "SuperSecret123!"
}

provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

**Correct:**

```hcl
variable "db_password" {
  type        = string
  description = "Master password for the database."
  sensitive   = true
  # No default -- injected via environment or secrets manager
}

provider "aws" {
  # Credentials via OIDC, IAM role, or environment variables
  # Never hardcoded
}
```

---

## 9. Module Composition Layers

Terraform modules organize into three tiers. Do NOT deep-chain modules
(module calling module calling module).

| Layer                     | Purpose                                               | Example                                      |
| ------------------------- | ----------------------------------------------------- | -------------------------------------------- |
| **Resource module**       | Wraps 1-3 closely related resources                   | `modules/rds`, `modules/security_group`      |
| **Infrastructure module** | Composes resource modules for a logical group         | `modules/app_cluster` (calls rds + ec2 + sg) |
| **Composition**           | Environment-specific root module, calls infra modules | `environments/production/main.tf`            |

**Incorrect:**

```hcl
# Deep chain: composition -> infra module -> resource module -> nested module
module "level1" {
  source = "./modules/wrapper"
  # which calls another module, which calls another...
}
```

**Correct:**

```
environments/
  production/
    main.tf            # Composition -- calls infrastructure modules
    terraform.tfvars
  staging/
    main.tf
    terraform.tfvars
modules/
  app_cluster/         # Infrastructure module -- composes resource modules
    main.tf
  rds/                 # Resource module -- single purpose
    main.tf
    variables.tf
    outputs.tf
  security_group/      # Resource module -- single purpose
    main.tf
    variables.tf
    outputs.tf
```

---

## 10. count vs for_each

| Use Case                        | Mechanism  | Reason                                 |
| ------------------------------- | ---------- | -------------------------------------- |
| Boolean toggle (create/skip)    | `count`    | `count = var.create ? 1 : 0`           |
| Collection of similar resources | `for_each` | Stable keys, no index shift on removal |

- **Never** combine `count` and `for_each` on the same resource.
- **Avoid** `length()` in `count` -- if items are removed from the middle,
  index-based resources shift and get destroyed/recreated.
- `for_each` requires a `map` or `set` of strings. Convert lists with
  `toset()`.

**Incorrect:**

```hcl
# Using count with a list -- removing item 0 destroys and recreates 1, 2, ...
variable "subnet_names" {
  type    = list(string)
  default = ["web", "app", "db"]
}

resource "aws_subnet" "this" {
  count      = length(var.subnet_names)
  cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index)

  tags = {
    Name = var.subnet_names[count.index]
  }
}
```

**Correct:**

```hcl
# Using for_each with a map -- stable keys, safe to add/remove
variable "subnets" {
  type = map(object({
    cidr_block = string
    az         = string
  }))
  description = "Map of subnet name to configuration."
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}
```

**Boolean toggle with count:**

```hcl
variable "create_bastion" {
  type        = bool
  description = "Whether to create the bastion host."
  default     = false
}

resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}
```
