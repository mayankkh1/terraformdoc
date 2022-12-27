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

#### Step-4: 
  
  
  


