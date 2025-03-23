# Meta argument - 
we have so far created one resource but we want to create many resource of same type, so META_ARGUMENT is used. we have already used mata argument - depends_on and lifecycle.

# COUNT - resources are created as list.
using count meta argument, we can create n number of instances of given resource - 

```yml
resource "local_file" "pet" {               # pet is just a placeholder.
    filename = var.filename[count.index]
    count = length(var.filename)
}

variable "filename" {
    type = list(string)
    default = [
        "/root/pet.txt"
        "/root/dog.txt"
        "/root/man.txt
    ]
}
```
This way terraform will create files with provided name and also it means we can add on names and terraform will keep adding them.


# FOR_EACH - resources are created as MAP.
```yml
resource "local_sensitive_file" "name" {
    filename = each.value  # Set the filename for each file, taking values from the 'users' list
    for_each = toset(var.users) # Use 'for_each' with a set to iterate over unique user file paths  
    content = var.content # Set the content of each file (sensitive data like a password) 
}

variable "users" {
    type = list(string)    # Specify that this variable is a list of strings  
    default = [ "/root/user10", "/root/user11", "/root/user12", "/root/user13"] # Default list of file paths. 
}
variable "content" {
    default = "password: S3cr3tP@ssw0rd"
  
}

📌 for_each vs count – Side-by-Side Comparison
Feature	                            for_each (✅ Preferred)	                count (⚠️ Use with Caution)
Works with...	                      Lists, Sets, Maps ✅	                    Only Lists ❌
Tracks Individual Resources?	      ✅ Yes	                                  ❌ No (Uses index numbers)
Handles Changes Gracefully?	        ✅ Yes (Only modifies what’s needed)	    ❌ No (Index shift can cause recreation)
Adding/Removing Items	              ✅ Only affects the specific resource	  ❌ Can recreate all resources
Referencing Items	                  each.value or each.key	                 count.index
Example (File Creation)	            for_each = toset(var.files)	             count = length(var.files)
```
............................................................................

# version constraints - 
we can push terraform to install specific version of provider, to do this we select version and code block with that.
then we paste that block in main.tf on top - something similar based on version you need to paste in main.tf or versions.tf - 
```yml
terraform {
  required_version = "= 1.4.0"  # Ensure Terraform uses exactly version 1.4.0

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 1.3.5"  # Lock AWS provider to 1.3.5
    }
  }
}

or we can suggest the specific version which we dont want terraform to download - 
version = "!= 2.0.0"

or to make use of version lesser than specific one - 
version = "< 1.4.0"

or any version in this category - 
version = "~> 1.2"
greater than or equal to 1.2.
```