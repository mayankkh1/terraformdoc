## Description:

- Launch an ec2 instance from terraform.
- deploy nginx server in that instance. 
- Show the IP of the server in the output. 
- Create these infra using terraform modules launch ec2 instance, configure ELB, ASG
- Create IAM, Policies & assigning to users and show Hello World instead of IP on webserver using root file at root location 

#### Step-1: Install terraform in local machine.

- Before installing any software, we need to update the system. To do so, run the below command:

  ```sudo apt-get update -y```

- Also, make sure to install (or check whether you already have) the following required packages:

  ```sudo apt-get install -y gnupg software-properties-common```
  
- Install the HashiCorp GPG key.
  
  ```
  wget -O- https://apt.releases.hashicorp.com/gpg | \
    gpg --dearmor | \
    sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
  
  ```
  
- Verify the key's fingerprint.

  ```
  gpg --no-default-keyring \
    --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
    --fingerprint
  ```
  
- Now add the official HashiCorp repository to your system with below command.
  
  ```
  echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list
  ```
 
- Download the package information from HashiCorp.

  ```sudo apt update```
  
- Install Terraform from the new repository.

  ```sudo apt-get install terraform```
  

#### Step-2: Install visual studio in local machine.  
  
- Run the following command to update the system's repository and ensure you get the latest vscode version:

  ```sudo apt update```
  
- Install Package Dependencies

  ```sudo apt install software-properties-common apt-transport-https wget -y```
  
- Import the GPG key provided by Microsoft to verify the package integrity. Enter:

  ```wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | sudo apt-key add -```
  
- Run the following command to add the Visual Studio Code repository to your system:
  
  ```sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main"```
  
- After enabling the repository, install vscode by running:

  ```sudo apt install code```
  
- Verify vscode installation by running:

  ```code --version```
  
#### Step-3: Once visual studio installed. You need to install the extension of terraform in visual studio.

#### Step-4:  Launch an ec2 instance from terraform

- For creating the ec2 instance through terraform we need a provider to connect with aws.
  
  ```
  A provider in Terraform is a plugin that enables interaction with an API. This includes Cloud providers such as AWS. 
  The providers are specified in the Terraform configuration code, telling Terraform which services it needs to interact with.
  ```
- If you want specific version of provider use below lines in your main.tf file.
   
  ``` 
  terraform {
    required_providers {
      aws = {
       source  = "hashicorp/aws"
       version = "~> 4.0"
       }
     }
   }
  
  ``` 
  
- After that we need to configure the provider in main.tf. For connectivity following methods are supported:

  - Static credentials
  - Environment variables
  - Shared credentials file
 
    Static credentials can be provided by adding an access_key and secret_key in-line in the AWS provider block as like below:
 
    ```
    provider "aws" {
    region     = "us-west-2"
    access_key = "my-access-key"
    secret_key = "my-secret-key"
    }
    ```
   
    Credentials can be provided through environment variables as like below:
    
    ```
    export AWS_ACCESS_KEY_ID="anaccesskey"
    export AWS_SECRET_ACCESS_KEY="asecretkey"
    export AWS_DEFAULT_REGION="us-west-2"
    ```
    
    The AWS Provider can source credentials and other settings from the shared configuration and credentials files as like below:
    
    ```
    provider "aws" {
      shared_config_files      = ["/home/ongraph/.aws/config"]
      shared_credentials_files = ["/home/ongraph/.aws/credentials"]
      profile                 = "default"
    }
    
    ```
 
    In config file region is mentioned and in credential file we have mentioned access key and secret key.
    
- I am using the shared configuration and credentials for configure and authentication aws provider. 

- Create the network file(network.tf) for creating VPC and subnets.

  First we need to add the variables in variable.tf file to use in network file.
  
  ```
  variable "enable_dns_hostnames" {
    type        = bool
    description = "Enable Dns hostname in VPC"
    default     = true
  }
  variable "vpc_cidr_block" {
    type        = string
    description = "Base CIDR block for VPC"
    default     = "10.0.0.0/16"
  }
  variable "vpc_subnet_count" {
    type        = string
    description = "Subnet count number"
    default     = 2
  }

  ```
  
  Now create the network file and add the below lines for creating the network through terraform.

  ```
  data "aws_availability_zones" "available" {
  state = "available"
   }

  resource "aws_vpc" "vpc" {
    cidr_block           = var.vpc_cidr_block
    enable_dns_hostnames = var.enable_dns_hostnames

  }

  resource "aws_internet_gateway" "igw" {
    vpc_id = aws_vpc.vpc.id

  }

  resource "aws_subnet" "subnet" {
    count                   = var.vpc_subnet_count
    cidr_block              = cidrsubnet(var.vpc_cidr_block, 8, count.index)
    vpc_id                  = aws_vpc.vpc.id
    map_public_ip_on_launch = var.map_public_ip_on_launch
    availability_zone       = data.aws_availability_zones.available.names[count.index]

  }

  resource "aws_route_table" "rtb" {
    vpc_id = aws_vpc.vpc.id

    route {

      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.igw.id

    }


  }

  resource "aws_route_table_association" "rtba" {
    count          = var.vpc_subnet_count
    subnet_id      = aws_subnet.subnet[count.index].id
    route_table_id = aws_route_table.rtb.id
  }

  resource "aws_security_group" "sg" {
    vpc_id = aws_vpc.vpc.id
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
      from_port   = 22
      to_port     = 22
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

- Create the EC2 instance with instance.tf file 
  
  First we need to add the variables in variable.tf file for use them in instance.tf file.
  
  ```
  variable "instance_type" {
    type        = string
    description = "Type for EC2 Instance"
    default     = "t2.micro"
  }

  variable "instance_count" {
    type        = string
    description = "No of instance"
    default     = 1
  }

  variable "ami_id" {
    type        = string
    description = "AMI for centod Image"
    default     = "ami-0763cf792771fe1bd"
  }


  variable "Key_name" {
    type        = string
    description = "Key for Centos Machine"
    default     = " test"
  }
  ```
  
  Now create the instance file and add the below lines for creating the EC2 instance through terraform.
  
  ```
  resource "aws_instance" "vm" {
  count                  = var.instance_count
  ami                    = var.ami_id
  subnet_id              = aws_subnet.subnet[(count.index % var.vpc_subnet_count)].id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.sg.id]
  key_name               = var.Key_name
  }
  ```
  
  Note: In this we have use resources(subnet_id and vpc_security_group_ids) of network file
  
  Once the above files are created we need to run the below command of terraform to create the resources in aws.
  
  ```terraform init```
  
  - This is the first command you should use to initialize the working directory and installed required plugins.
  
  ```terraform plan```
  
  - The terraform plan command lets you to preview the actions which Terraform would take to modify your infrastructure, or save a speculative plan 
    which you can apply later. 
  - The function of terraform plan is speculative: you cannot apply it unless you save its contents and pass them to a terraform apply command. 
  
  ```terraform apply```
  
  - The terraform apply command is the more common workflow outside of automation. 
  - If you do not pass a saved plan to the apply command, then it will perform all of the functions of plan and prompt you for approval before making the     changes.

- After apply command you got EC2 instance in your aws enviroment.

#### Step-5:  Deploy nginx server in EC2 instance.

- For this we can add the user data for running the commands in instance. 
  
  We need to add the below lines for installing the nginx server in instance.tf file
  
  ```
  resource "aws_instance" "vm" {
    count                  = var.instance_count
    ami                    = var.ami_id
    subnet_id              = aws_subnet.subnet[(count.index % var.vpc_subnet_count)].id
    instance_type          = var.instance_type
    vpc_security_group_ids = [aws_security_group.sg.id]
    key_name               = var.Key_name
    user_data              = <<EOF
  #! /bin/bash 
  sudo yum install epel-release -y
  sudo yum install bind-utils -y 
  sudo yum install nginx -y
  sudo systemctl start nginx
  sudo systemctl enable nginx
  EOF
  ```
 
#### Step-5:  Show the IP of the server in the output in console and website.

- For console you need to create a output.tf file and below lines:
  
  ```
  output "aws_instance_public_ip" {

    value = aws_instance.vm[*].public_ip
  }
 
- If you want to print the IP in website. We need to add the below lines in user data
  
  ```
  resource "aws_instance" "vm" {
    count                  = var.instance_count
    ami                    = var.ami_id
    subnet_id              = aws_subnet.subnet[(count.index % var.vpc_subnet_count)].id
    instance_type          = var.instance_type
    vpc_security_group_ids = [aws_security_group.sg.id]
    key_name               = var.Key_name
    user_data              = <<EOF
  #! /bin/bash
  sudo yum install epel-release -y
  sudo yum install bind-utils -y 
  sudo yum install nginx -y
  sudo systemctl start nginx
  sudo systemctl enable nginx
  sudo rm /usr/share/nginx/html/index.html
  sudo echo  Server Ip is `dig +short myip.opendns.com @resolver1.opendns.com` | sudo tee /usr/share/nginx/html/index.html
  sudo echo "@reboot echo "Server Ip is this `dig +short myip.opendns.com @resolver1.opendns.com`"| sudo tee /usr/share/nginx/html/index.html" | tee -a     /var/spool/cron/root
  EOF
  }
  ``` 
  - First command is used for get the public IP and put IP in file with tee command.
    ```sudo echo  Server Ip is `dig +short myip.opendns.com @resolver1.opendns.com` | sudo tee /usr/share/nginx/html/index.html```
  
  - Second command is used to add command with cronjob. If server rebooted it will automatically update the new IP in index file. 
    ```sudo echo "@reboot echo "Server Ip is this `dig +short myip.opendns.com @resolver1.opendns.com`"| sudo tee /usr/share/nginx/html/index.html" | tee -a  /var/spool/cron/root``` 

 
#### Step-6:  Create the infra using terraform modules launch ec2 instance, configure ELB, ASG
 
- Module allows you to group resources together and reuse this group later, possibly many times 

- We can create our custom modules or use provider module(AWS modules) which is in-build module.

- You can visit the below url for in-build modules.

  ```https://registry.terraform.io/namespaces/terraform-aws-modules```
 
- First we use in-build modules for build the infra.
  
  Let us create the network file as like above and use aws module rather than resources in that file.
  
  We need to create variable file to use the variable in network file.
  
  ```
  variable "enable_dns_hostnames" {
    type        = bool
    description = "Enable Dns hostname in VPC"
    default     = true
  }
  
  variable "vpc_cidr_block" {
    type        = string
    description = "Base CIDR block for VPC"
    default     = "10.0.0.0/16"
  }
  
  variable "map_public_ip_on_launch" {
    type        = bool
    description = "make public IP enable on VM"
    default     = true
  }
  
  variable "vpc_subnet_count" {
    type        = string
    description = "Subnet count number"
    default     = 2
  }
  ``` 
  Now create the network file as like below:  
  
  ```
  data "aws_availability_zones" "available" {
  state = "available"
  }
  module "vpc" {
    source = "terraform-aws-modules/vpc/aws"

    name = "my-vpc"
    cidr = var.vpc_cidr_block
    azs = slice(data.aws_availability_zones.available.names, 0, (var.vpc_subnet_count))
    public_subnets = [for subnet in range(var.vpc_subnet_count) : cidrsubnet(var.vpc_cidr_block, 8, subnet)]
    enable_nat_gateway      = false
    map_public_ip_on_launch = var.map_public_ip_on_launch
    enable_dns_hostnames    = var.enable_dns_hostnames

    }
  resource "aws_security_group" "sg" {
    vpc_id = module.vpc.vpc_id
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
      from_port   = 22
      to_port     = 22
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
  
  Create autoscaling wth aws modules. 
    
  We need to add the variables in variable.tf file for use them in scaling.tf file.
  
  ```
  variable "instance_type" {
    type        = string
    description = "Type for EC2 Instance"
    default     = "t2.micro"
  }
  variable "scale_count" {
    type        = string
    description = "No of instance"
    default     = 1
  }

  variable "ami_id" {
    type        = string
    description = "AMI for centod Image"
    default     = "ami-0763cf792771fe1bd"
  }
  variable "Key_name" {
    type        = string
    description = "Key for Centos Machine"
    default     = "mayank"
  }
  variable "vpc_subnet_count" {
    type        = string
    description = "Subnet count number"
    default     = 2
  }

  ```
  Now create the scaling file and add the below lines for creating the auto scaling through terraform.
  
  ```
  module "asg" {
    source = "terraform-aws-modules/autoscaling/aws"

    # Autoscaling group
    name             = "asg"
    count            = var.scale_count
    min_size         = 1
    max_size         = 3
    desired_capacity = 1

    health_check_type   = "EC2"
    vpc_zone_identifier = [module.vpc.public_subnets[(count.index % var.vpc_subnet_count)]]
    security_groups     = [aws_security_group.sg.id]

    # Launch template
    launch_template_name        = "myasg"
    launch_template_description = "myasgforvm"
    update_default_version      = true

    image_id          = var.ami_id
    instance_type     = var.instance_type
    user_data         = filebase64("deploy.sh")
    target_group_arns = module.alb.target_group_arns

  }
    
  ```  
  Now we need to enter the bash file in same directory for user data.
  
  ```
  #! /bin/bash
  sudo yum install epel-release -y
  sudo yum install bind-utils -y 
  sudo yum install nginx -y
  sudo systemctl start nginx
  sudo systemctl enable nginx
  sudo rm /usr/share/nginx/html/index.html
  sudo echo  Hello world1 | sudo tee /usr/share/nginx/html/index.html
  ```
  
  Create loadbalancer with aws module
  
  we need to add the variables in variable.tf file for use them in loadbalancer.tf file.
  
  ```
  variable "name_prefix" {
    type        = string
    description = "Naming prefix for resources"
    default     = "myweb" 
  }
  
  ```
  Now create the loadbalancer file and add the below lines for creating the loadbalancer through terraform.
  
  ```
  module "alb" {
    source             = "terraform-aws-modules/alb/aws"
    version = "~> 8.0"
    name               = "my-lb"
    load_balancer_type = "application"
    vpc_id             = module.vpc.vpc_id
    subnets            = module.vpc.public_subnets
    security_groups     = [aws_security_group.sg.id]
    target_groups = [
      {
        name_prefix      = "${var.name_prefix}"
        backend_protocol = "HTTP"
        backend_port     = 80
        target_type      = "instance"
      }
    ]

    http_tcp_listeners = [
      {
        port               = 80
        protocol           = "HTTP"
        target_group_index = 0
      }
    ]
 
  }
  ```
 
  After that run below terraform commands for create the resources and import the module
  
  ```terraform init```
  ```terraform plan```
  ```terraform apply```
  
  All done.
  
#### Step-7:  Create the infra using terraform custom modules launch ec2 instance, configure ELB, ASG  
  
- We can use custom modules for build the infra.   

- Let us create the network folder (VPC) and create create custome module.
  
  First we need to create variable file to use the variable in network file which is present in VPC folder
  
  ```
  variable "enable_dns_hostnames" {
    type        = bool
    description = "Enable Dns hostname in VPC"
    default     = true
  }

  variable "vpc_cidr_block" {
    type        = string
    description = "Base CIDR block for VPC"
    default     = "10.0.0.0/16"
  }

  variable "vpc_subnet_cidr_block" {
    type        = string
    description = "Base CIDR block for VPC"
    default     = "10.0.0.0/24"
  }

  variable "map_public_ip_on_launch" {
    type        = bool
    description = "make public IP enable on VM"
    default     = true

  }

  variable "vpc_subnet_count" {
    type        = string
    description = "Subnet count number"
    default     = 2
  }

  variable "name_prefix" {
    type        = string
    description = "Naming prefix for resources"
    default     = "mywebsite"
  }

  variable "azs" {
    description = "A list of availability zones names or ids in the region"
    type        = list(string)
    default     = []
  }

  variable "public_subnets" {
    description = "A list of availability zones names or ids in the region"
    type        = list(string)
  }
  ```
  
  Now create the network file as like below in same VPC folder
  
  ```
  resource "aws_vpc" "vpc" {
    cidr_block           = var.vpc_cidr_block
    enable_dns_hostnames = var.enable_dns_hostnames

  }

  resource "aws_internet_gateway" "igw" {
    vpc_id = aws_vpc.vpc.id

  }

  resource "aws_subnet" "subnet" {
    count                   = var.vpc_subnet_count
    cidr_block              = cidrsubnet(var.vpc_cidr_block, 8, count.index)
    vpc_id                  = aws_vpc.vpc.id
    map_public_ip_on_launch = var.map_public_ip_on_launch
    availability_zone       = var.azs[count.index]

  }

  resource "aws_route_table" "rtb" {
    vpc_id = aws_vpc.vpc.id

    route {

      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.igw.id

    }


  }

  resource "aws_route_table_association" "rtba" {
    count          = var.vpc_subnet_count
    subnet_id      = aws_subnet.subnet[count.index].id
    route_table_id = aws_route_table.rtb.id
  }

  resource "aws_security_group" "sg" {
    vpc_id = aws_vpc.vpc.id
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
      from_port   = 22
      to_port     = 22
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
 
  Now create the ouput file to use the modules output in another modules:
  
  ```
  output "public_subnets" {
    description = "List of IDs of public subnets"
    value       = aws_subnet.subnet[*].id
  }

  output "vpc_cidr_block" {
    description = "The CIDR block of the VPC"
    value       = aws_vpc.vpc.cidr_block 
  }

  output "vpc_id" {
    description = "The CIDR block of the VPC"
    value       = aws_vpc.vpc.id
  }

  output "azs" {
    description = "A list of availability zones specified as argument to this module"
    value       = var.azs
  }

  ```
  
- Now use this module in our main network file and add the parameters values in variable of custom module variable.
 
  Before that we need to create variable file for use the variables in main network file.
  
  ```
  variable "enable_dns_hostnames" {
    type        = bool
    description = "Enable Dns hostname in VPC"
    default     = true
  }
  
  variable "vpc_cidr_block" {
    type        = string
    description = "Base CIDR block for VPC"
    default     = "10.0.0.0/16"
  }
  
  variable "map_public_ip_on_launch" {
    type        = bool
    description = "make public IP enable on VM"
    default     = true

  }

  variable "vpc_subnet_count" {
    type        = string
    description = "Subnet count number"
    default     = 2
  }

  ```
  
  Main network file is look like below: 
  
  ```
  data "aws_availability_zones" "available" {
    state = "available"
  }

  module "vpc" {
    source         = "./vpc"
    vpc_cidr_block = var.vpc_cidr_block
    azs = slice(data.aws_availability_zones.available.names, 0, (var.vpc_subnet_count))
    public_subnets          = [for subnet in range(var.vpc_subnet_count) : cidrsubnet(var.vpc_cidr_block, 8, subnet)]
    map_public_ip_on_launch = var.map_public_ip_on_launch
    enable_dns_hostnames    = var.enable_dns_hostnames

  }
  resource "aws_security_group" "sg" {
    vpc_id = module.vpc.vpc_id
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
      from_port   = 22
      to_port     = 22
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
  
- Now create the  scaling folder (asg) and create custome module for asg.

  First we need to create variable file to use the variable in autoscaling file which is present in asg folder
  
  ```
  variable "instance_type" {
    type        = string
    description = "Type for EC2 Instance"
    default     = "t2.micro"
  }

  variable "instance_count" {
    type        = string
    description = "No of instance"
    default     = 2
  }

  variable "scale_count" {
    type        = string
    description = "No of instance"
    default     = 1
  }

  variable "ami_id" {
    type        = string
    description = "AMI for centod Image"
    default     = "ami-0763cf792771fe1bd"
  }


  variable "Key_name" {
    type        = string
    description = "Key for Centos Machine"
    default     = "test"
  }

  variable "name_prefix" {
    type        = string
    description = "Naming prefix for resources"
    default     = "mywebsite"
  }

  variable "min_size" {
    description = "The minimum size of the autoscaling group"
    type        = number
    default     = null
  }

  variable "max_size" {
    description = "The maximum size of the autoscaling group"
    type        = number
    default     = null
  }

  variable "desired_capacity" {
    description = "The number of Amazon EC2 instances that should be running in the autoscaling group"
    type        = number
    default      = null
  }

  variable "vpc_zone_identifier" {
    description = "zone identifier"
    type        = list(string)
    default     = null
  }

  variable "user_data" {
    description = "The Base64-encoded user data to provide when launching the instance"
    type        = string
    default     = null
  }

  variable "security_groups" {
    description = "A list of security group IDs to associate"
    type        = list(string)
    default     = []
  }

  variable "target_group_arns" {
    description = "A set of `aws_alb_target_group` ARNs, for use with Application or Network Load Balancing"
    type        = list(string)
    default     = []
  }
  ```
  Note: This variables is only for default values and pass this in main scaling file.
  
  Now create the autoscaling file as like below in same asg folder
  
  ```
  resource "aws_launch_template" "scalevm" {
    name            = var.name_prefix
    image_id        = var.ami_id
    instance_type   = var.instance_type
    user_data       = var.user_data
    network_interfaces {
      associate_public_ip_address = true
      security_groups             = var.security_groups
    }
    key_name        =  var.Key_name
  }

  resource "aws_autoscaling_group" "scalevmgroup" {
    count                = var.scale_count
    min_size             = var.min_size
    max_size             = var.max_size
    desired_capacity     = var.desired_capacity
    launch_template {
      id      = aws_launch_template.scalevm.id
      version = "$Latest"
    }
    target_group_arns   = var.target_group_arns
    vpc_zone_identifier  = var.vpc_zone_identifier
  }
  ```
  
  Now create the ouput file for asg module to use the modules output in another modules:
  
  ```
  output "image_id" {
    value       = var.ami_id
  }
  ```
  
- We can use asg module in our main autoscaling file and add the parameters values in variable.

  Variable file is look like below:
  
  ```
  variable "scale_count" {
    type        = string
    description = "No of instance"
    default     = 1
  }
  
  variable "vpc_subnet_count" {
    type        = string
    description = "Subnet count number"
    default     = 2
  }
  
  variable "ami_id" {
    type        = string
    description = "AMI for centod Image"
    default     = "ami-0763cf792771fe1bd"
  }


  variable "Key_name" {
    type        = string
    description = "Key for Centos Machine"
    default     = "test"
  }
 
  variable "instance_type" {
    type        = string
    description = "Type for EC2 Instance"
    default     = "t2.micro"
  }
  ```

  Main autoscaling file is look like below:  

  
  ```   
  module "asg" {
    source = "./asg"

    # Autoscaling group
    name_prefix         = "myasg"
    count               = var.scale_count
    min_size            = 1
    max_size            = 3
    desired_capacity    = 2
    vpc_zone_identifier = [module.vpc.public_subnets[(count.index % var.vpc_subnet_count)]]
    security_groups     = [aws_security_group.sg.id]
    ami_id              = var.ami_id
    instance_type       = var.instance_type
    user_data           = filebase64("deploy.sh")
    target_group_arns   = [module.alb.target_group_arns]
 
  }
  ```
  
  
