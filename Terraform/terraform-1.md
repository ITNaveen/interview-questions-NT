# 50 Terraform Interview Questions & Answers (Intermediate to Expert Level)

## Infrastructure as Code Fundamentals

### 1. What is the difference between Terraform state locking and state versioning?

**Answer:**
State locking prevents concurrent operations that might corrupt the state file. When a Terraform operation begins that could write to the state, it locks the state file to prevent other processes from acquiring the lock and potentially corrupting the state.

State versioning maintains a history of state file changes. This allows for recovering previous states if a mistake is made.

**Example:**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "path/to/my/key"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks" # For state locking
  }
}
```

In this example, the DynamoDB table provides locking functionality, while S3 versioning (enabled on the bucket) provides state versioning.

### 2. How does Terraform handle dependencies between resources, and what happens if you remove a dependency?

**Answer:**
Terraform builds a dependency graph to determine the order of resource creation, modification, and deletion. Dependencies can be:

1. **Implicit** - When one resource references attributes of another using interpolation.
2. **Explicit** - When using the `depends_on` argument.

When a dependency is removed:
- If resources still reference the removed dependency, Terraform will produce an error.
- If no more references exist, Terraform will destroy the resource during the next apply unless it's protected with `prevent_destroy = true`.

**Example:**
```hcl
# Explicit dependency
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  depends_on    = [aws_route_table_association.public]
}

# Implicit dependency
resource "aws_eip" "ip" {
  instance = aws_instance.app.id
}
```

### 3. Explain the concept of idempotence in Terraform and why it's important.

**Answer:**
Idempotence means that applying the same configuration multiple times will result in the same end state, regardless of the starting state. This is crucial for infrastructure as code because:

1. It allows for safe, repeatable deployments
2. It enables drift detection and remediation
3. It ensures consistency across environments
4. It makes configurations testable and predictable

Terraform achieves idempotence by:
- Tracking the current state and comparing it to the desired state
- Only making changes necessary to reach the desired state
- Handling dependencies correctly to ensure proper order of operations

**Example:**
Running `terraform apply` twice on the same configuration should only make changes the first time (assuming no external changes).

### 4. What is the difference between `terraform apply` and `terraform apply -auto-approve`?

**Answer:**
- `terraform apply`: Shows the execution plan and requires manual confirmation before making any changes.
- `terraform apply -auto-approve`: Skips the confirmation step and immediately applies the changes.

The difference is important from a safety perspective:
- Without `-auto-approve`, you have a chance to review changes before execution
- With `-auto-approve`, changes apply immediately (useful for CI/CD pipelines but riskier)

**Recommendation:** Never use `-auto-approve` for production environments without thorough testing in a pre-production environment.

### 5. How would you handle sensitive data in Terraform?

**Answer:**
Terraform offers several methods to handle sensitive data:

1. **Terraform Variables with the sensitive flag**:
   ```hcl
   variable "database_password" {
     type      = string
     sensitive = true
   }
   ```

2. **Environment Variables**:
   ```
   TF_VAR_database_password="secret123"
   ```

3. **External Secret Management**:
   - AWS Secrets Manager integration
   - HashiCorp Vault integration
   - Azure Key Vault integration

4. **Encrypted State Files**:
   - Using backend encryption (e.g., S3 with SSE)
   - Using remote backends with encryption in transit (HTTPS)

**Best Practices:**
- Never commit secrets to version control
- Leverage centralized secret management
- Mark variables as sensitive to prevent values appearing in logs
- Encrypt state files at rest and in transit
- Use IAM roles for authentication where possible instead of static credentials

## Terraform State Management

### 6. When would you use state locking, and what happens if you don't?

**Answer:**
You should use state locking whenever multiple users or processes might attempt to modify the same infrastructure concurrently. This is particularly important in team environments or CI/CD pipelines.

Without state locking:
- Multiple simultaneous operations could corrupt the state file
- Race conditions could lead to unexpected outcomes
- Resources might be created when they should be updated, or vice versa
- You risk inconsistency between your state and actual infrastructure

**Example Configuration (AWS S3 + DynamoDB):**
```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "path/to/my/state"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### 7. Explain the purpose of a remote backend and compare at least three options.

**Answer:**
Remote backends store Terraform state files remotely rather than on local disk. They provide:
- Collaboration support for teams
- State locking to prevent corruption
- Secure storage for potentially sensitive data
- A central source of truth for infrastructure state

**Comparison of three common backends:**

1. **S3 Backend (AWS)**:
   - Pros: Inexpensive, versioning, encryption, highly durable
   - Cons: AWS-specific, requires separate DynamoDB table for locking
   - Best for: AWS-focused teams

2. **Terraform Cloud/Enterprise**:
   - Pros: Managed service, CI/CD integration, granular permissions, centralized management
   - Cons: Cost (Enterprise), additional system to manage
   - Best for: Teams needing governance features or a SaaS solution

3. **Azure Storage**:
   - Pros: Blob versioning, encryption, Azure-native integration
   - Cons: Azure-specific, more complex setup than S3
   - Best for: Azure-focused teams

**Example (Terraform Cloud):**
```hcl
terraform {
  backend "remote" {
    organization = "my-company"
    workspaces {
      name = "my-app-prod"
    }
  }
}
```

### 8. What steps would you take to migrate from a local backend to a remote backend?

**Answer:**
1. **Create the remote backend infrastructure** (e.g., S3 bucket, Azure storage account)
2. **Configure the backend in your Terraform code**:
   ```hcl
   terraform {
     backend "s3" {
       bucket = "my-terraform-state"
       key    = "path/to/my/key"
       region = "us-west-2"
     }
   }
   ```
3. **Initialize with migration flag**: Run `terraform init -migrate-state` which will:
   - Prompt for confirmation
   - Copy the local state to the remote backend
   - Remove the local state file
4. **Verify migration success**: 
   - Check the remote backend contains the state
   - Run `terraform plan` to confirm no changes are detected
5. **Update documentation and team processes**
6. **Commit backend configuration to version control**

**Best Practice:** Test this process in a non-production environment first.

### 9. How would you handle a corrupted Terraform state file?

**Answer:**
1. **Stop all operations**: Ensure no other Terraform operations are running
2. **Check for backups**:
   - If using a remote backend with versioning (like S3), restore a previous version
   - Check for local backups (e.g., `terraform.tfstate.backup`)
3. **Use state recovery commands**:
   - `terraform state pull > terraform.tfstate` to retrieve the current state
   - Edit the state file if needed (as a last resort)
   - `terraform state push terraform.tfstate` to upload the fixed state
4. **Perform a refresh**: `terraform refresh` to reconcile the state with reality
5. **Run a plan**: Verify the state appears correct with `terraform plan`
6. **Implement preventive measures**:
   - Use remote backends with versioning
   - Establish proper locking mechanisms
   - Set up team processes for state management

**Emergency Recovery (last resort)**:
If no backups exist, you might need to:
```
# Create an empty state file
echo "{\"version\": 4, \"terraform_version\": \"1.3.0\", \"serial\": 0, \"lineage\": \"\", \"outputs\": {}, \"resources\": [] }" > terraform.tfstate

# Import existing resources
terraform import aws_instance.example i-1234567890abcdef0
```

### 10. Explain how to use workspaces in Terraform and when they should be used.

**Answer:**
Terraform workspaces allow you to manage multiple distinct states using the same configuration files. They're managed with the `terraform workspace` commands.

**Key Commands:**
```bash
terraform workspace new dev      # Create new workspace
terraform workspace select prod  # Switch to workspace
terraform workspace list         # List workspaces
terraform workspace show         # Show current workspace
```

**Example Usage in Configuration:**
```hcl
resource "aws_instance" "example" {
  instance_type = terraform.workspace == "prod" ? "m5.large" : "t3.micro"
  count         = terraform.workspace == "prod" ? 10 : 1
  
  tags = {
    Environment = terraform.workspace
  }
}
```

**When to Use Workspaces:**
- ✅ For minor environment variations (dev/test/stage)
- ✅ For ephemeral developer environments
- ✅ For feature branch testing environments

**When NOT to Use Workspaces:**
- ❌ For significantly different configurations
- ❌ For production vs. non-production (better to use separate state files)
- ❌ When different teams manage different environments

**Best Practice:** For production environments, use separate configurations with different backend state files rather than workspaces.

## Modules and Structure

### 11. What are the best practices for organizing large Terraform projects?

**Answer:**
For large-scale Terraform projects:

1. **Repository Structure**:
   - Use a monorepo or multiple repos based on team structure
   - Consider using Terragrunt for repo organization

2. **Module Organization**:
   - Create reusable modules for common patterns
   - Separate modules by resource type or functional area
   - Maintain separate modules for each major cloud provider

3. **Environment Segregation**:
   - Use separate state files for each environment
   - Structure as `environments/[env]/[component]`

4. **Component Separation**:
   - Split large infrastructures into logical components
   - Example: networking, data storage, compute, security, etc.

**Example Directory Structure:**
```
terraform-infrastructure/
├── modules/                    # Reusable modules
│   ├── networking/
│   ├── database/
│   └── kubernetes/
├── environments/               # Environment-specific configurations
│   ├── dev/
│   │   ├── vpc/
│   │   ├── databases/
│   │   └── services/
│   ├── staging/
│   │   ├── vpc/
│   │   ├── databases/
│   │   └── services/
│   └── prod/
│       ├── vpc/
│       ├── databases/
│       └── services/
└── global/                     # Global/shared resources
    ├── iam/
    ├── dns/
    └── monitoring/
```

### 12. How do you effectively use the Terraform Registry and what differentiates a good module?

**Answer:**
The Terraform Registry is a repository of modules shared by the community and HashiCorp, making it easier to reuse well-architected infrastructure code.

**Effective Usage:**
1. Search for modules that match your requirements
2. Evaluate module quality (stars, usage, age, docs)
3. Pin to specific versions using semantic versioning
4. Contribute improvements back to the community

**Example Usage:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"

  name            = "my-vpc"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

**Characteristics of Good Modules:**
1. **Clearly defined purpose**
2. **Well-documented** - README with examples, inputs, outputs
3. **Proper versioning** - semantic version tags
4. **Sensible defaults** but highly configurable
5. **Validation** of inputs and proper error handling
6. **Complete examples** showing various use cases
7. **Automated tests**
8. **Security considerations** built-in

### 13. How would you deal with a module that needs to be different between environments, beyond simple variable changes?

**Answer:**
There are several strategies to handle significant environment differences:

1. **Conditional Resource Creation**:
   ```hcl
   resource "aws_backup_vault" "example" {
     count = var.environment == "prod" ? 1 : 0
     name  = "backup-vault"
   }
   ```

2. **Provider Configurations by Environment**:
   ```hcl
   provider "aws" {
     region = var.region
     alias  = "primary"
   }

   provider "aws" {
     region = var.environment == "prod" ? "us-west-2" : "us-east-1"
     alias  = "secondary"
   }
   ```

3. **Module Composition** - Create different parent modules for different environments:
   ```hcl
   # For production
   module "database_prod" {
     source = "./modules/database-ha"
     # High-availability parameters
   }

   # For non-production
   module "database_dev" {
     source = "./modules/database-simple"
     # Simplified parameters
   }
   ```

4. **Feature Toggles with Locals**:
   ```hcl
   locals {
     enable_disaster_recovery = var.environment == "prod" ? true : false
     backup_retention         = var.environment == "prod" ? 30 : 7
     multi_az                 = var.environment == "prod" ? true : false
   }
   ```

5. **Terragrunt** for DRY configurations across environments

### 14. What is the difference between using a module from the Terraform Registry and a local module?

**Answer:**
**Registry Modules:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"
  
  # module inputs
}
```

**Local Modules:**
```hcl
module "vpc" {
  source = "../modules/vpc"
  
  # module inputs
}
```

**Key Differences:**

1. **Source Path**:
   - Registry: `[namespace]/[name]/[provider]`
   - Local: File path (relative or absolute)

2. **Versioning**:
   - Registry: Explicit versioning via the `version` attribute
   - Local: Often managed via Git (commits/branches/tags)

3. **Discoverability & Documentation**:
   - Registry: Searchable with standardized docs
   - Local: Documentation based on your own standards

4. **Maintenance Responsibility**:
   - Registry: Maintained by module author (may be HashiCorp or community)
   - Local: Maintained by your team

5. **Trust & Security**:
   - Registry: Must evaluate third-party code
   - Local: Code controlled by your organization

**When to use each:**
- Registry: For standard patterns with well-maintained modules
- Local: For organization-specific patterns or when you need full control

### 15. How do you handle versioning of your Terraform modules?

**Answer:**
Module versioning is crucial for stability and controlled upgrades. Here's a comprehensive approach:

1. **Use Semantic Versioning (SemVer)**:
   - MAJOR: Breaking changes
   - MINOR: New features, backward compatible
   - PATCH: Bug fixes, backward compatible

2. **Git Tags for Version Control**:
   ```bash
   git tag -a v1.2.3 -m "Release version 1.2.3"
   git push origin v1.2.3
   ```

3. **Version Constraints in Consumer Code**:
   ```hcl
   module "vpc" {
     source  = "terraform-aws-modules/vpc/aws"
     version = "~> 3.14.0"  # Allows 3.14.x but not 3.15.0+
   }
   ```

4. **Version Reference Types**:
   - Exact: `version = "3.14.0"`
   - Pessimistic: `version = "~> 3.14.0"` (3.14.x only)
   - Optimistic: `version = ">= 3.14.0"` (3.14.0 or greater)

5. **Document Breaking Changes**:
   - Keep a CHANGELOG.md file
   - Clearly document upgrade paths

6. **Testing Across Versions**:
   - Ensure backward compatibility
   - Test all supported Terraform versions

7. **Private Module Registry** (for enterprises):
   - Consider Terraform Cloud or Enterprise
   - Or use Azure DevOps Artifacts, GitLab, etc.

## Advanced Configuration Techniques

### 16. Explain the concept of count vs for_each, and when you would choose one over the other.

**Answer:**
Both `count` and `for_each` allow you to create multiple instances of a resource, but they have different strengths:

**Count**:
```hcl
resource "aws_instance" "server" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "server-${count.index}"
  }
}
```

**For_each with a map**:
```hcl
resource "aws_instance" "server" {
  for_each      = {
    web  = "t2.micro"
    api  = "t2.small"
    db   = "t2.medium"
  }
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value
  
  tags = {
    Name = "server-${each.key}"
  }
}
```

**For_each with a set**:
```hcl
resource "aws_instance" "server" {
  for_each      = toset(["web", "api", "db"])
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "server-${each.key}"
  }
}
```

**When to use Count**:
- When instances are identical except for a simple incrementing number
- When working with simple lists where order is important
- For simple scaling of homogeneous resources

**When to use For_each**:
- When resources have unique names/identifiers
- When each instance has different configurations
- When removing a middle item shouldn't affect other resources
- When order doesn't matter, but stability does

**Key advantage of for_each:** When you remove an item from the middle of a count list, all higher-indexed resources are recreated. With for_each, only the removed item is affected.

### 17. What are Terraform's meta-arguments and how would you use them?

**Answer:**
Meta-arguments are special arguments that affect the behavior of resources and modules rather than configuring the resource itself. The primary meta-arguments include:

1. **`count`**: Creates multiple instances based on a number.
   ```hcl
   resource "aws_instance" "server" {
     count = 3
     ami   = "ami-0c55b159cbfafe1f0"
   }
   ```

2. **`for_each`**: Creates multiple instances based on a map or set.
   ```hcl
   resource "aws_route53_record" "www" {
     for_each = {
       us-east-1 = "10.0.1.0"
       us-west-2 = "10.0.2.0"
     }
     zone_id = aws_route53_zone.primary.zone_id
     name    = "www.example.com"
     type    = "A"
     ttl     = 300
     records = [each.value]
   }
   ```

3. **`depends_on`**: Creates explicit dependencies.
   ```hcl
   resource "aws_instance" "example" {
     ami           = "ami-0c55b159cbfafe1f0"
     instance_type = "t2.micro"
     depends_on    = [aws_s3_bucket.example]
   }
   ```

4. **`lifecycle`**: Customizes lifecycle behavior.
   ```hcl
   resource "aws_instance" "example" {
     ami           = "ami-0c55b159cbfafe1f0"
     instance_type = "t2.micro"
     
     lifecycle {
       create_before_destroy = true
       prevent_destroy       = true
       ignore_changes        = [tags]
     }
   }
   ```

5. **`provider`**: Specifies which provider configuration to use.
   ```hcl
   resource "aws_instance" "example" {
     provider      = aws.west
     ami           = "ami-0c55b159cbfafe1f0"
     instance_type = "t2.micro"
   }
   ```

6. **`provisioner`**: Runs actions after resource creation.
   ```hcl
   resource "aws_instance" "example" {
     ami           = "ami-0c55b159cbfafe1f0"
     instance_type = "t2.micro"
     
     provisioner "local-exec" {
       command = "echo ${self.private_ip} > ip_address.txt"
     }
   }
   ```

### 18. Explain how the `terraform_remote_state` data source works and when it should be used.

**Answer:**
The `terraform_remote_state` data source retrieves the state from another Terraform configuration, allowing you to use outputs from one Terraform project as inputs to another.

**Example:**
```hcl
# In your networking project
output "vpc_id" {
  value = aws_vpc.main.id
}

# In your application project
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "terraform-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.subnet_ids[0]
}
```

**When to use:**
1. To share information between separate Terraform configurations
2. To maintain separation of concerns (e.g., networking vs. applications)
3. When different teams are responsible for different parts of infrastructure
4. When you need to break a large configuration into smaller, more manageable pieces

**Best practices:**
1. Only expose necessary outputs (treat it like an API)
2. Document outputs and their structures
3. Version outputs carefully to avoid breaking changes
4. Consider alternatives like Terraform modules for tightly coupled resources

**Limitations:**
1. Creates coupling between separate configurations
2. Can lead to a "wait and apply" workflow
3. Might need to manage access to multiple state files
4. Only provides read access to outputs, not to resource attributes

### 19. How would you implement custom validation for Terraform variables?

**Answer:**
Custom validation for variables was introduced in Terraform 0.13 and provides a way to ensure inputs meet specific requirements before execution.

**Basic Syntax:**
```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }
}
```

**Complex Validation Examples:**

1. **Regex Pattern Matching:**
```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"
  
  validation {
    condition     = can(regex("^(dev|staging|prod)$", var.environment))
    error_message = "Environment must be 'dev', 'staging', or 'prod'."
  }
}
```

2. **Number Range Validation:**
```hcl
variable "port_number" {
  type        = number
  description = "Application port number"
  
  validation {
    condition     = var.port_number > 0 && var.port_number < 65536
    error_message = "Port number must be between 1 and 65535."
  }
}
```

3. **CIDR Range Validation:**
```hcl
variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"
  
  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "VPC CIDR must be a valid CIDR block."
  }
  
  validation {
    condition     = can(regex("^10\\.", var.vpc_cidr))
    error_message = "VPC CIDR must be in the 10.x.x.x range."
  }
}
```

4. **List Length Validation:**
```hcl
variable "availability_zones" {
  type        = list(string)
  description = "List of AZs to deploy into"
  
  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones must be specified."
  }
}
```

### 20. Describe the purpose of `terraform.tfvars`, `terraform.tfvars.json`, and `*.auto.tfvars` files.

**Answer:**
These files provide values for your defined variables in a Terraform configuration:

**terraform.tfvars:**
A file containing variable values in HCL format, automatically loaded by Terraform.

```hcl
# terraform.tfvars
region      = "us-west-2"
instance_type = "t2.micro"
vpc_cidr    = "10.0.0.0/16"
environment = "dev"
```

**terraform.tfvars.json:**
A JSON alternative to terraform.tfvars, also automatically loaded.

```json
{
  "region": "us-west-2",
  "instance_type": "t2.micro",
  "vpc_cidr": "10.0.0.0/16",
  "environment": "dev"
}
```

**\*.auto.tfvars / \*.auto.tfvars.json:**
Any file with `.auto.tfvars` or `.auto.tfvars.json` extension is automatically loaded.

```hcl
# environment.auto.tfvars
environment = "dev"
project     = "example"
```

**Loading Priority (highest to lowest):**
1. Command line flags (`-var` and `-var-file`)
2. `*.auto.tfvars` / `*.auto.tfvars.json` (alphabetical order)
3. `terraform.tfvars.json`
4. `terraform.tfvars`
5. Environment variables (`TF_VAR_name`)
6. Default values in variable declarations

**Best Practices:**
1. Use `terraform.tfvars` for common variable values (but don't commit secrets)
2. Use `.auto.tfvars` files to separate concerns (e.g., `network.auto.tfvars`, `security.auto.tfvars`)
3. Use environment-specific files with `-var-file` (e.g., `terraform apply -var-file="prod.tfvars"`)
4. Consider `.gitignore` for sensitive `.tfvars` files
5. For CI/CD, generate `.tfvars` files during the pipeline

## Advanced Provider Configuration

### 21. How do you configure multiple provider instances in Terraform?

**Answer:**
Multiple provider configurations allow you to manage resources across different regions, accounts, or with different settings in the same Terraform configuration.

**Example with AWS provider across regions:**
```hcl
# Default provider configuration
provider "aws" {
  region = "us-east-1"
}

# Additional provider configuration for west region
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Using the default provider
resource "aws_instance" "east_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Using the aliased provider
resource "aws_instance" "west_instance" {
  provider      = aws.west
  ami           = "ami-0892d3c7ee96c37be"
  instance_type = "t2.micro"
}
```

**Example with multiple AWS accounts:**
```hcl
provider "aws" {
  alias   = "production"
  region  = "us-east-1"
  profile = "production"
}

provider "aws" {
  alias   = "development"
  region  = "us-east-1"
  profile = "development"
}
```

**Passing providers to modules:**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  providers = {
    aws = aws.west
  }
}
```

**Within module definition:**
```hcl
# modules/vpc/main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.0.0"
    }
  }
}
```

### 22. How do you implement cross-region and cross-account resource management in Terraform?

**Answer:**
Managing resources across multiple regions or accounts requires careful provider configuration and may involve assuming roles or using different authentication methods.

Cross-Region Management:
# Define providers for different regions
provider "aws" {
  region = "us-east-1"
  alias  = "east"
}

provider "aws" {
  region = "us-west-2"
  alias  = "west"
}

# Create resources in different regions
resource "aws_vpc" "east_vpc" {
  provider = aws.east
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "west_vpc" {
  provider = aws.west
  cidr_block = "10.1.0.0/16"
}

# Cross-region resource dependencies
resource "aws_ec2_transit_gateway_peering_attachment" "example" {
  provider                = aws.east
  peer_region             = "us-west-2"
  peer_transit_gateway_id = aws_ec2_transit_gateway.west.id
  transit_gateway_id      = aws_ec2_transit_gateway.east.id
}

Cross-Account Management:
# Provider for account A (source account)
provider "aws" {
  region = "us-east-1"
  alias  = "account_a"
  profile = "account_a"
}

# Provider for account B (assumes a role in destination account)
provider "aws" {
  region = "us-east-1"
  alias  = "account_b"
  
  assume_role {
    role_arn     = "arn:aws:iam::ACCOUNT_B_ID:role/TerraformCrossAccountRole"
    session_name = "terraform-cross-account"
  }
}

# Resource in account A
resource "aws_s3_bucket" "account_a_bucket" {
  provider = aws.account_a
  bucket   = "account-a-bucket"
}

# Resource in account B
resource "aws_s3_bucket" "account_b_bucket" {
  provider = aws.account_b
  bucket   = "account-b-bucket"
}

Cross-Account Resource Sharing (AWS RAM example):
# Share a subnet from account A with account B
resource "aws_ram_resource_share" "example" {
  provider = aws.account_a
  name     = "cross-account-subnet-share"
}

resource "aws_ram_resource_association" "example" {
  provider           = aws.account_a
  resource_arn       = aws_subnet.example.arn
  resource_share_arn = aws_ram_resource_share.example.arn
}

resource "aws_ram_principal_association" "example" {
  provider           = aws.account_a
  principal          = "ACCOUNT_B_ID"
  resource_share_arn = aws_ram_resource_share.example.arn
}

Best Practices:
Use a separate state file for each account/region combination
Minimize cross-region/cross-account dependencies
Document the required IAM permissions for cross-account access
Consider using AWS Organizations for managing multi-account setups
For complex multi-account scenarios, consider using tools like Terragrunt

