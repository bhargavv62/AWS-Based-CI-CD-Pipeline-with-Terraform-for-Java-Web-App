# End-to-End CI/CD Project with EKS, Jenkins, SonarQube, OWASP Dependency-Check, Trivy, Docker, and Tomcat Deployment

This document explains how I built a complete DevSecOps pipeline on AWS using Terraform (EKS), Jenkins, SonarQube, Docker, Trivy, OWASP Dependency-Check, S3 artifact storage, and final deployment to Tomcat.

## Project Overview
- Infrastructure: AWS EKS cluster using Terraform
- CI/CD: Jenkins Pipeline
- Code Quality: SonarQube
- Security Scanning: OWASP Dependency-Check + Trivy (Image Scanning)
- Artifact Storage: Amazon S3
- Container Registry: Docker Hub
- Application: Java Web App (Docker + WAR)
- Final Deployment: Tomcat Server

## Prerequisites
- AWS Account with billing enabled
- MobaXterm or any SSH client
- Basic knowledge of AWS, Linux, Jenkins, Docker

---

### Step 1: Launch EC2 Instance (Jenkins + Tools Server)

1. Go to AWS Console → EC2 → Launch Instance
2. Name: `jenkins-server` (or any)
3. OS: **Amazon Linux 2023 AMI**
4. Instance Type: **m7i-flex.large** (or t3.large if budget constrained)
5. Key pair: Create or use existing
6. Security Group: Allow **All Traffic** (0.0.0.0/0) → only for learning/lab
7. Storage: 28 GB gp3 EBS volume
8. IAM Role: Create new role with **AdministratorAccess** (for learning only)
9. Launch instance
10. Connect via MobaXterm as `ec2-user`, then switch to root:
    ```bash
    sudo su -
    ```

---

### Step 2: Provision EKS Cluster using Terraform

1. Install Terraform:
   ```bash
   yum install -y yum-utils
   yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
   yum -y install terraform
   ```

2. Clone project repo:
   ```bash
   git clone https://github.com/bhargavv62/k8s-project.git
   cd k8s-project/Eks-terraform
   ```

       cd K8s-project/Eks-terraform

3. Create S3 bucket in AWS Console for Terraform state (e.g., `my-terraform-state-bucket-unique-name`) → note region

4. Edit `backend.tf`:
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "your-unique-bucket-name"
       key            = "eks/terraform.tfstate"
       region         = "us-east-1"  # change if needed
       dynamodb_table = "terraform-lock"  # optional
       use_lockfile   = true
     }
   }
   ```

5. Edit `main.tf`:
   - Line ~89: Desired/Min/Max capacity → `desired_size = 2`, `min_size = 2`, `max_size = 4`
   - Line ~93: Change instance type to `c7i-flex.large` (or m6i.large)

6. Run Terraform:
   ```bash
   terraform init
   terraform plan
   terraform apply --auto-approve
   ```
   Wait 15–20 mins for EKS cluster to be ready.

7. Verify:
   ```bash
   terraform state list
   aws eks update-kubeconfig --name <cluster-name> --region us-east-1
   kubectl get nodes
   ```

---

### Step 3: Install Jenkins

Open new terminal → SSH into same instance as root

```bash
sudo yum update –y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade
sudo yum install java-17-amazon-corretto -y
sudo yum install jenkins git -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo mkdir -p /var/tmp_disk
sudo chmod 1777 /var/tmp_disk
sudo mount --bind /var/tmp_disk /tmp
echo '/var/tmp_disk /tmp none bind 0 0' | sudo tee -a /etc/fstab
sudo systemctl mask tmp.mount
df -h /tmp
sudo systemctl restart jenkins
```

Access Jenkins: `http://<public-ip>:8080`  
Get initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install **Suggested Plugins** → Create admin user (remember username/password)

---

### Step 4: Install Required Tools on Jenkins Server

```bash
# Docker
yum install docker -y
systemctl start docker
systemctl enable docker
chmod 777 /var/run/docker.sock   # only for lab

# SonarQube
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access SonarQube: `http://<public-ip>:9000`  
Login: `admin` / `admin` → Change password →  
My Account → Security → Generate Token → Name: `mytoken` → Copy token

---

### Step 5: Configure Jenkins Plugins

Manage Jenkins → Plugins → Available → Install:
- Pipeline: Stage View
- SonarQube Scanner for Jenkins
- OWASP Dependency-Check
- Kubernetes
- Kubernetes CLI
- Docker Pipeline
- Deploy to container

---

### Step 6: Configure Global Tools in Jenkins

Manage Jenkins → Tools →

1. **SonarQube Scanner** 
   - Name: `mysonar`
   - Install automatically

2. **Maven**
   - Name: `mymaven`
   - Install automatically (latest)

3. **Dependency-Check**
   - Name: `DP-Check`
   - Install automatically from GitHub

---

### Step 7: Configure SonarQube in Jenkins

Manage Jenkins → System → SonarQube servers
- Name: `SonarServer`
- Server URL: `http://<public-ip>:9000`
- Server authentication token → Add → Secret Text → Paste token from SonarQube → ID: `sonar-token`

---

### Step 8: Create Jenkins Pipeline Job

New Item → Name: `mydeployment` → Pipeline → OK

#### Pipeline Stages (Add one by one)

**Stage 1: Clean Workspace**
```groovy
cleanWs()
```

**Stage 2: Checkout Code**
```groovy
git 'https://github.com/bhargavv62/dockerwebapp.git'
```

**Stage 3: SonarQube Analysis**
Use Pipeline Syntax → Snippet Generator → `withSonarQubeEnv` → Select `SonarServer` → Generate:
```groovy
withSonarQubeEnv('SonarServer') {
  sh 'mvn clean verify sonar:sonar'
}
```

**Stage 4: Quality Gate**
Pipeline Syntax → `waitForQualityGate` → Generate:
```groovy
timeout(time: 10, unit: 'MINUTES') {
  waitForQualityGate abortPipeline: true
}
```

**Stage 5: Maven Build**
```groovy
sh 'mvn clean package'
sh 'cp -r target Docker-app'
```

**Stage 6: OWASP Dependency-Check**
- Get NVD API Key: https://nvd.nist.gov/developers/request-an-api-key → Apply → Get key from email
- In Jenkins → Add Credentials → Secret Text → Paste API key → ID: `nvd-api-key`

Pipeline Syntax → dependencyCheck → Additional arguments:
```
--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}
```
Then → dependencyCheckPublisher → pattern: `**/dependency-check-report.xml`

**Stage 7: Upload Artifact to S3**
- Create IAM User with AmazonS3FullAccess
- Create access key → Copy Access Key & Secret
- In Jenkins → Manage Jenkins → Amazon S3 Profile → Add → Test & Save

Pipeline Syntax → S3 Upload:
- File: `target/vprofile-v2.war`
- Bucket: `your-artifact-bucket-name`
- Region: match your bucket region

**Stage 8: Build Docker Images**
```groovy
sh 'docker build -t appimage Docker-app/'
sh 'docker build -t dbimage Docker-db/'
```

**Stage 9: Trivy Image Scan**
First install Trivy on Jenkins server:
```bash
rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.53.0/trivy_0.53.0_Linux-64bit.rpm
```

In pipeline:
```groovy
sh 'trivy image appimage'
sh 'trivy image dbimage'
```

**Stage 10: Push to Docker Hub**
Add Docker Hub credentials in Jenkins (ID: `dockerhub`)
```groovy
docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
  sh 'docker tag appimage yourusername/vprofile-app:latest'
  sh 'docker tag dbimage yourusername/vprofile-db:latest'
  sh 'docker push yourusername/vprofile-app:latest'
  sh 'docker push yourusername/vprofile-db:latest'
}
```

**Stage 11: Deploy to Tomcat**

1. Launch new EC2 → Amazon Linux → Allow All Traffic
2. Install Tomcat:
   ```bash
   yum install tomcat tomcat-webapps tomcat-admin-webapps -y
   systemctl start tomcat
   ```
3. Edit `/usr/share/tomcat/conf/tomcat-users.xml` → Add:
   ```xml
   <role rolename="manager-gui"/>
   <role rolename="manager-script"/>
   <user username="tomcat" password="admin@123" roles="manager-gui,manager-script"/>
   ```
4. Restart Tomcat: `systemctl restart tomcat`

5. In Jenkins → Credentials → Add → Username/Password → `tomcat` / `admin@123` → ID: `tomcat-creds`

6. Install **Deploy to container** plugin

7. Pipeline Syntax → deploy: Deploy war/ear to container
   - WAR: `target/vprofile-v2.war`
   - Context path: `vprofile`
   - Tomcat URL: `http://<tomcat-ip>:8080`
   - Credentials: `tomcat-creds`

Add generated step to pipeline.

---

### Final Pipeline Success!

After building the pipeline:
- Application successfully deployed
- Accessible at: `http://<tomcat-public-ip>:8080/vprofile`

---

## Project Repository Links
- EKS Terraform: https://github.com/bhargavv62/k8s-project.git
- Application Code: https://github.com/bhargavv62/dockerwebapp.git

---

**Note**: This setup uses open security groups and admin privileges for learning purposes only. In production, follow security best practices (VPC, least privilege IAM, HTTPS, etc.).
