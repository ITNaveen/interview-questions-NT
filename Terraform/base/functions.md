# we already know these funtions - 
file - to read file content.
length - to count the length of filename etc.
toset - to convert list to set.

# ex - 
```yaml
variable "user_data" {
  default = file("/root/terraform-projects/user_data.sh")
}
resource "aws_instance" "development" {
    ami           = "ami-0edab43b6fa892279"
    instance_type = "t2.micro"
    user_data     = var.user_data
}
```
- file("/root/terraform-projects/main.tf"): Reads the contents of the specified Terraform configuration file.

# length(var.region)
3
length(var.region): Returns the number of elements in the var.region list, which is 3.

# toset(var.region)
[
    "ca-central-1",
    "us-east-1",
]
toset(var.region): Converts var.region (which is presumably a list) into a set, removing duplicates and returning unique region values. The output shows two unique regions: "ca-central-1" and "us-east-1".

....................................................................
....................................................................
# commanly used functions - 

-------NUMERIC FUNCTION - 
1. to transform and manupulate numbers - min, max
2. to use with variable - max(var.num...)
3. ceil(10.1) = 11 (more than and closer to previous number)
4. floor(10.1) = 10 (lesser then and close to previous number)

#### STRING FUNCTION - 
Transform and manupulate string type data. 

```yml
var.tf
variable "ami" {
    type        = string
    default     = "ami-xyz,AMI-ABC,ami-efg"
    description = "A string containing ami ids"
}
```
terraform console

```yml
- split(",", "ami-xyz,AMI-ABC,ami-efg")
[ "ami-xyz", "AMI-ABC", "ami-efg" ]

- split(",", var.ami)
[ "ami-xyz", "AMI-ABC", "ami-efg" ]

- lower(var.ami)
ami-xyz,ami-abc,ami-efg

- upper(var.ami)
AMI-XYZ,AMI-ABC,AMI-EFG

- title(var.ami)
Ami-Xyz,AMI-ABC,Ami-Efg

- substr(var.ami, 0, 7)              # 0 = starting point, 7 = length
ami-xyz

- substr(var.ami, 8, 7)
AMI-ABC

- substr(var.ami, 16, 7)
ami-efg
```

### COLLECTION FUNCTION - 
for list, set, map -
```yml
var.tf
variable "ami" {
    type        = list(string)
    default     = ["ami-xyz,AMI-ABC,ami-efg"]
    description = "A string containing ami ids"
}

terraform console

- length(var.ami)
3

- index(var.ami, "AMI-ABC")  # index = position
1

- element(var.ami, 2) # element = Value at a Position
ami-efg

- contains(var.ami, "AMI-ABC")
true

- contains(var.ami, "AMI-XYZ")
false
```

####  MAP FUNCTIONS - 

```yml
var.tf
variable "ami" {
    type = map
    default = {
        "us-east-1"    = "ami-xyz",
        "ca-central-1" = "ami-efg",
        "ap-south-1"   = "ami-ABC"
    }
    description = "A map of AMI ID's for specific regions"
}

$ terraform console

- keys(var.ami)
[
    "ap-south-1",
    "ca-central-1",
    "us-east-1"
]

- values(var.ami)
[
    "ami-ABC",
    "ami-efg",
    "ami-xyz"
]

lookup(var.ami, "ca-central-1")   #this give you value.
ami-efg
```

