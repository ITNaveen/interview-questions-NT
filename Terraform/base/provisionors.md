# terraform provisioners are used to execute scripts or commands on a resource after it is created or destroyed. They help with bootstrapping, configuration, and automation of infrastructure.
```yml
- we have used remote-exec as we are making resources in aws from local terraform - 
resource "aws_instance" "webserver" {
  ami           = "ami-0edab43b6fa892279"
  instance_type = "t2.micro"
  
  key_name               = aws_key_pair.web.id
  vpc_security_group_ids = [aws_security_group.ssh-access.id]

  connection {
    type        = "ssh"
    user        = "ubuntu"  # For Amazon Linux, use "ec2-user"
    private_key = file("~/.ssh/id_rsa")  # Replace with your key path
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install nginx -y",
      "sudo systemctl enable nginx",
      "sudo systemctl start nginx"
    ]
  }
}

# using script
provisioner "remote-exec" {
  script = "scripts/install_nginx.sh"
}


.........................................................
- Local-exec can be used to fetch info from remote resource and save that in local.
Creating ec2 and then saving its IP in local dir - 
resource "aws_instance" "webserver" {
  ami           = "ami-0edab43b6fa892279"
  instance_type = "t2.micro"

  key_name               = aws_key_pair.web.key_name
  vpc_security_group_ids = [aws_security_group.ssh-access.id]

  # Run local-exec after EC2 is created
  provisioner "local-exec" {
    command = "echo '${self.id} - ${self.private_ip}' >> /root/local/resource_details.txt" # self is used inside the resource block where the provisioner is defined.
  }
}

........................................................................
........................................................................
# provisionar behaviour - 
1. creation time provisioner - means when resource is created then its used, so used during the creation of resources.

2. Destroy time provisioner - means its created when resource is destroyed.
resource "aws_instance" "webserver" {
  ami           = "ami-0edab43b6fa892279"
  instance_type = "t2.micro"

  # Log instance creation
  provisioner "local-exec" {
    on-failure = continue
    command = "echo Instance ${aws_instance.webserver.public_ip} Created on $(date) >> /tmp/instance_created.log"
  }

  # Log instance destruction
  provisioner "local-exec" {
    when = destroy
    command = "echo Instance ${aws_instance.webserver.public_ip} Destroyed on $(date) >> /tmp/instance_destroyed.log"
  }
}
this way it will create resource and save its ip in local in instance_created.log and this - instance_destroyed.log when deleted.
```
