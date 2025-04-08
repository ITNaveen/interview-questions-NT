# 50 Terraform Interview Questions & Answers (Intermediate to Expert Level)

## Infrastructure as Code Fundamentals

### 1. What is the difference between Terraform state locking and state versioning?

Why Only DynamoDB for Terraform State Locking?

3️⃣ Fully Managed & Highly Available – No need to manage servers, patch software, or handle failover—DynamoDB is AWS-managed and replicated across AZs.

4️⃣ Automatic Lock Expiry (TTL Feature) – Prevents stale locks if Terraform process crashes, the DynamoDB lock will automatically expire after its TTL (Time-to-Live) period, unlike RDS or file-based locks that require manual cleanup.
If Terraform stops unexpectedly (crashes, disconnects, etc.), DynamoDB will wait 600 seconds (TTL) from the last recorded operation. After that, the lock expires automatically, and Terraform becomes available for other users.

🔹 If Terraform is running normally, TTL does nothing.
🔹 If Terraform crashes, TTL ensures the lock doesn't stay forever.

5️⃣ Cost-Effective & IAM-Based Security – Cheaper than RDS, scales automatically, and uses IAM policies for fine-grained access control, making it more secure and efficient

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
Terraform automatically creates a dependency graph of all resources in your configuration. This helps determine:
✔ Order of resource creation (which resources must be created first).
✔ Parallelization (which resources can be created at the same time).
✔ Order of resource destruction (which dependencies must be deleted first).

1. **Implicit** - When one resource references attributes of another using interpolation.
resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "my_subnet" {
  vpc_id = aws_vpc.my_vpc.id  # 🔥 Implicit dependency
  cidr_block = "10.0.1.0/24"
}

2. **Explicit** - When using the `depends_on` argument.
resource "aws_instance" "my_vm" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  depends_on    = [aws_security_group.my_sg]  # 🔥 Explicit dependency
}

resource "aws_security_group" "my_sg" {
  name = "my-security-group"
}
Terraform Creates SG First, Then EC2 – But Deletes EC2 First, Then SG ✅

### 3. Explain the concept of idempotence in Terraform and why it's important.

**Answer:**
Terraform compares the current infrastructure state (terraform.tfstate) with the desired configuration (.tf files) and only applies changes if necessary.

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

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # 🔥 Prevents exposure in logs
}

resource "aws_db_instance" "my_db" {
  engine         = "mysql"
  instance_class = "db.t2.micro"
  password       = var.db_password  # Terraform will not show this in CLI output
}

export TF_VAR_db_password="EnvPassword123"  (Terraform picks up TF_VAR_db_password and assigns it to var.db_password automatically)
terraform apply - 
export TF_VAR_db_password="thispass"  # Set the password
terraform plan   # Uses the exported variable
terraform apply  # Uses the exported variable

..............................
AWS Secrets Manager integration --
- save the password - 
  aws secretsmanager create-secret --name "my-db-password" --secret-string "SuperSecret123!"

- retreive - 
  data "aws_secretsmanager_secret" "db_secret" {    ///This finds the secret based on the name "my-db-password".
  name = "my-db-password" 
  sensitive = true    ////Prevents Terraform from displaying it
  }
  ✔ This only finds metadata about the secret (e.g., ID, ARN).
  ✔ It does NOT return the actual secret value!

  data "aws_secretsmanager_secret_version" "db_secret_version" {  
  secret_id = data.aws_secretsmanager_secret.db_secret.id
  }
  use that - data.aws_secretsmanager_secret_version.db_secret_version.secret_string

  # so - 
  create password with aws secret manager
  then find with "aws_secretsmanager_secret"
  then pull "aws_secretsmanager_secret_version"
  now use it anywhere you want using data resource add secret_string at the end.
   
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
- from aws console in s3 check version then make that version current.
- terraform state list (this will show me what resources it has)
- terraform plan
- terraform apply

3. **Use state recovery commands**:
   - `terraform state pull > terraform.tfstate` to retrieve the current state
   -  Edit the state file if needed (as a last resort)
   - `terraform state push terraform.tfstate` to upload the fixed state
   - `terraform refresh` to reconcile the state with reality
   - `terraform plan`

### 10. Explain how to use workspaces in Terraform and when they should be used.

**Answer:**
Terraform workspaces allow you to manage multiple distinct states using the same configuration files. They're managed with the `terraform workspace` commands.

**Key Commands:**
```bash
terraform workspace new dev      # Create new workspace
terraform workspace list         # List workspaces
terraform workspace show         # Show current workspace
```
1. create profiles for each workspaces - 
aws configure --profile dev
aws configure --profile test
aws configure --profile prod
This will prompt you to enter AWS Access Key, Secret Key, and Region for each profile

2. terraform workspace new dev   # Create dev workspace
   terraform workspace new test  # Create test workspace
   terraform workspace list      # Check available workspaces

3. provider "aws" {
  region  = "us-west-1"
  profile = terraform.workspace == "dev" ? "dev" :
            terraform.workspace == "test" ? "test" :
            "prod"  # Default workspace uses the prod profile
  }

4. select terraform workspace and thats it.

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
When working with large Terraform projects, I break it down into three core concepts:
🔹 1. Multi-Repo Strategy
    What it means:
    Break the infrastructure into multiple Git repositories based on team, function, or environment.

    Why it's useful:
    Each team can work independently without stepping on each other’s toes. Smaller repos = easier code reviews, CI/CD, and version control.
    
    Example:
    terraform-networking/ – All VPCs, subnets, gateways
    terraform-security/ – IAM roles, policies
    terraform-app-infra/ – EC2, RDS, S3 for app stack

🔹 2. Modules for Reuse

    What it means:
    Write once, use everywhere. Create reusable Terraform modules for common patterns.

    Why it's useful:
    No need to repeat code in every project. Also helps enforce standards across environments.

    Example:
    A vpc module can be reused for dev, staging, and prod just by passing different CIDR blocks.
    ec2_instance module can be used by many teams with different AMIs or tags.

🔹 3. Terragrunt for DRY & Environment Management
    What it means:
    Terragrunt is a thin wrapper over Terraform that helps manage environments, state, and repeated code more cleanly.

    Why it's useful:
    You don’t want to copy-paste the same backend config or provider settings for every environment.

    Terragrunt lets you reuse modules but keep environment configs separate.
    How it fits in:
    If I have 3 environments (dev, stage, prod), instead of creating full copies of Terraform code for each, I create just one module and use Terragrunt to manage configs per environment.

### 15. How do you handle versioning of your Terraform modules?

**Answer:**
Module versioning is crucial for stability and controlled upgrades. Here's a comprehensive approach:

🌟 Scenario - You have:
Module Repo (terraform-modules.git) → Contains your EC2 module.
Project Repo → Uses the module to create an EC2 instance.

Now, you want to:
Version your module correctly using Git.
Check version history and see what changed.
Use a specific version in your Terraform project.

1️⃣ Versioning Your Module
Step 1: Tagging the First Version (v1.0.0)
You finished your first working version of the EC2 module, so you tag it:
cd /path/to/terraform-modules
- git tag v1.0.0
- git push origin v1.0.0  # Push the tag to GitHub

✅ Now, v1.0.0 is saved in GitHub as a versioned release.
Step 2: Making Changes (Adding SSH Key)
Now, you add SSH key support to the module.
Before pushing the changes, you create a new tag:
- git tag v1.0.1
- git push origin v1.0.1  # Push the new tag to GitHub

✅ Now, you have two versions (v1.0.0 and v1.0.1) available in GitHub.
2️⃣ Listing Versions & Checking Changes
Step 3: List All Versions
To see all tagged versions:
git tag   //Lists all tags that exist locally.   ///If you tagged a version but haven't pushed it yet, it will only appear here.
This will show:
v1.0.0
v1.0.1

If you want to see tags in GitHub, run:
git ls-remote --tags origin  ///Shows all tags that exist in the remote repo.  ///If a tag appears locally but not in this list, you likely forgot to push it.

Step 4: Check What Changed in Each Version
To see what changed between v1.0.0 and v1.0.1:
git diff v1.0.0 v1.0.1
It will show something like:
+ variable "ssh_key" {
+   type = string
+ }
✅ This tells you that v1.0.1 introduced the SSH key feature.

If you just want to see commit messages per version, run:
git log v1.0.0..v1.0.1 --oneline
This shows all commits between v1.0.0 and v1.0.1.

3️⃣ Using a Specific Module Version in Terraform
Now, in your Terraform project repo, you can use a specific module version.
Use the first version (v1.0.0):
module "ec2" {
  source = "git::https://github.com/my-org/terraform-modules.git//ec2?ref=v1.0.0"
}

Switch to the new version (v1.0.1):

module "ec2" {
  source = "git::https://github.com/my-org/terraform-modules.git//ec2?ref=v1.0.1"
}

Update Terraform to fetch the new module version:
terraform get -update
terraform init -upgrade

✅ Now, your Terraform project is using v1.0.1, which includes SSH key support.

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
✅ Good for:
✔ Same instance type, AMI, and configuration.
✔ Simple, predictable structure.
✔ Order matters (like server-0, server-1, server-2).

❌ Problem with Deletion:
If you remove server-0 by reducing count = 2, Terraform will recreate server-1 as server-0 and server-2 as server-1.

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
✅ Good for:
✔ Unique instance types (or other attributes).
✔ Resources won’t be recreated if one is removed.
✔ Useful for maps or sets of dynamic values.

✅ How Deletion Works (for_each)

If you remove "web" from the map, only server-web is deleted.
"server-api" and "server-db" remain untouched.
❌ Why Not Use for_each All the Time?

More complex than count – requires a map/set instead of just a number.
Extra management overhead – better for unique configurations, not simple duplication.
Terraform doesn’t support for_each on lists, so you need to convert lists to sets/maps (toset() or {}), adding complexity.

📌 Example: Using for_each with toset
resource "aws_instance" "server" {
  for_each = toset(["app1", "app2", "app3"])  # Convert list to a set
  ami      = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${each.key}"  # Creates "server-app1", "server-app2", "server-app3"
  }
}


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
Meta-arguments in Terraform don’t define resources but instead change their behavior—like how many to create, conditions for creation, or dependencies.

Terraform Meta-Arguments Guide

1️⃣ count: Create Multiple Identical Instances
Creates a specified number of identical resources.✅ Best for: When you need identical resources but with a unique index.
resource "aws_instance" "server" {
  count         = 3  # Creates 3 instances
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${count.index}"  # "server-0", "server-1", "server-2"
  }
}
📌 Issue with Deletion: If you remove count = 2, Terraform may recreate instances with shifted indexes.

2️⃣ for_each: Create Multiple Unique Resources
Creates multiple resources dynamically using a map or set.✅ Best for: When each resource has unique attributes.
🟢 Using a Map (Key-Value Pairs)
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
✅ Creates:
www-us-east-1 → IP 10.0.1.0
www-us-west-2 → IP 10.0.2.0

🟢 Using a Set (List of Unique Names)
resource "aws_instance" "server" {
  for_each = toset(["app1", "app2", "app3"])  # Converts list to a set
  ami      = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "server-${each.key}"  # "server-app1", "server-app2", "server-app3"
  }
}
📌 Advantage: If you remove "app1", only that instance is deleted, without affecting others.

3️⃣ depends_on: Define Dependencies
Ensures a resource is created only after another resource exists.✅ Best for: When Terraform doesn’t detect dependencies automatically.
resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name"
}
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  depends_on    = [aws_s3_bucket.example]  # Ensures S3 bucket is created first
}
📌 Without depends_on, Terraform might try to create the EC2 before the S3 bucket exists.

4️⃣ lifecycle: Manage Resource Behavior
Controls how Terraform creates, updates, and destroys resources.✅ Best for: Preventing unwanted changes, ensuring smooth updates.
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true  # Avoids downtime by creating a new instance first
    prevent_destroy       = true  # Prevents accidental deletion
    ignore_changes        = [tags]  # Ignore tag changes (e.g., manually updated tags)
  }
}
📌 Behavior:
If Terraform tries to delete this instance, it fails due to prevent_destroy.
Tag changes won’t trigger updates.

5️⃣ provider: Use a Specific Provider Configuration

provider "aws" {
  region  = "us-east-1"
  profile = "default"
}

provider "aws" {
  alias   = "dev"
  region  = "us-west-2"
  profile = "dev"
}

provider "aws" {
  alias   = "test"
  region  = "eu-central-1"
  profile = "test"
}

resource "aws_s3_bucket" "prod_bucket" {
  provider = aws  # Uses default provider (us-east-1)
  bucket   = "my-prod-bucket"
}

resource "aws_s3_bucket" "dev_bucket" {
  provider = aws.dev  # Uses "dev" provider (us-west-2)
  bucket   = "my-dev-bucket"
}

resource "aws_s3_bucket" "test_bucket" {
  provider = aws.test  # Uses "test" provider (eu-central-1)
  bucket   = "my-test-bucket"
}


### 18. Explain how the `terraform_remote_state` data source works and when it should be used.
**Answer:**
The `terraform_remote_state` data source retrieves the state from another Terraform configuration, allowing you to use outputs from one Terraform project as inputs to another.

Example Scenario
Networking Team creates VPC, Subnets, Security Groups, and stores Terraform state remotely (e.g., in an S3 bucket or Terraform Cloud).
App Team needs the App Subnet ID to deploy EC2 instances but shouldn’t manage the networking itself.
Solution: The Networking Team outputs the app_subnet_id, and the App Team fetches it using terraform_remote_state.

Networking Team - Creating VPC with Two Subnets
terraform {
  backend "s3" {  
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"  
    region = "us-east-1"
  }
}
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
resource "aws_subnet" "app_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
resource "aws_subnet" "db_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
}

# Outputs for other teams
output "vpc_id" {
  value = aws_vpc.main.id
}

output "app_subnet_id" {
  value = aws_subnet.app_subnet.id
}

output "db_subnet_id" {
  value = aws_subnet.db_subnet.id
}
📌 Networking state is stored in: s3://my-terraform-state/networking/terraform.tfstate

- App Team - Fetching the App Subnet Using terraform_remote_state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = data.terraform_remote_state.networking.outputs.app_subnet_id  # Fetching App Subnet

  tags = {
    Name = "App-Instance"
  }
}
📌 The App Team now dynamically fetches the app_subnet_id without needing to modify networking infrastructure.
**When to use:**
1. To share information between separate Terraform configurations
2. To maintain separation of concerns (e.g., networking vs. applications)
3. When different teams are responsible for different parts of infrastructure
4. When you need to break a large configuration into smaller, more manageable pieces

### 20. Describe the purpose of `terraform.tfvars`, `terraform.tfvars.json`, and `*.auto.tfvars` files.
**Answer:**
These files provide values for your defined variables in a Terraform configuration:

Scenario - We want to deploy an EC2 instance in a VPC with a dynamic subnet and instance type, where the configuration differs for prod and test environments.

- main.tf (Defines EC2 Instance, VPC, and Subnet)
provider "aws" {
  region = "us-east-1"
}
resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = {
    Name = "${var.environment}-app"
  }
}

- variables.tf (Defines Variables Without Values)
variable "instance_type" {
  type = string
}
variable "vpc_id" {
  type = string
}
variable "subnet_id" {
  type = string
}
variable "environment" {
  type = string
}

# Environment-Specific Variable Files
- prod.tfvars (For Production Environment)
instance_type = "t3.large"
vpc_id        = "vpc-12345678"
subnet_id     = "subnet-abcdef12"
environment   = "prod"

- test.tfvars (For Testing Environment)
instance_type = "t2.micro"
vpc_id        = "vpc-87654321"
subnet_id     = "subnet-fedcba21"
environment   = "test"

{
  "resources": [
    {
      "type": "aws_instance",
      "name": "app",
      "instances": [
        {
          "attributes": {
            "ami": "ami-0c55b159cbfafe1f0",
            "instance_type": "t2.micro",
            "subnet_id": "subnet-fedcba21",
            "tags": {
              "Name": "test-app"
            }
          }
        }
      ]
    }
  ]
}
# How auto.tfvars Works
Any file ending in *.auto.tfvars is automatically loaded when running terraform apply.
No need to specify -var-file manually.
If -var-file is used, Terraform ignores *.auto.tfvars and only uses the explicitly mentioned variable file.
Example Without -var-file (auto.tfvars is used)
 
terraform apply --var-file=test.tfvars  (to use test file rather)

## Advanced Provider Configuration

### 21. How do you configure multiple provider instances in Terraform?

**Answer:**
Multiple provider configurations allow you to manage resources across different regions, accounts, or with different settings in the same Terraform configuration.

# Default provider (us-east-1)
provider "aws" {
  region = "us-east-1"
}

# Additional provider configuration (us-west-2)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# Instance using the default provider (us-east-1)
resource "aws_instance" "east_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Instance using the aliased provider (us-west-2)
resource "aws_instance" "west_instance" {
  provider      = aws.west  # Specifies the "west" alias
  ami           = "ami-0892d3c7ee96c37be"
  instance_type = "t2.micro"
}


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


--------------------
---------------------

## Terraform State Operations  ------

### 24. How would you modify a resource that has drifted from the Terraform configuration?

- if someone manually creates a resource, Terraform won’t track it. I detect this by running terraform plan. To bring it under Terraform’s control, I use terraform import, then check its configuration using terraform show. Finally, I manually define it in main.tf, verify with terraform plan, and apply the changes to fully manage it in Terraform.

📌 How to Fix a Resource That Drifted from Terraform
Scenario:
You originally created 10 EC2 instances and 2 S3 buckets using Terraform.
A developer manually created 1 extra EC2 instance and 1 extra S3 bucket outside Terraform.
You need to detect, import, and sync these changes into Terraform.

🔹 Step 1: Detect Drift in Terraform
Run the following command:
terraform plan
What happens?
Terraform checks if the infrastructure matches the configuration.

It detects the extra resources not managed by Terraform and shows a warning like this:
Note: 1 additional EC2 instance found in AWS that is not managed by Terraform:
- Instance ID: i-0abcdef1234567890
Note: 1 additional S3 bucket found in AWS that is not managed by Terraform:
- Bucket Name: my-manual-bucket
✅ This confirms that AWS has extra resources that Terraform doesn’t know about.

🔹 Step 2: Import the Manually Created Resources
Since these resources already exist in AWS, we need to import them into Terraform’s state file.

Import the EC2 Instance
terraform import aws_instance.manual_instance i-0abcdef1234567890    (manual_instance, this is of my choice)
What happens?
Terraform now tracks this EC2 instance in its state file.
But it is NOT yet added to your main.tf file.

Import the S3 Bucket
terraform import aws_s3_bucket.manual_bucket my-manual-bucket
What happens?
Terraform now tracks this S3 bucket in its state file.
But it is NOT yet in main.tf.

✅ Now, Terraform recognizes the manually created EC2 instance and S3 bucket, but we still need to update main.tf.

🔹 Step 3: Verify Imported Resources
To check if Terraform now tracks these resources, run:
terraform show
Expected Output (Snippet for EC2 and S3):
# aws_instance.manual_instance:
resource "aws_instance" "manual_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  id            = "i-0abcdef1234567890"
}

# aws_s3_bucket.manual_bucket:
resource "aws_s3_bucket" "manual_bucket" {
  bucket = "my-manual-bucket"
}
✅ Terraform now tracks the resources in its state file, but they are missing in main.tf.

🔹 Step 4: Update main.tf to Include Imported Resources
Now, we manually add the imported resources to main.tf to prevent Terraform from deleting them in the future.

Modify main.tf like this:
resource "aws_instance" "manual_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

resource "aws_s3_bucket" "manual_bucket" {
  bucket = "my-manual-bucket"
}
✅ Now, main.tf matches the actual infrastructure in AWS.

🔹 Step 5: Apply Changes to Sync Terraform with AWS
Run the following command:
terraform apply -auto-approve
Expected Output:
No changes. Your infrastructure matches the configuration.
✅ Terraform now fully manages the manually created EC2 instance and S3 bucket. Everything is in sync! 🚀

# recap of steps - 
1️⃣ Detect drift: terraform plan
🔹 Shows extra EC2 instance & S3 bucket that are unmanaged by Terraform.
2️⃣ Import EC2: terraform import aws_instance.my_instance i-0abcdef1234567890
🔹 Adds EC2 instance to Terraform state but not to main.tf.
3️⃣ Import S3 bucket: terraform import aws_s3_bucket.my_bucket my-manual-bucket
🔹 Adds S3 bucket to Terraform state but not to main.tf.
4️⃣ Verify imports: terraform show
🔹 Displays the full configuration of imported EC2 & S3 bucket.
5️⃣ Update main.tf with the imported resources using nano main.tf
🔹 Ensures Terraform configuration matches the actual infrastructure.
6️⃣ Check changes: terraform plan
🔹 Confirms that main.tf and state are now in sync.
7️⃣ Apply to sync: terraform apply -auto-approve ✅
🔹 Terraform now fully manages the imported EC2 & S3 bucket! 🚀

### 25. How would you rename a resource in Terraform without destroying and recreating it?
- terraform state mv aws_instance.old_name aws_instance.new_name
✅ This ensures Terraform keeps track of the resource under the new name without thinking it's deleted.

- Then, manually update main.tf to reflect the new name:
resource "aws_instance" "new_name" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
}

- After that, run terraform plan to confirm everything is in sync:
terraform plan
✅ If done correctly, Terraform should show no changes instead of trying to destroy/recreate anything.

- Finally, apply the changes:
terraform apply -auto-approve

### 26. Explain the purpose of the `terraform state` commands and provide examples.

**Answer:**
The `terraform state` command provides subcommands for advanced state management. These operations modify the state file directly and should be used with caution.

**Key Subcommands:**

1. **list** - Shows all resources in the state:
   ```bash 
   terraform state list   (only show the list of all resources and not attributes)
   terraform state list aws_instance.*  # Filter by resource type
   ```

2. **show** - Shows detailed information about a specific resource:
   ```bash
   terraform state show aws_instance.example
   ```

3. **mv** - Moves an item in state (rename or move between modules):
   ```bash
   # Rename a resource
   terraform state mv aws_instance.old aws_instance.new
   
   # Move a resource into a module
   terraform state mv aws_instance.example module.instances.aws_instance.example
   ```

4. **rm** - Removes items from state (without destroying resources):
   ```bash
   terraform state rm aws_instance.example
   ```


