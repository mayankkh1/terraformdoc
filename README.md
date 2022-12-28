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
 

  
 

  
  
  


