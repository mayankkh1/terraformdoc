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
    
    
  

 

  
  
  


