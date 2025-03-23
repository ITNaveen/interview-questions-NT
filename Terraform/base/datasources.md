# Terraform data sources allow you to fetch and use information about resources created outside Terraform (or managed by another Terraform configuration).
data resource in Terraform is used to fetch and use resources created outside Terraform within your Terraform configuration.

How Data Sources Work
    They query existing infrastructure and retrieve details like IDs, names, or configurations.
    The fetched data can then be used in Terraform configurations.

ex - 
resource "aws_instance" "my_ec2" {
  ami             = "ami-12345678"
  instance_type   = "t2.micro"
  security_groups = [data.aws_security_group.existing_sg.name]
}

so i can use data as - data.aws_instance.my_ec2

# terraform console - 
The Terraform console acts like a sandbox environment where you can test Terraform expressions, commands, and logic before applying them to your infrastructure.

# so data type are used to read information from existing resource and then use it further.

To see which resources were created by terraform - 
terraform state list
all resources managed by Terraform, even if they were created 6 months ago or earlier

......................................................
......................................................




