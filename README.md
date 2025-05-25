# How to deploy a three-tier architecture in AWS using Terraform?

### What is Terraform?

Terraform is an open-source infrastructure as a code (IAC) tool that allows to create, manage & deploy the production-ready environment. Terraform codifies cloud APIs into declarative configuration files. Terraform can manage both existing service providers and custom in-house solutions.

![1](https://github.com/DhruvinSoni30/Terraform-AWS-3tier-Architecture/blob/main/1.png)

In this tutorial, I will deploy a three-tier application in AWS using Terraform.

![2](https://github.com/DhruvinSoni30/Terraform-AWS-3tier-Architecture/blob/main/2.png)

### Prerequisites:

* Basic knowledge of AWS & Terraform
* AWS account
* AWS Access & Secret Key

> In this project, I have used some variables also that I will discuss later in this article.

**Step 1:- Create a file for the VPC**

* Create vpc.tf file and add the below code to it

  ``` # Creating VPC
       resource "aws_vpc" "vpc" {
       cidr_block       = var.vpc_cidr
       tags = {
          Name = "tp-aws-vpc"
       }
  }
  ```
  
**Step 2:- Create a file for the Subnet**

* For this project, I will create total 6 subnets for the front-end tier and back-end tier with a mixture of public & private subnet
* Create subnet.tf file and add the below code to it

```hcl
# Creating 1st public subnet 
resource "aws_subnet" "public_subnet_1" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.subnet_cidr
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "Public Subnet 1"
  }
}

# Creating 2nd public subnet 
resource "aws_subnet" "public_subnet_2" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.subnet1_cidr
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true

  tags = {
    Name = "Public Subnet 2"
  }
}

# Creating 1st private subnet 
resource "aws_subnet" "private_subnet_1" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.subnet2_cidr
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = false

  tags = {
    Name = "Private Subnet 1"
  }
}

# Creating 2nd private subnet 
resource "aws_subnet" "private_subnet_2" {
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = var.subnet3_cidr
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = false

  tags = {
    Name = "Private Subnet 2"
  }
}

# Creating 1st database subnet 
resource "aws_subnet" "database_subnet_1" {
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = var.subnet4_cidr
  availability_zone = "us-east-1a"

  tags = {
    Name = "Database Subnet 1"
  }
}

# Creating 2nd database subnet 
resource "aws_subnet" "database_subnet_2" {
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = var.subnet5_cidr
  availability_zone = "us-east-1b"

  tags = {
    Name = "Database Subnet 2"
  }
}
```

  
**Step 3:- Create a file for the Internet Gateway**

* Create igw.tf file and add the below code to it

```hcl
resource "aws_internet_gateway" "ig1" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "Internet Gateway"
  }
}
  ```

**Step 4:- Create a file for the Route table**

* Create route_table_public.tf file and add the below code to it

```hcl
  # Creating Route Table
 resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.ig1.id
  }

  tags = {
    Name = "Route Table Public"
  }
}

resource "aws_route_table_association" "public_subnet_1_assoc" {
  subnet_id      = aws_subnet.public_subnet_1.id
  route_table_id = aws_route_table.public-rt.id
}

resource "aws_route_table_association" "public_subnet_2_assoc" {
  subnet_id      = aws_subnet.public_subnet_2.id
  route_table_id = aws_route_table.public-rt.id
}
 ```
* In the above code, I am creating a new route table and forwarding all the requests to the 0.0.0.0/0 CIDR block.
* I am also attaching this route table to the subnet created earlier. So, it will work as the Public Subnet

**Step 5:- Create a file for EC2 instances**

* Create ec2.tf file and add the below code to it


```hcl 
resource "aws_instance" "my1ec2" {
  ami             = data.aws_ami.amazon_linux.id
  instance_type   = "t2.nano"
  key_name        = "devops-stevy"
  subnet_id       = aws_subnet.public_subnet_1.id
  security_groups = [aws_security_group.allow_ssh_http_https.id]
  tags = {
    Name = "Ec21"
  }

  user_data = file("data.sh")
}

data "aws_ami" "amazon_linux" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["amazon"]
}


resource "aws_instance" "my2ec2" {
  ami             = data.aws_ami.amazon_linux.id
  instance_type   = "t2.nano"
  key_name        = "devops-stevy"
  subnet_id       = aws_subnet.public_subnet_2.id
  security_groups = [aws_security_group.allow_ssh_http_https.id]
  tags = {
    Name = "Ec22"
  }

  user_data = file("data.sh")
}
```

* I have used the userdata to configure the EC2 instance, I will discuss data.sh file later in the article

**Step 6:- Create a file for Security Group for the FrontEnd tier**

* Create sg.tf file and add the below code to it

```hcl 
  # Creating Security Group 
  resource "aws_security_group" "allow_ssh_http_https" {
  name        = "stevy-sg"
  description = "Allow SSH, HTTP and HTTPS inbound traffic"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description     = "HTTP from anywhere"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id] # Only from ALB
  }

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

* I have opened 80,443 & 22 ports for the inbound connection and I have opened all the ports for the outbound connection

**Step 7:- Create a file for Security Group for the Database tier and ALB**

* Create database_sg.tf file and add the below code to it

 ```hcl 
  # Create Security Group for RDS Instance
resource "aws_security_group" "rds_security_group" {
  vpc_id = aws_vpc.vpc.id

  # Inbound rule to accept connections from EC2 security group on port 3306
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.allow_ssh_http_https.id]
  }

  egress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.allow_ssh_http_https.id]
  }
}
# Create Security Group for ALB
resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow HTTP from anywhere to ALB"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Public access to ALB
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
 ```
* I have opened 3306 ports for the inbound connection and I have opened all the ports for the outbound connection.

**Step 8:- Create a file Application Load Balancer**

* Create alb.tf file and add the below code to it

```hcl 
  # Creating Application LoadBalancer
   resource "aws_lb" "alb" {
  name               = "my-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.public_subnet_1.id, aws_subnet.public_subnet_2.id]
}

resource "aws_lb_target_group" "tg" {
  name     = "my-targets"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.vpc.id
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

resource "aws_lb_target_group_attachment" "my1ec2" {
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = aws_instance.my1ec2.id
  port             = 80
}

resource "aws_lb_target_group_attachment" "my2ec2" {
  target_group_arn = aws_lb_target_group.tg.arn
  target_id        = aws_instance.my2ec2.id
  port             = 80
}
```
* The above load balancer is of type external
* Load balancer type is set to application
* The aws_lb_target_group_attachment resource will attach our instances to the Target Group.
* The load balancer will listen requests on port 80

**Step 9:- Create a file for the RDS instance**

* Create a rds.tf file and add the below code to it


```hcl 
 # Create DB Subnet Group
 resource "aws_db_subnet_group" "my_db_subnet_group" {
  name       = "my-db-subnet-group"
  subnet_ids = [aws_subnet.database-subnet-1.id, aws_subnet.database-subnet-2.id]

  tags = {
    Name = "my-db-subnet-group"
  }
}

# Create RDS Instance
resource "aws_db_instance" "my_rds" {
  identifier             = "my-rds"
  engine                 = "mysql"
  engine_version         = "5.7"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  storage_type           = "gp2"
  username               = "admin"
  password               = "password"
  vpc_security_group_ids = [aws_security_group.rds_security_group.id]
  db_subnet_group_name   = aws_db_subnet_group.my_db_subnet_group.name
  multi_az               = true
  skip_final_snapshot    = true

}
```
* In the above code, you need to change the value of username & password
* multi-az is set to true for the high availability

**Step 10:- Create a file for outputs**

* Create outputs.tf file and add the below code to it



```hcl 
 # Getting the DNS of load balancer
output "lb_dns_name" {
  description = "The DNS name of the load balancer"
  value       = aws_lb.alb.dns_name
}
```
  
* From the above code, I will get the DNS of the application load balancer.

**Step 11:- Create a file for variable**

* Create vars.tf file and add the below code to it

 
```hcl 
  # Defining CIDR Block for VPC
variable "vpc_cidr" {
  default = "10.0.0.0/16"
}
# Defining CIDR Block for 1st Subnet
variable "subnet_cidr" {
  default = "10.0.0.0/24"
}
variable "subnet1_cidr" {
  default = "10.0.1.0/24"
}
# Defining CIDR Block for 2nd Subnet
variable "subnet2_cidr" {
  default = "10.0.2.0/24"
}
# Defining CIDR Block for 3rd Subnet
variable "subnet3_cidr" {
  default = "10.0.3.0/24"
}
# Defining CIDR Block for 3rd Subnet
variable "subnet4_cidr" {
  default = "10.0.4.0/24"
}
# Defining CIDR Block for 3rd Subnet
variable "subnet5_cidr" {
  default = "10.0.5.0/24"
}
# Defining CIDR Block for 3rd Subnet
 ```

**Step 12:- Create a file for user data**

* Create data.sh file and add the below code to it

``hcl 
#!/bin/bash
              yum update -y
              amazon-linux-extras install nginx1 -y
              systemctl enable nginx
              systemctl start nginx
              EOF
  ```
  
* The above code will install an apache webserver in the EC2 instances

So, now our entire code is ready. We need to run the below steps to create infrastructure.

* terraform init is to initialize the working directory and downloading plugins of the provider
* terraform plan is to create the execution plan for our code
* terraform apply is to create the actual infrastructure. It will ask you to provide the Access Key and Secret Key in order to create the infrastructure. So, instead of hardcoding the Access Key and Secret Key, it is better to apply at the run time.


**Step 13:- Verify the resources**

* Terraform will create below resources

  * VPC
  * Application Load Balancer
  * Public & Private Subnets
  * EC2 instances
  * RDS instance
  * Route Table
  * Internet Gateway
  * Security Groups for Web & RDS instances
  * Route Table

Once the resource creation finishes you can get the DNS of a load balancer and paste it into the browser and you can see load balancer will send the request to two instances.

That’s it now, you have learned how to create various resources in AWS using Terraform.

 
