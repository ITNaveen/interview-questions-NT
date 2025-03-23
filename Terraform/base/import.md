This way i can import current infra to terraform.
There may be resource created manually using console or ansible, so we want these resources to be controlled by terraform.

# Do terraform plan first.

1. terraform import aws_instance.webserver-2 i-0907080ub67v7t6897 
   terraform import <resource_type>.<resource.name> <attribute>
   1. This tells Terraform: "Start tracking this AWS instance in the state file."
   2. But it does NOT create or modify main.tf.

2. terraform show | grep webserver-2 -A 50
   1. This shows all attributes of the imported resource from the Terraform state.
   2. Example: If webserver-2 references a security group, it may appear as part of its attributes.

3. Manually define the resource in main.tf - 
```yml
resource "aws_instance" "webserver-2" {
  ami           = "ami-0edab43b6fa892279"
  instance_type = "t2.micro"
  subnet_id     = "subnet-abc12345"
  key_name      = "my-key"
  tags = {
    Name = "webserver-2"
  }
}
```

4. terraform plan 
   1. If main.tf matches the actual resource state, Terraform should show "No changes".
   2. If Terraform suggests changes, adjust main.tf to match the current resource settings.

5. terraform apply -refresh-only
   It's like doing a "health check" on just your Terraform-managed resources. It completely ignores any resources that were created manually outside of Terraform, even if they're in the same environment.


