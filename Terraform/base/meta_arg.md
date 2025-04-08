# Understanding Terraform Meta Arguments: for_each and count
Meta Arguments: The Basics
Meta arguments are special parameters in Terraform that affect how resources are created. The most common ones that people get confused about are count and for_each, which both help you create multiple instances of a resource.

```yml
# 1. When to Use Count
count is simpler and best used when:
You need a specific number of identical resources
The resources are numbered sequentially
The resources differ only by a simple index

Count Example:
resource "aws_instance" "server" {
  count = 3
  ami   = "ami-123456"
  instance_type = "t2.micro"
  
  tags = {
    Name = "server-${count.index}"  # Creates server-0, server-1, server-2
  }
}

# When to Use For_Each
for_each is more powerful and better when:
Resources need unique, non-sequential names
Resources have multiple varying properties
You're working with map or set data
You need to create resources based on a collection

exa - 
resource "aws_instance" "server" {
  for_each = {
    web = {
      type = "t2.micro"
      zone = "us-west-1a"
    }
    app = {
      type = "t2.medium"
      zone = "us-west-1b"
    }
    db = {
      type = "t3.large"
      zone = "us-west-1c"
    }
  }
  
  ami           = "ami-123456"
  instance_type = each.value.type
  availability_zone = each.value.zone
  
  tags = {
    Name = "server-${each.key}"  # Creates server-web, server-app, server-db
  }
}

# for set - 
resource "aws_security_group_rule" "allow_ports" {
  for_each = toset(["22", "80", "443"])
  
  type              = "ingress"
  from_port         = each.value
  to_port           = each.value
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.example.id
  
  description = "Allow port ${each.value}"
}

# Key Differences for Your Interview

Indexing:
count uses numeric indexes: count.index (0, 1, 2...)
for_each uses named keys: each.key and each.value


Input Data:
count takes a numeric value
for_each takes a map or set


Resource Naming:
count creates resources with indexes: resource.name[0], resource.name[1]
for_each creates resources with keys: resource.name["key1"], resource.name["key2"]


Resource Updates:
count: Changing the order can destroy and recreate resources
for_each: More stable with changes as resources are identified by keys


Resource References:
With count: aws_instance.server[0]
With for_each: aws_instance.server["web"]

When to Use Which?
Use count for: Simple, identical resources when the number might change
Use for_each for: Complex resources with unique configurations or when working with maps/sets of data