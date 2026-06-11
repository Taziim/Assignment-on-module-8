# Assingment 8
# Infrastructure Workflow
--Created access key from aws using access key added to tf vars
--created vpc
--

# Screenshots

## Terraform Apply
<img width="100%" alt="terraform-apply" src="./screenshots/terraformInit.png">
<img width="100%" alt="terraform-apply" src="./screenshots/terraformPlan.png">
<img width="100%" alt="terraform-apply" src="./screenshots/terraformApply.png">

## AWS VPC
![VPC](./screenshots/vpc.png)

---

## Public and Private Subnets
![Subnets](./screenshots/subnets.png)

---

## EC2 Instance
![EC2 Instances](./screenshots/ec2.png)

---

## Security Groups
![Security Groups](./screenshots/securitygroups.png)

---

## Route Tables
![Route Tables](./screenshots/routetables.png)

---

## Internet Gateway
![Internet Gateway](./screenshots/internetgateway.png)

---

## NAT Gateway
![NAT Gateway](./screenshots/natgateway.png)

---

## Elastic IP
![Elastic IP](./screenshots/elasticIp.png)

## GitHub Actions Workflow

#Install and Build

<img width="100%" alt="terraform-apply" src="./screenshots/image.png">

---

##Unabel to depoy to ec2

---

## Terraform Code
```main.tf
# -------------------------
# VPC
# -------------------------
resource "aws_vpc" "main_vpc" {
  cidr_block           = "12.0.0.0/16"
  enable_dns_hostnames = true
  tags = {
    Name = "main_vpc"
  }
}

# -------------------------
# Public Subnet
# -------------------------
resource "aws_subnet" "public_subnet_1" {
  vpc_id                  = aws_vpc.main_vpc.id
  cidr_block              = "12.0.1.0/24"
  availability_zone       = "ap-northeast-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "main_vpc_Public-Subnet_1"
  }
}

# -------------------------
# Private Subnet
# -------------------------
resource "aws_subnet" "private_subnet_1" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "12.0.2.0/24"
  availability_zone = "ap-northeast-1a"
  tags = {
    Name = "main_vpc_Private-Subnet_1"
  }
}

# -------------------------
# Internet Gateway
# -------------------------
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main_vpc.id
  tags = {
    Name = "main_vpc-IGW"
  }
}

# -------------------------
# Elastic IP
# -------------------------
resource "aws_eip" "nat_eip" {
  #domain = "vpc"
}

# -------------------------
# NAT Gateway
# -------------------------
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet_1.id
  depends_on    = [aws_internet_gateway.igw]
  tags = {
    Name = "main_vpc_natgateway"
  }
}

# -------------------------
# Route Tables
# -------------------------
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "main_Vpc_Public-RouteTable"
  }
}

resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.main_vpc.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
  tags = {
    Name = "main_vpc_Private-RouteTable"
  }
}

# -------------------------
# Route Table Associations
# -------------------------
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet_1.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private_subnet_1.id
  route_table_id = aws_route_table.private_rt.id
}

# -------------------------
# Security Group - Bastion (Public)
# -------------------------
resource "aws_security_group" "bastion_sg" {
  name        = "bastion_sg"
  description = "Allow SSH from trusted IP"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description = "SSH Access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Restrict to your IP in production, e.g. ["YOUR.IP.ADDRESS/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "public_bastion_sg-SG"
  }
}

# -------------------------
# Security Group - Private Instance
# -------------------------
resource "aws_security_group" "private_sg" {
  name        = "private_sg"
  description = "Allow SSH from Bastion only"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description     = "SSH from Bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "private_sg-SG"
  }
}

# -------------------------
# EC2 Instances
# -------------------------
resource "aws_instance" "main_vpc_bastion" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.public_subnet_1.id
  vpc_security_group_ids      = [aws_security_group.bastion_sg.id]
  associate_public_ip_address = true
  key_name                    = var.key_pair_name
  tags = {
    Name = "main_vpc_bastion_public"
  }
}

resource "aws_instance" "main_vpc_private" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.private_subnet_1.id
  vpc_security_group_ids = [aws_security_group.private_sg.id]
  key_name               = var.key_pair_name
  tags = {
    Name = "main_vpc_private"
  }
}

# -------------------------
# Outputs
# -------------------------
output "main_vpc_bastion_public_ip" {
  value = aws_instance.main_vpc_bastion.public_ip
}

output "private_ip" {
  value = aws_instance.main_vpc_private.private_ip
}
---

```provide.tf
provider "aws" {
  region     = var.aws_region
  access_key = var.my_access_key
  secret_key = var.my_secret_key
}
```terraform.tfvars
cannot add cause of github sensitive data violation

```variables.tf
variable "aws_region" {}
variable "my_access_key" {}
variable "my_secret_key" {}
variable "key_pair_name" {}
variable "ami_id" {}
variable "instance_type" {}

```version.tf
terraform {
  required_version = ">= 1.15.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~>5.0"
    }
  }
}

---

