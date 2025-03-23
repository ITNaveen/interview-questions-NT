# mutable and immutable update - 
Terraform generally follows an immutable infrastructure approach, but it's important to understand that it can support both patterns - 

Resources are never modified in-place.
When changes are needed, new resources are created and old ones are destroyed.
This ensures consistency and reproducibility.

# Reduces configuration drift.
📌 Causes of Drift:
✅ Manual changes in the cloud console (e.g., modifying an EC2 instance size).
✅ External automation tools (e.g., AWS Auto Scaling modifying an instance).
✅ Provider changes (e.g., a security group rule is altered by another tool).

🔹 How to Prevent Terraform Drift?
✅ Use terraform import to bring manually created resources into Terraform management.
✅ Enable prevent_destroy = true on critical resources to stop accidental deletions.
✅ Automate drift detection with scheduled terraform plan runs in CI/CD pipelines.
✅ Restrict manual changes by enforcing IAM permissions to prevent direct edits.

# how to make sure terraform is used for aws resource creation and not manual way - 
🔹 Best and Easiest Way: Use an IAM Policy to Block Manual Changes & Allow Only Terraform
The simplest and most effective way is to create an IAM policy that:
✅ Denies manual resource creation/modification via AWS Console, CLI, or SDK.
✅ Allows Terraform to create resources by requiring a special tag (ManagedBy=Terraform).

📌 Steps to Implement
🔹 Step 1: Create an IAM Policy
This policy denies resource creation unless the request includes the tag "ManagedBy": "Terraform".
```yml
{
    "Version": "2012-10-17",  // Specifies the IAM policy language version
    "Statement": [
        {
            "Effect": "Deny",  // Denies the specified actions if the condition is met
            "Action": "*",  // Denies all AWS actions
            "Resource": "*",  // Applies the denial to all AWS resources
            "Condition": {
                "StringNotEqualsIfExists": {  // Condition applies only if the key exists
                    "aws:RequestTag/ManagedBy": "Terraform"  // Denies actions if the "ManagedBy" tag is NOT "Terraform"
                }
            }
        }
    ]
}
```

🔹 Step 2: Attach the Policy to All IAM Users
Attach this policy to IAM users, groups, or roles who should NOT create resources manually.
Terraform must always add the tag ManagedBy = Terraform when creating resources.

🔹 Step 3: Ensure Terraform Applies the Required Tag
Modify your Terraform code to always include the tag.
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    ManagedBy = "Terraform"  # This allows Terraform to create the resource
  }
}
🚀 How This Works
✅ If someone manually creates a resource (AWS Console, CLI, SDK) ❌ → DENIED
✅ If Terraform creates a resource (with the required tag) ✅ → ALLOWED

......................................................................
# Lifycycle rule - 
normally terraform destroy old infra and then create new one but sometime we want it to create new then destroy old or not
destroy old one at all , i am talking about when we do update.

1. create new before destroying old - 
resource "local_file" "pet" {
  filename        = "/root/pets.txt"
  content         = "We love pets!"
  file_permission = "0700"

  lifecycle {
    create_before_destroy = true      #this way terraform destroy old one after creating new one
  }
}
2. to prevent destroying old one - 
resource "local_file" "pet" {
  filename        = "/root/pets.txt"
  content         = "We love pets!"
  file_permission = "0700"

  lifecycle {
    prevent_destroy = true      #this way terraform cant destroy old resource at all
  }
}

.......................................
# Ignore change - 
we can also intruct terraform not to touch some specifics in the resource with ignore change arguement.

resouces "aws_instance" "webserver" {
    ami= 697977023hihoo707
    instance_type= t2.micro
    tags= {
        Name = "ProjectA-Webserver"
    }
    lifecycle {
        ignore_changes = [
            tags               # This way i have stopped terraform to change tag name.
        ]
    }
}

......................
    lifecycle {
        ignore_changes = [
            tags, ami              # This way i have stopped terraform to change tag, ami.
        ]
    }

    lifecycle {
        ignore_changes = all       # no chnages for this resource                        
    }
........................