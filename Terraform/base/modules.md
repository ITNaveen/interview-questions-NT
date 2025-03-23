A company has a payroll app and this company has several international branch and each branch will deploy the same terraform code for app - 
# Infra includes - 
1. ec2 with custom ami to host app server.
2. dynamodb to store employee and payroll data (no sql).
3. s3 bucket to store pay stuff and tax form etc.

so this infra will be deployed in different branch across the world.
we create this as -  /root/terraform-project/modules/payroll-app

# This payroll-app will have  - 
1. app_server.tf - 
  resource "aws_instance" "app_server" {
  ami           = var.ami
  instance_type = "t2.medium"

  tags = {
    Name = "${var.app_region}-app-server"
  }

  depends_on = [
    aws_dynamodb_table.payroll_db,
    aws_s3_bucket.payroll_data
  ]
}

2. s3_bucket.tf - 
   resource "aws_s3_bucket" "payroll_data" {
   bucket = "${var.app_region}-${var.bucket}"
   }

3. dynamodb_table.tf - 
   resource "aws_dynamodb_table" "payroll_db" {
   name         = "user_data"
   billing_mode = "PAY_PER_REQUEST"
   hash_key     = "EmployeeID"

   attribute {
     name = "EmployeeID"
     type = "N"
   }
 }

4. variable.tf 
variable "ami" {
  description = "The AMI ID for the EC2 instance"
  type        = string
}

variable "app_region" {
  description = "The AWS region for the application"
  type        = string
}

variable "bucket" {
  description = "The name suffix for the S3 bucket"
  type        = string
}
...................................................

# Now we create a dir in /terraform-projects/us-payroll-app
main.tf
module "us_payroll" {
    source = "../modules/payroll-app"
    app_region = "us-east-1"
    ami = "ami-6879699svf"
}

lets do the same for UK - 
main.tf 
module "uk_payroll" {
    source = "../modules/payroll-app"
    app_region = "eu-west-1"
    ami = "ami-669969699svf"
}




....................................
....................................
....................................

