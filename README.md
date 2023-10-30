# Setup-Wordpress-securely-in-a-custom-aws-vpc-using-terraform
# Description

Database servers are the most important servers in any small or big company because it contains all the data sets that can be of any user or employee of that company. Therefore, the best practices of server security are focused on protecting digital assets from cybercriminals. This approach is necessary because numerous hackers exploit existing vulnerabilities for financial gain. Here we set up a custom-made secure setup for installing WordPress and protecting our server.

--------------------------------------------------------------------------------------------------------------------



## Provider
* A provider is a plugin that lets Terraform manage an external API

```
provider "aws" {

  region     = "ap-south-1"
  access_key = "Your_access_key"
  secret_key = "Your_secrey_key"
}
```
---------------------------------------------------------------------------------------------------------------------
##Data Resource

* Data Resource - allow you to write configuration that is flexible and easier to re-use - Data sources allow Terraform to use the information defined outside of Terraform

```
data "aws_route53_zone" "selected" {
  name         = "learndevops.website"
  private_zone = false
}

data "aws_availability_zones" "available" {
  state = "available"
}
```
---------------------------------------------------------------------------------------------------------------------
* ***Variable Creation*** - allows you to write a configuration that is flexible and easier to re-use

##Variable

```
variable "project_name" {

  description = "my project name"
  type        = string
  default     = "zomato"
}

variable "project_environment" {

  description = "my project environment"
  type        = string
  default     = "prod"
}

variable "instance_type" {

  description = "my instance type"
  type        = string
  default     = "t2.micro"
}

variable "ami_id" {

  description = "my instance ami id"
  type        = string
  default     = "ami-057752b3f1d6c4d6c"
}

variable "vpc_cidr_block" {

  type    = string
  default = "172.16.0.0/16"
}

variable "main_cidr_block" {

  default = "172.16.0.0/16"
}

variable "region" {

  type    = string
  default = "ap-south-1"
}

variable "private_zone_name" {

  type    = string
  default = "learndevops.local"
}

variable "enable_natgw" {
  type    = bool
  default = true #or false
}
```
---------------------------------------------------------------------------------------------------------------
* ***Output Creation***  - Terraform output values let you export structured data about your resources
##Output

```
output "frontend_public_ip" {
  value = aws_instance.frontend.public_ip

}

output "frontend_private_ip" {

  value = aws_instance.frontend.private_ip


}

output "frontend_public_dns" {

  value = aws_instance.frontend.public_dns

}

output "frontend_url" {

  value = "http://${aws_instance.frontend.public_dns}"

}


output "aws_security_group_frontend" {
  value = aws_security_group.frontend.id

}

============================================

output "bastion_public_ip" {
  value = aws_instance.bastion.public_ip

}

output "bastion_private_ip" {

  value = aws_instance.bastion.private_ip


}

output "bastion_public_dns" {

  value = aws_instance.bastion.public_dns

}


output "aws_security_group_bastion" {
  value = aws_security_group.bastion.id

}

==================================================


output "backend_private_ip" {

  value = aws_instance.backend.private_ip


}

output "bastion_private_dns" {

  value = aws_instance.backend.private_dns

}


output "aws_security_group_backend" {
  value = aws_security_group.backend.id

}

====================================================


#cidrsubnet output

output "main_cidr_block_subnet1" {

  value = cidrsubnet(var.main_cidr_block, 3, 0)


}

output "main_cidr_block_subnet2" {

  value = cidrsubnet(var.main_cidr_block, 3, 1)


}
output "main_cidr_block_subnet3" {

  value = cidrsubnet(var.main_cidr_block, 3, 2)


}
output "main_cidr_block_subnet4" {

  value = cidrsubnet(var.main_cidr_block, 3, 3)


}

output "main_cidr_block_subnet5" {

  value = cidrsubnet(var.main_cidr_block, 3, 4)


}
output "main_cidr_block_subnet6" {

  value = cidrsubnet(var.main_cidr_block, 3, 5)


}


==================================================

#Route53 public hosted zone

output "details_of_zone" {
  value = data.aws_route53_zone.selected.id
}


#Availability zone names

output "details_of_availability_zone" {
  value = data.aws_availability_zones.available.names
}
```

##Wordpress userdata

```
#!/bin/bash


yum install httpd php php-mysqlnd -y
systemctl restart httpd php-fpm
systemctl enable httpd php-fpm

cd /tmp/
wget  https://wordpress.org/latest.tar.gz
tar -xvf latest.tar.gz
mv wordpress/* /var/www/html/
chown -R apache: apache /var/www/html/
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php


sed -i "s/database_name_here/blogdb/" /var/www/html/wp-config.php

sed -i "s/username_here/wpuser/" /var/www/html/wp-config.php

sed -i "s/password_here/wpuser1234/" /var/www/html/wp-config.php

sed -i "s/localhost/backend.learndevops.local/" /var/www/html/wp-config.php

rm -rf wordpress
```
----------------------------------------------------------------------------------------------
##Mysql userdata

```
#!/bin/bash


yum install mariadb105-server -y
systemctl restart mariadb.service
systemctl enable mariadb.service

mysql -u root -e "create database blogdb;"

mysql -u root -e "create user wpuser@'%' identified by 'wpuser1234' ;"

mysql -u root -e "grant all privileges on blogdb.* to wpuser@'%';"

mysql -u root -e "flush privileges;"
```
-----------------------------------------------------------------------------------------------------
##Main

```
#vpc creation

resource "aws_vpc" "my-vpc" {
  cidr_block           = var.vpc_cidr_block
  instance_tenancy     = "default"
  enable_dns_hostnames = "true"

  tags = {
    Name        = "${var.project_name}-${var.project_environment}"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#igw creation

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.my-vpc.id

  tags = {
    Name        = "${var.project_name}-${var.project_environment}"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#private subnets creation

resource "aws_subnet" "private" {

  count                   = 3
  vpc_id                  = aws_vpc.my-vpc.id
  cidr_block              = cidrsubnet(var.main_cidr_block, 3, (count.index))
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = "false"

  tags = {
    Name        = "${var.project_name}-${var.project_environment}-private${count.index + 1}"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#public subnets creation

resource "aws_subnet" "public" {

  count                   = 3
  vpc_id                  = aws_vpc.my-vpc.id
  cidr_block              = cidrsubnet(var.main_cidr_block, 3, (count.index + 3))
  map_public_ip_on_launch = "true"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  tags = {
    Name        = "${var.project_name}-${var.project_environment}-public${count.index + 4}"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#elastic-ip  creation

resource "aws_eip" "nat" {

  count  = var.enable_natgw == true ? 1 : 0
  domain = "vpc"
}

#nat-gw creation

resource "aws_nat_gateway" "my-nat" {

  count         = var.enable_natgw == true ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[2].id

  tags = {
    Name        = "${var.project_name}-${var.project_environment}"
    Environment = var.project_environment
    Project     = var.project_name
  }

  depends_on = [aws_internet_gateway.igw]
}


#public route table creation

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.my-vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name        = "${var.project_name}-${var.project_environment}-public"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#private route table creation

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.my-vpc.id


  tags = {
    Name        = "${var.project_name}-${var.project_environment}-private"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#nat-gw route creation

resource "aws_route" "nat" {

  count                  = var.enable_natgw == true ? 1 : 0
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.my-nat[0].id
}



#private route table assosciation

resource "aws_route_table_association" "private" {

  count          = 3
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

#public route table assosciation

resource "aws_route_table_association" "public" {

  count          = 3
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}


#Key-pair creation

resource "aws_key_pair" "wordpress" {
  key_name   = "${var.project_name}-${var.project_environment}"
  public_key = file("wordkey.pub")

  tags = {
    Name        = "${var.project_name}-${var.project_environment}"
    Environment = var.project_environment
    Project     = var.project_name
  }

}



#bastion security group creation

resource "aws_security_group" "bastion" {
  name_prefix = "${var.project_name}-${var.project_environment}-bastion"
  description = "Allow shh from all"
  vpc_id      = aws_vpc.my-vpc.id

  ingress {
    description      = "allow ssh from all"
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
    Name        = "${var.project_name}-${var.project_environment}-bastion"
    Environment = var.project_environment
    Project     = var.project_name
  }

}


#frontend security group creation

resource "aws_security_group" "frontend" {

  name_prefix = "${var.project_name}-${var.project_environment}-frontend"
  description = "Allow https and http traffic"
  vpc_id      = aws_vpc.my-vpc.id

  ingress {
    description = "allow https"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  ingress {
    description = "allow http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description     = "allow ssh from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-${var.project_environment}-frontend"
    Environment = var.project_environment
    Project     = var.project_name
  }
}

#backend security group creation

resource "aws_security_group" "backend" {
  name_prefix = "${var.project_name}-${var.project_environment}--backend"
  description = "Allow https and http traffic"
  vpc_id      = aws_vpc.my-vpc.id

  ingress {
    description     = "allow mysql from frontend"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.frontend.id]
  }

  ingress {
    description     = "allow ssh from bastion"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }




  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-${var.project_environment}-backend"
    Environment = var.project_environment
    Project     = var.project_name
  }

}


#bastion ec2-instance creation

resource "aws_instance" "bastion" {

  ami                    = var.ami_id
  instance_type          = var.instance_type
  key_name               = aws_key_pair.wordpress.id
  subnet_id              = aws_subnet.public[1].id
  vpc_security_group_ids = [aws_security_group.bastion.id]


  tags = {
    Name        = "${var.project_name}-${var.project_environment}-bastion"
    Environment = var.project_environment
    Project     = var.project_name
  }
  lifecycle {
    create_before_destroy = true
  }

}


#backend ec2-instance creation

resource "aws_instance" "backend" {

  ami                         = var.ami_id
  instance_type               = var.instance_type
  key_name                    = aws_key_pair.wordpress.id
  subnet_id                   = aws_subnet.private[1].id
  vpc_security_group_ids      = [aws_security_group.backend.id]
  user_data                   = file("mysql_userdata.sh")
  user_data_replace_on_change = true


  tags = {
    Name        = "${var.project_name}-${var.project_environment}-backend"
    Environment = var.project_environment
    Project     = var.project_name
  }
  lifecycle {
    create_before_destroy = true
  }


#frontend ec2-instance creation

resource "aws_instance" "frontend" {

  ami                         = var.ami_id
  instance_type               = var.instance_type
  key_name                    = aws_key_pair.wordpress.id
  subnet_id                   = aws_subnet.public[0].id
  vpc_security_group_ids      = [aws_security_group.frontend.id]
  user_data                   = file("wordpress_userdata.sh")
  user_data_replace_on_change = true

  tags = {
    Name        = "${var.project_name}-${var.project_environment}-frontend"
    Environment = var.project_environment
    Project     = var.project_name
  }

  lifecycle {
    create_before_destroy = true
  }


}


#route53 private record creation

resource "aws_route53_zone" "private" {
  name = var.private_zone_name

  vpc {
    vpc_id = aws_vpc.my-vpc.id
  }
  tags = {
    Name        = "${var.private_zone_name}"
    Environment = var.project_environment
    Project     = var.project_name
  }


}


resource "aws_route53_record" "frontend" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "frontend"
  type    = "A"
  ttl     = 300
  records = [aws_instance.frontend.private_ip]
}

resource "aws_route53_record" "bastion" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "bastion"
  type    = "A"
  ttl     = 300
  records = [aws_instance.bastion.private_ip]
}


resource "aws_route53_record" "backend" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "backend"
  type    = "A"
  ttl     = 300
  records = [aws_instance.backend.private_ip]
}

#route53 public record creation

resource "aws_route53_record" "public" {

  zone_id = data.aws_route53_zone.selected.id
  name    = "blog"
  type    = "A"
  ttl     = "300"
  records = [aws_instance.frontend.public_ip]
}

```
