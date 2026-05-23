# terraform-eks-jenkin
terraform version
terraform init
terraform plan
terraform apply
terraform destroy

### Pre-requisites to implement this project:

AWSCLI Install:

  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  sudo apt install unzip
  unzip awscliv2.zip
  sudo ./aws/install

  
Configure AWSCLI:

aws --version
aws configure

IAM role console user > security > access key
> This project will be deployed on United States (Oregon) - us-west-2 but deploy your preffered region.

- <b>Create 1 Master machine on AWS with 2CPU, 8GB of RAM (t2.large) and 30 GB of storage manually or using Terraform.</b>
#
- <b>Open all the PORTs in security group of master machine</b> <br />
  | Port Range    | Source    | Description           |
  | ------------- | --------- | --------------------- |
  | 22            | 0.0.0.0/0 | SSH                   |
  | 443           | 0.0.0.0/0 | HTTPS                 |
  | 30000 - 32767 | 0.0.0.0/0 | NodePort services     |
  | 25            | 0.0.0.0/0 | SMTP                  |
  | 3000 - 10000  | 0.0.0.0/0 | Registered Ports      |
  | 6443          | 0.0.0.0/0 | Kubernetes API server |
  | 80            | 0.0.0.0/0 | HTTP                  |
  | 465           | 0.0.0.0/0 | SMTPS                 |

  Master Node Install :
  - Configure AWSCLI:
```bash
aws --version
aws configure

Create an AWS USER mega-admin and Attach Policy > Administrator Access
  - Create Access_key and Secret_key
```

## Create an AWS Role Called mega-ec2-role and attach it to Master machine
  - Create Role:
    AWS IAM > roles > Create role > AWS Service > Use case (ec2) > Next > AdministratorAccess> Role name (mega-ec2-role) > Create Role
  - Add Role to EC2 Master machine:
    Master machine > Actions > Security > Modify IAM Role > Select mega-ec2-role > Update IAM Role
## Create EKS Cluster on AWS (Master machine)
### Install **kubectl** and **eksctl** (Master machine)
  - Install **kubectl** 
  ```bash
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin
  kubectl version --short --client
  ```

  - Install **eksctl** 
  ```bash
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  eksctl version
  ```
  
  - <b>Create EKS Cluster (Master machine - it might take 15 to 20 minutes)</b>
  ```bash
  eksctl create cluster --name=mega \
                      --region=us-west-2 \
                      --version=1.30 \
                      --without-nodegroup
  ```
  - <b> Check clusters
    ```bash
      eksctl get clusters -o json
      Go to AWS CloudFormation, you should see ***eksctl-mega-cluster***
    ```
  - <b>Associate IAM Open ID Connect provider (OIDC Provider) on Master machine</b>
  ```bash
  eksctl utils associate-iam-oidc-provider \
    --region us-west-2 \
    --cluster mega \
    --approve
  ```
  - <b>Create Nodegroup on Master machine, it might take 15 to 20 minutes</b>
  - <i>It will create 2 nodes ec2 machines</i>
  ```bash
  eksctl create nodegroup --cluster=mega \
                       --region=us-west-2 \
                       --name=mega \
                       --node-type=t2.large \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=2 \
                       --node-volume-size=29 \
                       --ssh-access \
                       --ssh-public-key=eks-nodegroup-key 
  ```
- <i>OR, It will create 1 node ec2 machine</i>
  ```bash
  eksctl create nodegroup --cluster=mega \
                       --region=us-west-2 \
                       --name=mega \
                       --node-type=t2.large \
                       --nodes=1 \
                       --nodes-min=1 \
                       --nodes-max=1 \
                       --node-volume-size=29 \
                       --ssh-access \
                       --ssh-public-key=eks-nodegroup-key 
  ```

>  Make sure the ssh-public-key "eks-nodegroup-key" is available in your aws account

- <b>Check if Nodegroup is created</b>
```bash
  kubectl get nodes -n mega
  Also, go to AWS EC2, you should see your desired node machines got created
```
##########################################################################################################
jenkins install......................................
1. Install Java
Jenkins requires Java to run. Ensure you have the required version (Java 21 or later) installed:

sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version


3. Add the Jenkins Repository
Follow these commands exactly to add the official repository and the GPG key, which allows apt to verify the package authenticity:

# Create the keyrings directory if it doesn't exist
sudo mkdir -p /etc/apt/keyrings

# Download the Jenkins GPG key
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

# Add the repository to your system
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
3. Install Jenkins
sudo apt update
sudo apt install -y jenkins

4. Start and Verify Jenkins

sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins


6. Access the Web Interface
sudo ufw allow 8080

2. **Unlock Jenkins:** Retrieve your initial administrative password by running:
   
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword

###################################################################################################
docker install:

sudo apt update

sudo apt-get install docker.io -y

sudo usermod -aG docker ubuntu && newgrp docker


########################################################

#
- Install and configure SonarQube (Master machine)
- <i>Pull the latest SonarQube Community Edition</i>
```bash
    docker pull sonarqube:community
```
- <i>Run the latest SonarQube docker container</i>
```bash
        docker run -d \
          --name sonarqube \
          -p 9000:9000 \
          -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
          sonarqube:community
```
```bash
    docker ps
    You should see SonarQube container is running
```
```bash
    Got to the link: <ec2-machine-ip>:9000/ and setup SonarQube account
    Initial user & passwoed: admin, admin
```
#
- Install Trivy (On Master Machine)
```bash
> Update dependencies
sudo apt update -y && sudo apt install -y wget curl apt-transport-https gnupg lsb-release

> Add the Trivy signing key (secure method)
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | \
  sudo gpg --dearmor -o /usr/share/keyrings/trivy-archive-keyring.gpg

> Add the Trivy repository (clean, non-duplicating)
echo "deb [signed-by=/usr/share/keyrings/trivy-archive-keyring.gpg] \
  https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list > /dev/null

> Install the latest Trivy
sudo apt update -y
sudo apt install -y trivy

> Verify version
trivy --version
```

#
## Add email for notification
<p>
  Follow this 
  <a href="https://docs.google.com/document/d/1dFRT_RP4yhHcCMiZug1mMVc3XWnDap8g4iMR8XLIAHw/view" target="_blank">document</a> for email app and set it up to Jenkins
</p>

## Steps to implement the project:
- <b>Go to Jenkins Master and click on <mark> Manage Jenkins --> Plugins --> Available plugins</mark> install the below plugins:</b>
  - OWASP Dependency-Check
  - SonarQube Scanner
  - Docker
  - Pipeline: Stage View
  - Blue Ocean


