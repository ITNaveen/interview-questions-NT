when we do terraform apply first time, then it create a terraform.tfstate files and this file is important for any further actions - 
1. mapping configuration to real world. 
2. tracking metadata - dependency, so create and delete resources in correct order.
3. as team it allows member to collaborate and work as a team.

.................
# remote state with s3 - 
we need S3 to store the file and DynamoDB for state locking and consistency check.
Once these 2 are created then we need - 
1. bucket name
2. key name
3. region
4. DynamoDB table name.

```yml
Create resource - 
MAIN.TF
resource "local_file" "pet" {
  filename = "/root/pets.txt"
  content  = "We love pets!"
}

Now we want to save state file to S3, DynamoDB - so terraform then backend "s3" then key value based bucketname, key, region, tablename.
TERRAFORM.TF
terraform {
  backend "s3" {
    bucket         = "kodekloud-terraform-state-bucket01"
    key            = "finance/terraform.tfstate"    # state file will be stored in folder called finance.
    region         = "us-west-1"
    dynamodb_table = "state-locking"    # this table should have primary key/hash key with the name lock_id
  }
}


then - 
terraform init -migrate-state - we will see option to copy state file from local to s3 as we already have state file in local.
Then remove local state file.
terraform apply will now lock state file in S3 and pull it down to memory.

.....................................
.....................................
# Terraform state command to manupulate state file - 
Terraform state file cant be edited like with vi or nano but should be used with proper command - 

terraform state show aws_s3_bucket.finance    # This show all the content of state file.
                list # list all state file.
                mv  # to resources to state file, we have source and destination.
                    # terraform state mv aws_dynamodb_table.state-locking aws_dynamodb_table.state-lockim. 
                pull # to display state file content on screen specially in terms of s3 storage state file.
                rm  # to delete items from state file.

# so resource can be seen - 
terraform state list 
to see for info on specific resource - 
terraform state show aws_instance.ec2-1

# check the status of state file wheather its locked or not - 
aws dynamodb scan --table-name my-lock-table
if it shows manual lock then unlock it or it will prevent any further operation on state file.
- aws dynamodb delete-item --table-name my-lock-table --key '{"LockID": {"S": "<lockID>"}}'
- terraform force-unlock <lockID>
```