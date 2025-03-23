# ...........Key Commands....................
Checking Tainted Resources -------
terraform show
terraform state list | grep "\(tainted\)"

Tainting a Resource - 
terraform taint resource_type.resource_name
terraform taint aws_instance.webserver

Removing a Taint --------
terraform untaint resource_type.resource_name
terraform untaint aws_instance.webserver

you need to check individually resources wheather they are tainted or not - terraform state show aws_my_ec2.
✅ Yes, if a resource is tainted, it means that Terraform will destroy and recreate it in the next terraform apply.

...........When to Use Tainting................
Resource appears misconfigured
Provisioners failed during initial setup
Debugging complex infrastructure states
Forcing complete resource recreation

............Important Considerations..............
Destroys and recreates the entire resource
Can cause temporary infrastructure downtime
Loses any state not defined in configuration
Dependent resources may be affected