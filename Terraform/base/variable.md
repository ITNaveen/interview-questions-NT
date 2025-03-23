```yml
# main.tf
resource "local_file" "pet" {
  filename = var.filename
  content  = var.content
}

resource "random_pet" "my-pet" {
  prefix    var.prefix
  separator = var.sperator
  length    = var.length
}

# variable.tf
variable "filename" {
  default = "/root/pets.txt"
}

variable "content" {
  default = "We love pets!"
}

variable "prefix" {
  default = "Mrs"
}

variable "separator" {
  default = "."
}

variable "length" {
  default = "1"
}
...............................................................................
# variable block description - 
1. 
    variable "resource-name" {
    default = default_value
    type = string/number/boolean
    description = "This is the local file"
    }

2. LIST 
    variable "prefix" {
    default = ["Mr", "Mrs", "They"]
    type = list
    }

    main.tf
    resource "random_pet" "my_pet" {
        prefix = var.prefix[2]
    }

3. MAP 
   variable "file_content" {
    type = map
    default = {
        statement1 = "tu jaa"
        statement2 = "tu bhag jaa"
    }
   }

   main.tf
   resource local_file my_pet {
    filename = /var/new/pet.txt
    content = var.file_content["statement2"]
   }

4. OBJECT 
   varible "bella" {
    type = object ({
        name = string
        color = string
        age = number
        food = list(string)
        favourote_pet = boolean
    })
    default = {
        name = "bella"
        color = "black"
        age = 6
        food = ["fish", "chicken", "turkey"]
        favourote_pet = false
    }
   }

   5. TUPLE 
      variable "kitty" {
        type = tuple([string, number, boolean])
        default = "nona", 8, true
      }

......................................................................
# input variable - 
1. if we leave var files like this - 
   variable "kitty" {

      }
   then terraform will ask for input when we do terraform apply or we can apply values directly in command line - 
   terraform apply -var='kitty=["nona", 8, true]'

2.  variable "file_content" {
    type = map
    default = {
        statement1 = "tu jaa"
        statement2 = "tu bhag jaa"
    }
    }
    terraform apply -var 'file_content={statement1 = "tu jaa", statement2 = "tu bhag jaa"}'

3.  variable "prefix" {
    default = ["Mr", "Mrs", "They"]
    type = list
    }
    terraform apply -var 'prefix=["Mr", "Mrs", "They"]'

.........................................................................

# using .tfvars - 
  this is normal var block - 
  OBJECT 
   varible "bella" {
    type = object ({
        name = string
        color = string
        age = number
        food = list(string)
        favourote_pet = boolean
    })
    default = {
        name = "bella"
        color = "black"
        age = 6
        food = ["fish", "chicken", "turkey"]
        favourote_pet = false
    }
    }
    var file - 
    now making it in var and .tfvars - 
    variable "bella" {
    type = object({
    name          = string
    color         = string
    age           = number
    food          = list(string)
    favourote_pet = boolean
    })
    }
    .tfvarfile - if this file name is auto.tfvars or terraform.tfvars then it will pick tfvars values by itself or we need to mention 
    terraform apply -var-file new.tfvars
    bella = {
    name = "bella"
    color = "black"
    age = 6
    food = ["fish", "chicken", "turkey"]
    favourote_pet = false
    }

..........................................................
# variable presidence - 
1. command line
   terraform apply -var="instance_type=t2.micro"

2. env var (TF_VAR_*)
   export TF_VAR_instance_type="t2.medium"
   terraform apply

2. terraform.tfvars

3. any.tfvars
   terraform apply -var-file="custom.tfvars"

3. variable.tf 
```