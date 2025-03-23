# HCL consist of block and arguments - 
block in curly braces - 

<block> <argument> {
    key1: value1,
    key2: value2
}

block contains information of infrastructure platform and set of resources within that platform we want to create.
ex- we can create a local dir then within that - LOCAL.TF and in here we will define resources as block

resource "local_file" "pet" {
  filename = "/root/pets.txt"
  content  = "I love pets"
}
resource type = local_file (provider_type-of-resource)
resource name = pet
argument - filename, content (these are based on the resource we are creating)

for EC2 - 
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"  # Replace with a valid AMI ID
  instance_type = "t2.micro"
}


