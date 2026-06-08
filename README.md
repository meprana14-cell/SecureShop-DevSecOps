# 🚀 Jenkins Pipeline for SecureShop Java App with Maven, Git, Trivy, SonarQube, Kubernetes, Nexus, ECR, Amazon EKS
*A Complete Step-by-Step Technical Implementation Guide (with VPC & EC2 Setup)*

---

## 🧭 Table of Contents
1. [Overview](#overview)
2. [Architecture Overview](#architecture-overview)
3. [Tools and Technologies](#tools-and-technologies)
4. [Create a Custom VPC](#create-a-custom-vpc)
5. [Launch a Jenkins Server (Ubuntu EC2)](#launch-a-jenkins-server-ubuntu-ec2)
6. [Login to Jenkins Server](#login-to-jenkins-server)
7. [Launch a Nexus Server (Ubuntu EC2)](#launch-a-nexus-server-ubuntu-ec2)
8. [Launch a SonarQube Server (Ubuntu EC2)](#launch-a-sonarqube-server-ubuntu-ec2)
9. [Create an EKS Cluster](#create-an-eks-cluster)


---

## 🧩 Overview

This tutorial walks you through implementation of a **complete CI/CD pipeline using Jenkins for a Java application.**
You’ll learn how to automate the entire **build, testing, security scanning, code quality analysis, containerization, and deployment process.**

## 🏗️ Architecture Overview

![architecture](images/architecture.jpg)

---

## 🧰 Tools and Technologies

Below are the tools and services used.

| Tool | Description |
|------|--------------|
| **Jenkins:** | Building CI/CD Pipeline. |
| **Maven:** | Build automation tool for Java projects |
| **Git:** | Distributed Version Control System |
| **Trivy:** | Vulnerability scanner for container images |
| **OWASP Dependency Check:** | Identifies project dependencies and checks for known, publicly disclosed vulnerabilities |
| **SonarQube:** | Continuous code quality and security analysis |
| **GitHub:** | Cloud-based hosting service for Git repositories |
| **Nexus Artifact Repository:** | Stores and distributes artifacts (e.g., libraries, dependencies) |
| **Amazon Elastic Container Registry (ECR):** | Fully-managed Docker container registry |
| **Amazon EKS:** | Container Orchestration Service (Elastic Kubernetes Service) |
| **Kubernetes:** | Container orchestration platform for deployment |

---

## 💻 Local Machine
## 💻 Install AWS CLI on Windows 

```bash
# 📦 Step 1: Download the latest AWS CLI v2 Installer (64-bit)
Invoke-WebRequest -Uri "https://awscli.amazonaws.com/AWSCLIV2.msi" -OutFile "AWSCLIV2.msi"

# 🧩 Step 2: Run the installer
Start-Process msiexec.exe -Wait -ArgumentList '/i AWSCLIV2.msi /qn'

# 🧹 Step 3: Clean up the installer
Remove-Item "AWSCLIV2.msi"

# ✅ Step 4: Verify installation
aws --version
```

## 🍎 Install AWS CLI on macOS
```bash
# 📦 Step 1: Download and install AWS CLI v2
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"

# 🧩 Step 2: Run the installer
sudo installer -pkg AWSCLIV2.pkg -target /

# 🧹 Step 3: Clean up the installer
rm AWSCLIV2.pkg

# ✅ Step 4: Verify installation
aws --version
```
## 🐧 Install AWS CLI on Linux (Ubuntu/Debian) 
```bash
# 📦 Step 1: Download AWS CLI bundle
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 🧩 Step 2: Install unzip if not available
sudo apt install unzip -y

# 🧩 Step 3: Unzip and install
unzip awscliv2.zip
sudo ./aws/install

# 🧹 Step 4: Clean up
rm -rf awscliv2.zip aws/

# ✅ Step 5: Verify installation
aws --version
```

## 🪐 Create a Custom VPC

### 💻 Windows Powershell
```bash

# Variables
$REGION = "ap-south-2"
$VPC_CIDR = "10.0.0.0/16"
$VPC_NAME = "Jenkins-EKS"

# Create VPC
$VPC_ID = (aws ec2 create-vpc `
  --cidr-block $VPC_CIDR `
  --region $REGION `
  --query "Vpc.VpcId" `
  --output text)

# Tag VPC
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=$VPC_NAME

Write-Host "✅ VPC Created: $VPC_ID"


# Public Subnets

# ✅ Public Subnet 1
$PUB_SUBNET1 = (aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block 10.0.1.0/24 `
  --availability-zone ap-south-2a `
  --query "Subnet.SubnetId" `
  --output text)
  aws ec2 create-tags `
  --resources $PUB_SUBNET1 `
  --tags Key=Name,Value=PUB_SUBNET1 `
         Key=kubernetes.io/role/elb,Value=1 `
         Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared
  aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET1 --map-public-ip-on-launch

# ✅ Public Subnet 2
$PUB_SUBNET2 = (aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block 10.0.2.0/24 `
  --availability-zone ap-south-2b `
  --query "Subnet.SubnetId" `
  --output text)
  aws ec2 create-tags `
  --resources $PUB_SUBNET2 `
  --tags Key=Name,Value=PUB_SUBNET2 `
         Key=kubernetes.io/role/elb,Value=1 `
         Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared
  aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET2 --map-public-ip-on-launch
         
# Private Subnets
# ✅ Private Subnet 1
$PRI_SUBNET1 = (aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block 10.0.3.0/24 `
  --availability-zone ap-south-2a `
  --query "Subnet.SubnetId" `
  --output text)
  aws ec2 create-tags `
  --resources $PRI_SUBNET1 `
  --tags Key=Name,Value=PRI_SUBNET1 `
         Key=kubernetes.io/role/internal-elb,Value=1 `
         Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared

# ✅ Private Subnet 2
$PRI_SUBNET2 = (aws ec2 create-subnet `
  --vpc-id $VPC_ID `
  --cidr-block 10.0.4.0/24 `
  --availability-zone ap-south-2b `
  --query "Subnet.SubnetId" `
  --output text)
  aws ec2 create-tags `
  --resources $PRI_SUBNET2 `
  --tags Key=Name,Value=PRI_SUBNET2 `
         Key=kubernetes.io/role/internal-elb,Value=1 `
         Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared
  
Write-Host "✅ Public Subnets: $PUB_SUBNET1, $PUB_SUBNET2"
Write-Host "✅ Private Subnets: $PRI_SUBNET1, $PRI_SUBNET2"


# Internet Gateway and Route Tables
$IGW_ID = (aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 create-tags --resources $IGW_ID --tags Key=Name,Value=${VPC_NAME}-igw

$PUB_RT_ID = (aws ec2 create-route-table --vpc-id $VPC_ID --query "RouteTable.RouteTableId" --output text)
aws ec2 create-route --route-table-id $PUB_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET1
aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET2

Write-Host "✅ Internet Gateway: $IGW_ID"
Write-Host "✅ Public Route Table: $PUB_RT_ID"


# Security Group for Jenkins Server
$SG_ID_1 = (aws ec2 create-security-group `
  --group-name jenkins-sg `
  --description "Jenkins Server" `
  --vpc-id $VPC_ID `
  --query "GroupId" `
  --output text)

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_1 `
  --protocol tcp `
  --port 22 `
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_1 `
  --protocol tcp `
  --port 8080 `
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_1 `
  --protocol tcp `
  --port 80 `
  --cidr 0.0.0.0/0
  
  Write-Host "✅ Security Group Created: $SG_ID_1 (Jenkins Server)"
 
 # Security Group for Nexus Server
 $SG_ID_2 = (aws ec2 create-security-group `
  --group-name nexus-sg `
  --description "Nexus Server" `
  --vpc-id $VPC_ID `
  --query "GroupId" `
  --output text)

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_2 `
  --protocol tcp `
  --port 22 `
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_2 `
  --protocol tcp `
  --port 8081 `
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_2 `
  --protocol tcp `
  --port 80 `
  --cidr 0.0.0.0/0

Write-Host "✅ Security Group Created: $SG_ID_2 (Nexus Server)"

# Security Group for SonarQube Server
$SG_ID_3 = (aws ec2 create-security-group `
  --group-name sonarqube-sg `
  --description "SonarQube Server" `
  --vpc-id $VPC_ID `
  --query "GroupId" `
  --output text)

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_3 `
  --protocol tcp `
  --port 22 `
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_3 `
  --protocol tcp `
  --port 9000 `
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress `
  --group-id $SG_ID_3 `
  --protocol tcp `
  --port 80 `
  --cidr 0.0.0.0/0

Write-Host "✅ Security Group Created: $SG_ID_3 (SonarQube Server)"

  ```

### 🐧 Linux / 🍎 macOS

```bash

# Variables
REGION="ap-south-2"
VPC_CIDR="10.0.0.0/16"
VPC_NAME="Jenkins-EKS"
ECR_APP_NAME="secureshop"
AWS_ACCOUNT_ID="822654906952"

# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block $VPC_CIDR \
  --region $REGION \
  --query 'Vpc.VpcId' \
  --output text)

# Tag VPC
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=$VPC_NAME

echo "✅ VPC Created: $VPC_ID"


# Public Subnets
# ✅ Public Subnet 1

PUB_SUBNET1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-2a \
  --query 'Subnet.SubnetId' \
  --output text)
  
  aws ec2 create-tags \
  --resources $PUB_SUBNET1 \
  --tags 'Key=Name,Value=PUB_SUBNET1' \
         'Key=kubernetes.io/role/elb,Value=1' \
         'Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared'
  aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET1 --map-public-ip-on-launch

# ✅ Public Subnet 2
PUB_SUBNET2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-2b \
  --query 'Subnet.SubnetId' \
  --output text)
  
  aws ec2 create-tags \
  --resources $PUB_SUBNET2 \
  --tags 'Key=Name,Value=PUB_SUBNET2' \
         'Key=kubernetes.io/role/elb,Value=1' \
         'Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared'
  aws ec2 modify-subnet-attribute --subnet-id $PUB_SUBNET2 --map-public-ip-on-launch
# Private Subnets
# ✅ Private Subnet 1
PRI_SUBNET1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-2a \
  --query 'Subnet.SubnetId' \
  --output text)
  
  aws ec2 create-tags \
  --resources $PRI_SUBNET1 \
  --tags 'Key=Name,Value=PRI_SUBNET1' \
         'Key=kubernetes.io/role/internal-elb,Value=1' \
         'Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared'


# ✅ Private Subnet 2
PRI_SUBNET2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 \
  --availability-zone ap-south-2b \
  --query 'Subnet.SubnetId' \
  --output text)
  
  aws ec2 create-tags \
  --resources $PRI_SUBNET2 \
  --tags 'Key=Name,Value=PRI_SUBNET2' \
         'Key=kubernetes.io/role/internal-elb,Value=1' \
         'Key=kubernetes.io/cluster/jenkins-secureapp,Value=shared'

echo "✅ Public Subnets: $PUB_SUBNET1, $PUB_SUBNET2"
echo "✅ Private Subnets: $PRI_SUBNET1, $PRI_SUBNET2"


# Internet Gateway and Route Tables
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 create-tags --resources $IGW_ID --tags Key=Name,Value=${VPC_NAME}-igw

PUB_RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET1
aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET2

echo "✅ Internet Gateway: $IGW_ID"
echo "✅ Public Route Table: $PUB_RT_ID"


# Security Group for Jenkins Server
SG_ID_1=$(aws ec2 create-security-group \
  --group-name jenkins-sg \
  --description "Jenkins Server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_1 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_1 \
  --protocol tcp \
  --port 8080 \
  --cidr 0.0.0.0/0
  
 aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_1 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
echo "✅ Security Group Created: $SG_ID_1"

# Security Group for Nexus Server
SG_ID_2=$(aws ec2 create-security-group \
  --group-name nexus-sg \
  --description "Nexus Server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_2 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_2 \
  --protocol tcp \
  --port 8081 \
  --cidr 0.0.0.0/0
  
 aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_2 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
echo "✅ Security Group Created: $SG_ID_1"

# Security Group for SonarQube Server
SG_ID_3=$(aws ec2 create-security-group \
  --group-name sonarqube-sg \
  --description "SonarQube Server" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_3 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_3 \
  --protocol tcp \
  --port 9000 \
  --cidr 0.0.0.0/0
  
 aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID_3 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
echo "✅ Security Group Created: $SG_ID_1"

```

---

> ### Before Getting Started let us Create 3 EC2 Ubuntu Instances With Proper Security Group For 3 Different Servers (Jenkins, Sonarqube, Nexus). All Of Them With 4gb Ram And 2vcpu. (T3.Medium)

## 💻 Launch a Jenkins Server (Ubuntu EC2)

```bash

# Define variables
$AMI_ID = "ami-0ecb62995f68bb549"
$KEY_NAME = "jenkins-key"
```

### 💻 Windows Powershell

```bash

# Create and save a new PEM key
aws ec2 create-key-pair `
  --key-name $KEY_NAME `
  --query "KeyMaterial" `
  --output text | Out-File -FilePath "$KEY_NAME.pem" -Encoding ascii

# Restrict file permissions (Windows equivalent of chmod 400)
icacls "$KEY_NAME.pem" /inheritance:r /grant:r "$($env:USERNAME):R"

# Verify file
Get-Item "$KEY_NAME.pem"

# Launch EC2 Instance for Jenkins Server
$JENKINS_ID = (aws ec2 run-instances `
  --image-id $AMI_ID `
  --count 1 `
  --instance-type t3.medium `
  --key-name $KEY_NAME `
  --security-group-ids $SG_ID_1 `
  --subnet-id $PUB_SUBNET1 `
  --associate-public-ip-address `
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Jenkins-Server}]" `
  --query "Instances[0].InstanceId" `
  --region ap-south-2 `
  --output text)

Write-Host " Jenkins Server Instance ID: $JENKINS_ID"

# Launch EC2 Instance for Nexus Server
$NEXUS_ID = (aws ec2 run-instances `
  --image-id $AMI_ID `
  --count 1 `
  --instance-type t3.medium `
  --key-name $KEY_NAME `
  --security-group-ids $SG_ID_2 `
  --subnet-id $PUB_SUBNET1 `
  --associate-public-ip-address `
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Nexus-Server}]" `
  --query "Instances[0].InstanceId" `
  --region ap-south-2 `
  --output text)

Write-Host " Nexus Server Instance ID: $NEXUS_ID"

# Launch EC2 Instance for SonarQube Server
$SONARQUBE_ID = (aws ec2 run-instances `
  --image-id $AMI_ID `
  --count 1 `
  --instance-type t3.medium `
  --key-name $KEY_NAME `
  --security-group-ids $SG_ID_3 `
  --subnet-id $PUB_SUBNET1 `
  --associate-public-ip-address `
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=SonarQube-Server}]" `
  --query "Instances[0].InstanceId" `
  --region ap-south-2 `
  --output text)

Write-Host " SonarQube Server Instance ID: $SONARQUBE_ID"
```

### 🐧 Linux / 🧠 macOS

```bash

# Create and save a new PEM key

aws ec2 create-key-pair \
  --key-name $KEY_NAME \
  --query "KeyMaterial" \
  --output text > ${KEY_NAME}.pem

# Restrict permissions
chmod 400 ${KEY_NAME}.pem

# Verify
ls -l ${KEY_NAME}.pem

# Launch EC2 Instance for Jenkins Server
JENKINS_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type t3.medium \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID_1 \
  --subnet-id $PUB_SUBNET1 \
  --associate-public-ip-address \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Jenkins-Server}]" \
  --query "Instances[0].InstanceId" \
  --region ap-south-2 \
  --output text)

Write-Host " Jenkins Server Instance ID: $JENKINS_ID"

# Launch EC2 Instance for Nexus Server
NEXUS_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type t3.medium \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID_2 \
  --subnet-id $PUB_SUBNET1 \
  --associate-public-ip-address \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Nexus-Server}]" \
  --query "Instances[0].InstanceId" \
  --region ap-south-2 \
  --output text)

Write-Host " Nexus Server Instance ID: $NEXUS_ID"

# Launch EC2 Instance for SonarQube Server
SONARQUBE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type t3.medium \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID_3 \
  --subnet-id $PUB_SUBNET1 \
  --associate-public-ip-address \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=SonarQube-Server}]" \
  --query "Instances[0].InstanceId" \
  --region ap-south-2 \
  --output text)

Write-Host " SonarQube Server Instance ID: $SONARQUBE_ID"
```

---

## 🔍 Verify Resources

```bash
aws ec2 describe-vpcs --vpc-ids $VPC_ID
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID"
aws ec2 describe-instances --instance-ids $JENKINS_ID
aws ec2 describe-instances --instance-ids $NEXUS_ID
aws ec2 describe-instances --instance-ids $SONARQUBE_ID
```

---

## 🔐 Login to Jenkins

After creating your Jenkins EC2 instance, you can securely connect to it using the SSH key pair you generated earlier.

---

### 💻 Windows PowerShell

```powershell
# Variables
$JENKINS_IP = "YOUR_JENKINS_PUBLIC_IP"
$KEY_NAME = "Jenkins-key"

# Change directory to where the PEM file is stored
cd "C:\path\to\your\pem\file"

# Connect to the Jenkins Server
ssh -i "$KEY_NAME.pem" ubuntu@$JENKINS_IP
```
---

### 🐧 Linux / 🧠 macOS

```powershell
# Variables
$JENKINS_IP = "YOUR_JENKINS_PUBLIC_IP"
$KEY_NAME = "Jenkins-key"

# Navigate to PEM file location
cd ~/Downloads   # or the directory where you saved the PEM file

# Ensure proper permissions
chmod 400 ${KEY_NAME}.pem

# Connect to the Jenkins Server
ssh -i ${KEY_NAME}.pem ubuntu@${JENKINS_IP}
```
---
## ⚙️ Prerequisites

### 🐧 Ubuntu / Linux
### 🧭 Basic System Setup
```bash
# Set Hostname
sudo hostname Jenkins-Server
sudo nano /etc/hostname - For permanent change

# Update System Packages
sudo apt update -y
sudo apt upgrade -y
```
### ⚙️ Install AWS CLI on Linux 
```bash
Run the following commands to install the latest AWS CLI v2:
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
``` 
### 🧩 Install eksctl on Linux
```bash
Run the following command to download and install the latest version of **eksctl**:
curl -sL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | sudo tar xz -C /usr/local/bin
eksctl version
```
### ⚙️ Install kubectl on Linux
```bash
Run the following commands to download and install **kubectl** for Amazon EKS:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" 
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```
### 🐳 Install Docker on Linux
```bash
# Update and install dependencies
sudo apt update -y
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker’s official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine, CLI, and Compose
sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start and enable Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify Docker
docker --version
sudo docker run hello-world
```
### 🧱 Install Jenkins on Linux
```bash
# Install Java (required by Jenkins)
sudo apt install -y fontconfig openjdk-21-jre

# Add Jenkins repository and key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update -y
sudo apt install -y jenkins

# Start and enable Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
### 🧩 Install Maven
```bash
# Install Apache Maven
sudo apt install maven -y

# Verify Maven installation
mvn --version
```
### 👤 Allow Jenkins to Use Docker
```bash
# Add Jenkins user to Docker group
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins
```
### 🔑 Get Jenkins Unlock Password
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
*Then open your Jenkins dashboard:*
```bash
http://<your-ec2-public-ip>:8080
```
*Use the above password to unlock Jenkins and install suggested plugins.*


## 🔐 Login to Nexus Server
After creating your Nexus EC2 instance, you can securely connect to it using the SSH key pair you generated earlier.

First exit from the Jenkins Server and then login again to the Nexus Server

---

```powershell
### 💻 Windows PowerShell
# Variables
$NEXUS_IP = "YOUR_NEXUS_PUBLIC_IP"
$KEY_NAME = "Jenkins-key"

# Change directory to where the PEM file is stored
cd "C:\path\to\your\pem\file"

# Connect to the Bastion Host
ssh -i "$KEY_NAME.pem" ubuntu@$NEXUS_IP
```
---

### 🐧 Linux / 🧠 macOS

```bash
# Variables
$NEXUS_IP = "YOUR_NEXUS_PUBLIC_IP"
$KEY_NAME = "Jenkins-key"

# Navigate to PEM file location
cd ~/Downloads   # or the directory where you saved the PEM file

# Ensure proper permissions
chmod 400 ${KEY_NAME}.pem

# Connect to the Bastion Host
ssh -i ${KEY_NAME}.pem ubuntu@${NEXUS_IP}
```
---
## ⚙️ Prerequisites

### 🐧 Ubuntu / Linux
### 🧭 Basic System Setup
```bash
# Set Hostname
sudo hostname Nexus-Server
nano /etc/hostname - For permanent change

# Update System Packages
sudo apt update -y
sudo apt upgrade -y
```
### 🧩 Install Nexus Repository Manager

```bash
# Update system
sudo apt update -y
sudo apt install -y openjdk-21-jre wget tar

# Create Nexus user
sudo useradd -M -d /opt/nexus -s /bin/bash nexus
sudo mkdir /opt/nexus
sudo mkdir -p /opt/sonatype-work/nexus3
ls -ld /opt/nexus /opt/sonatype-work
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work

# Download Nexus 3
cd /tmp
wget https://download.sonatype.com/nexus/3/nexus-3.85.0-03-linux-x86_64.tar.gz
mv nexus-3.85.0-03-linux-x86_64.tar.gz latest.tar.gz

# Extract and move files
sudo tar -xvzf latest.tar.gz -C /opt/nexus --strip-components=1
sudo chown -R nexus:nexus /opt/nexus

# Set environment variables
echo 'run_as_user="nexus"' | sudo tee /opt/nexus/bin/nexus.rc

# Create a systemd service for Nexus
sudo tee /etc/systemd/system/nexus.service > /dev/null <<EOF
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
User=nexus
Group=nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-abort

[Install]
WantedBy=multi-user.target
EOF

# Reload, enable, and start Nexus
sudo systemctl daemon-reload
sudo systemctl enable nexus
sudo systemctl start nexus

# Check Nexus status
sudo systemctl status nexus
```
### 🌐 Access Nexus
```bash
Open your browser:
    http://<nexus-ec2-public-ip>:8081
    
Default credentials:
    Username: admin
    Password: (found in /opt/sonatype-work/nexus3/admin.password)

To get the password:
    sudo cat /opt/sonatype-work/nexus3/admin.password
```

## 🔐 Login to SonarQube Server

After creating your SonarQube EC2 instance, you can securely connect to it using the SSH key pair you generated earlier.

First exit from the Nexus Server and then login again to the SonarQube Server

---

### 💻 Windows PowerShell

```powershell
# Variables
$SONARQUBE_IP = "YOUR_SONARQUBE_PUBLIC_IP"
$KEY_NAME = "Jenkins-key"

# Change directory to where the PEM file is stored
cd "C:\path\to\your\pem\file"

# Connect to the SOnarQube Host
ssh -i "$KEY_NAME.pem" ubuntu@$SONARQUBE_IP
```
---

### 🐧 Linux / 🧠 macOS

```bash
# Variables
$SONARQUBE_IP = "YOUR_SONARQUBE_PUBLIC_IP"
$KEY_NAME = "Jenkins-key"

# Navigate to PEM file location
cd ~/Downloads   # or the directory where you saved the PEM file

# Ensure proper permissions
chmod 400 ${KEY_NAME}.pem

# Connect to the SonarQube Host
ssh -i ${KEY_NAME}.pem ubuntu@${SONARQUBE_IP}
```
---
## ⚙️ Prerequisites

### 🐧 Ubuntu / Linux
### 🧭 Basic System Setup
```bash
# Set Hostname
sudo hostname SonarQube-Server
sudo nano /etc/hostname - For permanent change

# Update System Packages
sudo apt update -y
sudo apt upgrade -y
```
### SonarQube with PostgreSQL (Persistent Docker Setup)
### 🐳  Install Docker on Ubuntu
```bash
# Update system
sudo apt update -y
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose
sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable and start Docker
sudo systemctl enable docker
sudo systemctl start docker

# Verify Docker installation
docker --version
sudo docker run hello-world
```

### Create a folder for Sonar Data
```bash
sudo mkdir /opt/sonar-data/
sudo chmod 755 /opt/sonar-data/
```

### 🌐 Create Docker Network
```bash
docker network create sonar-network

# Create persistent volumes for database and sonar data
docker volume create sonar_data
docker volume create sonar_extensions
docker volume create sonar_logs
docker volume create sonar_db
```
### 🐘 Run PostgreSQL (Persistent Database Container)
```bash
docker run -d --name sonar-db --network sonar-network \
  -e POSTGRES_USER=sonar \
  -e POSTGRES_PASSWORD=sonar \
  -e POSTGRES_DB=sonar \
  -v /opt/sonar-data/postgres:/var/lib/postgresql/data \
  postgres:15
```
✅ Persistent PostgreSQL database for SonarQube.
### 🚀 Run SonarQube (Persistent App Container)
```bash
docker run -d --name sonar \
  --restart unless-stopped \
  -p 9000:9000 \
  --user root \
  --network sonar-network \
  -e SONAR_JDBC_URL=jdbc:postgresql://sonar-db:5432/sonar \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=sonar \
  -v /opt/sonar-data/sonarqube_conf:/opt/sonarqube/conf \
  -v /opt/sonar-data/sonarqube_data:/opt/sonarqube/data \
  -v /opt/sonar-data/sonarqube_logs:/opt/sonarqube/logs \
  -v /opt/sonar-data/sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:lts-community
```
✅ SonarQube runs persistently and connects automatically to the PostgreSQL database.
### 🔍 Verify Containers
```bash
docker ps

CONTAINER ID   IMAGE                      PORTS                  NAMES
a1b2c3d4e5f6   sonarqube:lts-community    0.0.0.0:9000->9000/tcp sonar
f1e2d3c4b5a6   postgres:15                5432/tcp               sonar-db
```
### 🌐 Access SonarQube
```bash
http://<your-ec2-public-ip>:9000

Username: admin
Password: admin
```
After login:
Change the default password.
Go to *Administration → Security → Users* → Generate a token for Jenkins.
### 🧰 Manage Containers
```bash
# View logs
docker logs -f sonar
docker logs -f sonar-db

# Restart
docker restart sonar sonar-db

# Stop
docker stop sonar sonar-db

# Remove (only if needed)
docker rm -f sonar sonar-db
```

## ☸️ Create an EKS Cluster

### Creating AWS EKS Cluster using eksctl
*Before creating a server login to the Jenkins server where the tools are installed*
```bash

# Variables
REGION="ap-south-2"
VPC_NAME="Jenkins-EKS"
ECR_APP_NAME="secureshop"

# Get VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=${VPC_NAME}" \
  --region ${REGION} \
  --query "Vpcs[0].VpcId" \
  --output text)

echo "✅ VPC ID: ${VPC_ID}"

# List all subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --region $REGION \
  --query "Subnets[*].{SubnetId:SubnetId, AZ:AvailabilityZone, CIDR:CidrBlock, Tags:Tags}" \
  --output table
 # Create a key for cluster
  KEY_NAME="jenkins-cluster-key"
  
  aws ec2 create-key-pair \
  --key-name $KEY_NAME \   
  --query "KeyMaterial" \
  --output text > ${KEY_NAME}.pem
  
 ```
 
 
 ### Create the YAML file and save it as jenkins-eks-cluster.yaml
 ```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: jenkins-secureapp
  region: ap-south-2 # Must match your subnet AZs
  version: "1.33"   # Latest stable Kubernetes version

vpc:
  id: "vpc-0b088e420a893f9dd"   # Replace with your VPC_ID
  cidr: "10.0.0.0/16"   # Replace with your CIDR Value
  subnets:
    public:
      ap-south-2a: { id: subnet-0d52d70f8e9f222ea }   # Replace with your PUB_SUBNET1
      ap-south-2b: { id: subnet-07aba60194037fad4 }   # Replace with your PUB_SUBNET2
    private:
      ap-south-2a: { id: subnet-0d0014d3a98b1d5ce }   # Replace with your PRI_SUBNET1
      ap-south-2b: { id: subnet-0c8f6c6bc5082ec87 }   # Replace with your PRI_SUBNET2
      
managedNodeGroups:
  - name: worker-nodes
    instanceType: t3.medium
    minSize: 1
    maxSize: 2
    spot: false
    ssh: 
      allow: true
      publicKeyName: jenkins-cluster-key
    labels: { role: jenkins }
    tags:
      Name: worker-nodes

addons:
  - name: aws-ebs-csi-driver
    version: latest
  - name: eks-pod-identity-agent
    version: latest
  - name: vpc-cni
    version: latest
  - name: kube-proxy
    version: latest
  - name: coredns
    version: latest

addonsConfig:    
   autoApplyPodIdentityAssociations: true
```
### 🧱 Save your cluster config file
```bash
jenkins-eks-cluster.yaml
```

### 🧾 Validate your config
```bash
eksctl utils validate cluster --config-file jenkins-eks-cluster.yaml --region ap-south-2
```

### 🚀 Create the cluster
```bash
eksctl create cluster -f jenkins-eks-cluster.yaml
```

### 🧩 Verify the cluster
```bash
aws eks --region ap-south-2 update-kubeconfig --name jenkins-secureapp
kubectl get nodes
kubectl get pods -A
```

###🧭 Create Namespace

```bash
kubectl create namespace secureapp
kubectl config set-context --current --namespace=secureapp
```

## ☸️ Create the ECR Repository
```bash
aws ecr create-repository \
  --repository-name $ECR_APP_NAME \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --region $AWS_REGION
```
### 🧩  Verify the repository
```bash
aws ecr describe-repositories --repository-names $ECR_APP_NAME --region $AWS_REGION
```
### 🧰  Authenticate Docker to ECR
```bash
aws ecr get-login-password --region $AWS_REGION \
| docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

## ⚙️ Configure Jenkins

### 🔑 Get Jenkins Unlock Password
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

Then open your Jenkins dashboard:
http://<your-ec2-public-ip>:8080
```
### ⚙️ Recommended Jenkins Plugins
*Plugins Installation: Go to Manage Jenkins → Plugins → Available Plugins, then install:*
```bash
1. SonarQube Scanner Plugin: Integrates Jenkins with SonarQube for code analysis.
2. Nexus Artifact Uploader: Provides integration with Nexus Repository Manager for artifact management.
3. OWASP Dependency-Check Plugin: Scans dependencies for known vulnerabilities using OWASP CVE data.
4. Config File Provider: Supplies configuration files (e.g., settings.xml for Maven/Nexus) inside pipelines.
5. Eclipse Temurin Installer: Provides an installer for different JDK versions.
6. Pipeline Maven Integration: Enables Maven build and artifact management within Jenkins pipelines.
7. Kubernetes Plugin: Deploys Jenkins workloads (agents or jobs) to Kubernetes pods.
8. Kubernetes CLI Plugin: Provides kubectl access directly inside Jenkins pipelines.
9. Amazon EKS Plugin: Adds native EKS integration for cluster management and deployments.
10. Amazon ECR Plugin: Enables Docker image push/pull with AWS Elastic Container Registry.
11. AWS Credentials Plugin: Manages AWS access keys and roles within Jenkins.
12. Docker Plugin: Builds, tags, and pushes Docker images during CI/CD pipelines.
13. Git & GitHub Plugins: Integrate Jenkins with Git/GitHub for SCM, commits, and webhooks.
```
### Grant Jenkins User Access to AWS CLI & EKS (kubectl)
🧩 Switch to Root
*If not already:
```bash
sudo -i
```
⚙️ Create AWS Configuration for Jenkins User
*Move your existing AWS credentials and config to Jenkins’ home:*
```bash
sudo mkdir -p /var/lib/jenkins/.aws
sudo cp -r ~/.aws/* /var/lib/jenkins/.aws/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.aws
sudo chmod 600 /var/lib/jenkins/.aws/credentials
```
✅ This gives Jenkins the same AWS permissions you’ve configured for your root or ubuntu user.

☸️ Configure kubectl for Jenkins (EKS Access)
*Run the following as root (or ubuntu) to create kubeconfig for Jenkins:*
```bash
aws eks --region ap-south-2 update-kubeconfig --name jenkins-secureapp
```
This creates /root/.kube/config.
*Now copy it to the Jenkins user home:*
```bash
sudo mkdir -p /var/lib/jenkins/.kube
sudo cp /root/.kube/config /var/lib/jenkins/.kube/config
sudo chown -R jenkins:jenkins /var/lib/jenkins/.kube
```
🧰 Verify Permissions
*Switch to Jenkins user and test:*
```bash
sudo -u jenkins -i
aws sts get-caller-identity
kubectl get nodes
kubectl get pods -A
        
```
✅ Expected Output:
The aws sts get-caller-identity shows your AWS Account ID (e.g., 822654906952)
The kubectl get nodes lists EKS worker nodes.

🧱 5️⃣ Ensure Jenkins Has PATH to AWS & kubectl
Sometimes Jenkins doesn’t inherit the full environment.

```bash
sudo tee /etc/profile.d/jenkins-path.sh > /dev/null <<'EOF'
export PATH=$PATH:/usr/local/bin
EOF
sudo chmod +x /etc/profile.d/jenkins-path.sh
```
Then restart Jenkins:
```bash
sudo systemctl restart jenkins
```
## Configuration Screenshots in Jenkins

![newitems](images/newitems.jpg)
![Project](images/Project.jpg)
![configure](images/configure.jpg)
![credentials 1](images/credentials-1.jpg)
![credentials](images/credentials.jpg)
![settings](images/settings.jpg)
![tools](images/tools.jpg)
![global config](images/global-config.jpg)
![maven global settings](images/maven-gloval-settings.jpg)
![maven settings](images/maven-settings.jpg)
![DC install](images/DC-install.jpg)
![git location](images/git-location.jpg)
![jenkins url](images/jenkins-url.jpg)
![reload](images/reload.jpg)
![sonarscanner](images/sonarscanner.jpg)
![sonarqube details](images/sonarqube-details.jpg)
![token create](images/token-create.jpg)
![gitwebhook](images/gitwebhook.jpg)
![webhook 1](images/webhook-1.jpg)
