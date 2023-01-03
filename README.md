# WordPress-installation-on-AWS-Ec2-using-Terraform

Today we are using terraform for scripting on AWS to create a VPC,EC2 and install wordpress using user data.
### Terraform benefits
- Orchestration, not merely configuration
- Immutable infrastructure
- Declarative, not procedural code
- Client-only architecture
- Portability
- Automation
- Reliability
- Clearly mapped resource dependencies 

### About this project
In this project I would like to demonstrate how to install WordPress application in AWS using Terraform. The frontend of the website is hosted on an independant Ec2 instance and the database is managed using another Ec2 instance which is created in private zone. Access to the Database server will be restricted as it is created in a private subnet. Public SSH access into both these instances are only be possible though the third one Bastion server. 

##### AWS resources used in this project are
- VPC
- Subnets
- Internet Gateway
- Nat Gateway 
- Route Tables
- EC2 instance
- EIP for NAT Gateway
- Security Groups
- Route53

##### Prerequisite
- IAM user with Programmatic access to AWS with AmazonEc2FullAccess and AmazonRoute53FullAccess
- It is desirable to have Terraform v1.3.6 and Terraform AWS provider version 4.48.0

### Step 1
What I do first is, create a project directory and create the following files for the installation
provider.tf, variables.tf , outputs.tf , datasource.tf and main.tf

### Step 2
Configure provider.tf file with the access key and secretkey for the IAM user
```sh
 provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
 default_tags {
    tags = local.common_tags
  }
}
```
### Step3
Creating the variables needed for this project in the variable.tf file

```sh
variable "project" {
  default = "zomato"
}
 
variable "environment" {
  default = "production"
}
 
variable "region" {
  default = "ap-south-1"
}
variable "access_key" {
 
  default     = "xxxxxxxxxxx"
  description = "project access_key"
}
 
variable "secret_key" {
 
  default     = "xxxxxxxxxxxxxxx"
  description = "project secret_key"
 
}
 
variable "instance_ami" {
  default = "ami-XXXXXXXXX"
}
 
 
variable "instance_type" {
  default = "t2.micro"
}
 
locals {
  common_tags = {
    "project"     = var.project
    "environment" = var.environment
  }
 
}
 
variable "vpc_cidr" {
  default = "172.16.0.0/16"
}
locals {
  subnets = length(data.aws_availability_zones.available.names)
}
variable "private_domain" {
  default = "domain_name.local"
}
 
variable "wp_domain" {
  default = "domain_name"
}
```
### Step 4
Create a datasource.tf file to mention the datasources that we apply in this project.
```sh
data "aws_availability_zones" "available" {
  state = "available"
}


data "aws_route53_zone" "mydomain" {
  name = "domain_name."
}
```
### Step 5

Creating userdata files for the database and webserver

1. Database server -backend.sh
```sh
#!/bin/bash
 
echo "ClientAliveInterval 60" >> /etc/ssh/sshd_config
echo "LANG=en_US.utf-8" >> /etc/environment
echo "LC_ALL=en_US.utf-8" >> /etc/environment
service sshd restart
 
hostnamectl set-hostname backend
amazon-linux-extras install php7.4 -y
rm -rf /var/lib/mysql/*
yum remove mysql -y
 
yum install httpd mariadb-server -y
systemctl restart mariadb.service
systemctl enable mariadb.service
 
mysqladmin -u root password 'mypass123'
mysql -u root -pmypass123 -e "create database db;"
mysql -u root -pmypass123 -e "create user 'dbguser'@'%' identified by 'password123';"
mysql -u root -pmypass123 -e "grant all privileges on blog.* to 'dbuser'@'%'"
mysql -u root -pmypass123 -e "flush privileges"
```
- 2 webserver -frontend.sh
```sh
#!/bin/bash
 
echo "ClientAliveInterval 60" >> /etc/ssh/sshd_config
echo "LANG=en_US.utf-8" >> /etc/environment
echo "LC_ALL=en_US.utf-8" >> /etc/environment
service sshd restart
 
hostnamectl set-hostname frontend-server
 
amazon-linux-extras install php7.4 
yum install httpd -y
systemctl restart httpd
systemctl enable httpd
 
wget https://wordpress.org/latest.zip
unzip latest.zip
cp -rf wordpress/* /var/www/html/
mv /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
chown -R apache:apache /var/www/html/*
cd  /var/www/html/
    
 
sed -i 's/database_name_here/db/g' wp-config.php
sed -i 's/username_here/dbuser/g' wp-config.php
sed -i 's/password_here/password123/g' wp-config.php
sed -i 's/localhost/db.sachin.local/g' wp-config.php
```
### Step 6
Creation of vpc,instances and other aws resources through main.tf file

```sh
resource "aws_vpc" "vpc" {
  cidr_block           = var.vpc_cidr
  instance_tenancy     = "default"
  enable_dns_hostnames = true
  enable_dns_support   = true
 
  tags = {
    Name = "${var.project}-${var.environment}"
  }
}
 
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id
 
  tags = {
    Name = "${var.project}-${var.environment}"
  }
}
 
resource "aws_subnet" "public" {
 
  count                   = local.subnets
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = {
    Name = "${var.project}-${var.environment}-public${count.index + 1}"
  }
}
 
 
resource "aws_subnet" "private" {
 
  count                   = local.subnets
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, "${local.subnets + count.index}")
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = false
  tags = {
    Name = "${var.project}-${var.environment}-private${count.index + 1}"
  }
 
}
 
resource "aws_eip" "nat" {
  vpc = true
  tags = {
    Name = "${var.project}-${var.environment}-natgw"
  }
}
 
resource "aws_nat_gateway" "nat" {
 
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags = {
    Name = "${var.project}-${var.environment}"
  }
  depends_on = [aws_internet_gateway.igw]
}
 
 
resource "aws_route_table" "public" {
 
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "${var.project}-${var.environment}-public"
  }
 
}
 
 
resource "aws_route_table" "private" {
 
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }
  tags = {
    Name = "${var.project}-${var.environment}-private"
  }
 
}
 
 
resource "aws_route_table_association" "public" {
  count          = local.subnets
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
 
 
resource "aws_route_table_association" "private" {
  count          = local.subnets
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
 
 
resource "aws_security_group" "bastion-traffic" {
 
  name_prefix = "${var.project}-${var.environment}-bastion-"
  description = "Allows ssh traffic only"
  vpc_id      = aws_vpc.vpc.id
 
  ingress {
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
 
 
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
 
  tags = {
    "Name" = "${var.project}-${var.environment}-bastion"
  }
 
  lifecycle {
    create_before_destroy = true
  }
}
 
resource "aws_security_group" "frontend-traffic" {
 
  name_prefix = "${var.project}-${var.environment}-frontend-"
  description = "Allow http,https,ssh traffic only"
  vpc_id      = aws_vpc.vpc.id
 
  ingress {
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
 
  ingress {
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
 
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion-traffic.id]
  }
 
 
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
 
  tags = {
    "Name" = "${var.project}-${var.environment}-frontend"
  }
 
  lifecycle {
    create_before_destroy = true
  }
}
 
 
resource "aws_security_group" "backend-traffic" {
 
  name_prefix = "${var.project}-${var.environment}-backend-"
  description = "Allow mysql,ssh traffic only"
  vpc_id      = aws_vpc.vpc.id
 
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.frontend-traffic.id]
  }
 
 
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion-traffic.id]
  }
 
 
 
 
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
 
  tags = {
    "Name" = "${var.project}-${var.environment}-backend"
  }
 
  lifecycle {
    create_before_destroy = true
  }
}
 
 
resource "aws_key_pair" "ssh_key" {
 
  key_name   = "${var.project}-${var.environment}"
  public_key = file("mykey.pub")
  tags = {
    "Name" = "${var.project}-${var.environment}-mykey"
  }
}
 
resource "aws_instance" "bastion" {
 
  ami                    = var.instance_ami
  instance_type          = var.instance_type
  key_name               = aws_key_pair.ssh_key.key_name
  subnet_id              = aws_subnet.public.1.id
  vpc_security_group_ids = [aws_security_group.bastion-traffic.id]
  associate_public_ip_address = true
  user_data_replace_on_change = true
 
  tags = {
    "Name" = "${var.project}-${var.environment}-Bastion"
  }
}
 
resource "aws_instance" "frontend" {
 
  ami                    = var.instance_ami
  instance_type          = var.instance_type
  key_name               = aws_key_pair.ssh_key.key_name
  subnet_id              = aws_subnet.public.0.id
  vpc_security_group_ids = [aws_security_group.frontend-traffic.id]
  associate_public_ip_address = true
  user_data = file("frontend.sh")
  user_data_replace_on_change = true
 
  tags = {
    "Name" = "${var.project}-${var.environment}-Frontend"
  }
}
resource "aws_instance" "backend" {
 
  ami                    = var.instance_ami
  instance_type          = var.instance_type
  key_name               = aws_key_pair.ssh_key.key_name
  subnet_id              = aws_subnet.private.0.id
  associate_public_ip_address = false
  user_data = file("backend.sh")
  user_data_replace_on_change = true
  vpc_security_group_ids = [aws_security_group.backend-traffic.id]
  depends_on = [aws_nat_gateway.nat_gw]
 
  tags = {
    "Name" = "${var.project}-${var.environment}-Backend"
  }
}
 
 
resource "aws_route53_zone" "private" {
  name = var.private_domain
 
  vpc {
    vpc_id = aws_vpc.vpc.id
    
  }
}
resource "aws_route53_record" "mydomain" {
    zone_id = aws_route53_zone.private.zone_id
    name = "db.${var.private_domain}"
    type = "A"
    ttl = "300"
    records = [aws_instance.backend.private_ip]
}
 
resource "aws_route53_record" "wordpress" {
  zone_id = data.aws_route53_zone.mydomain.zone_id
  name    = "wordpress.${var.wp_domain}"
  type    = "A"
  ttl     = 300
  records = [aws_instance.frontend.public_ip]
}
```
### Step 7
Create an output.tf to display the URL for wordpress
```sh
output "site_url" {
  value = "http://wordpress.domain_name"
}
```
## NB:-
It is not neccesory to create the files with the same name that I have created. You may create a single file with .tf extenstion and run the installation.

### Step 8
Installastion of terraform file/provider file is done using the following command
```sh
terraform init
```
### Step 9
Use the below command to check the codes and any corrections on it

```sh
terraform validate
```

### Step 10
To check what are the resources going to build use the command below
```sh
terraform plan
```
### Step 11
To start the installation 
```sh
terraform apply
```




