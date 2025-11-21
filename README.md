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

## PORT-IP'S
- JENKINS - 8080
- SONARQUBE - 9000
- TOMCAT - 8080
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
 11. Launch another with c7i-flex.large for TOMCAT everything same expect IAM configuration   

---
 
   



### Step 2: JENKINS SERVER → Provision EKS Cluster using Terraform

1. Install Terraform:
   ```bash
   sudo yum install -y yum-utils shadow-utils
   sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
   sudo yum install terraform
   ```

2. Clone project repo:
   ```bash
   git clone https://github.com/bhargavv62/k8s-project.git
   cd k8s-project/Eks-terraform
   ```


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
   - Line ~93: Change instance type to `c7i-flex.large` 

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

### Step 4: Install Docker on Jenkins Server

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

Manage Jenkins → Plugins → Available → Select below listed → Install:
- Pipeline: Stage View
- SonarQube Scanner for Jenkins
- OWASP Dependency-Check
- Kubernetes
- Kubernetes CLI
- Docker Pipeline
- Deploy to container

---
- Restart jenkins
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
- Name: `mysonar`
- Server URL: `http://<public-ip>:9000`
- Server authentication token → Add → Secret Text → Paste token from SonarQube → ID: `sonar-token`

---
Manage Jenkins → System → Amazon s3 profiles → Profile name: mybucket

### Step 8: Create Jenkins Pipeline Job

Jenkins-Full-Pipeline  →  https://github.com/bhargavv62/AWS-Based-CI-CD-Pipeline-with-Terraform-for-Java-Web-App/blob/fb07b5ce490e827f500a4ec318dcb20da0f7aa9a/AWS-Based-CI-CD-Pipeline-with-Terraform-for-Java-Web-App.txt

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

Use Pipeline Syntax → Snippet Generator → `withSonarQubeEnv` → Select `mysonar` → select → add → jenkins → add credentials → select → Generate:

```groovy
          steps{
                withSonarQubeEnv('mysonar'){
                    sh "mvn clean verify sonar:sonar -Dsonar.projectKey=MyProject"
          }
```

**Stage 4: Quality Gates**

Sonarqube server → project settings → webhooks → create → name: jenkins-integration →  url: jenkinsurl/sonarqube-webhook/ →  save  
Pipeline Syntax → `waitForQualityGate` → select → sonartoken → Generate:

```groovy
  script{
  waitForQualityGate abortPipeline: false, credentialsId: 'sonartoken'
}
```

**Stage 5: Code Build**

```groovy
sh 'mvn clean package'
sh 'cp -r target Docker-app'
```

**Stage 6: OWASP Dependency-Check**

- Get NVD API Key: https://nvd.nist.gov/developers/request-an-api-key → Apply → Get key from email
- In Jenkins → dependencyCheck → add → jenkins → Add Credentials → Secret Text → Paste API key → ID: `nvd-api-key`

Pipeline Syntax → dependencyCheck → select →  Additional arguments:

```
--scan ./ --disableYarnAudit --disableNodeAudit 
```
Then → dependencyCheckPublisher → Xml report pattern: `**/dependency-check-report.xml`

Generate pipeline →  script should look like this 

    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', nvdCredentialsId: 'nvd', odcInstallation: 'DP-check'
      dependencyCheckPublisher pattern: '**/dependency-check-report.xml'

**Stage 7: Upload Artifact to S3**

AWS console
- Create IAM User with AmazonS3FullAccess
- Create access key → Copy Access Key & Secret
- In Jenkins → Manage Jenkins → Amazon S3 Profile → Add → Test & Save
- AWS console → s3 → create a bucket → copy bucket name remember region name

Pipeline Syntax → S3 Upload:

- File: `target/vprofile-v2.war` To check war file → cd /var/lib/jenkins/workspace/mydeployment →  ll target
- Bucket: `your-artifact-bucket-name`
- Region: match your bucket region
- opt for no upload on build failure

      s3Upload consoleLogLevel: 'INFO', dontSetBuildResultOnFailure: false, dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: 'bhargavvb', excludedFile: '', flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: true, selectedRegion: 'ap-northeast-1', showDirectlyInBrowser: false, sourceFile: 'target/vprofile-v2.war', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: false]], pluginFailureResultConstraint: 'FAILURE', profileName: 'mybucket', userMetadata: []

**Stage 8: Build Docker Images**

```groovy
sh 'docker build -t appimage Docker-app'
sh 'docker build -t dbimage Docker-db'
```

**Stage 9: Trivy Image Scan**

First install Trivy on Jenkins server:
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.67.2
```

In pipeline:

```groovy
sh 'trivy image appimage'
sh 'trivy image dbimage'
```

**Stage 10: Tag&push**

Push to Docker Hub

Add Docker Hub credentials in Jenkins → Pipeline syntax → with docker Registry → Registry credentials → add →  jenkins → Username: Dockerhub username, Password: Dockerhub password → (ID: `dockerhub`) → select → dockerhub

```groovy
docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
  sh 'docker tag appimage yourusername/vprofile-app:latest'
  sh 'docker tag dbimage yourusername/vprofile-db:latest'
  sh 'docker push yourusername/vprofile-app:latest'
  sh 'docker push yourusername/vprofile-db:latest'
}
```
It should look like this

      script{
                withDockerRegistry(credentialsId: 'dockerhub') {
                    sh 'docker tag appimage bhargav62/socialmedia:app'
                    sh 'docker tag dbimage bhargav62/socialmedia:db'
                    sh 'docker push bhargav62/socialmedia:app'
                    sh 'docker push bhargav62/socialmedia:db'


**Stage 11: Deploy to Tomcat**

Installing in TOMCAT SERVER → connect →

  1. Tomcat EC2 → connect 
  2. Install Tomcat:
    
   ```bash
   # 1. Install Java 17 (Amazon Corretto)
yum install -y java-17-amazon-corretto-devel

# 2. Download latest Tomcat 9
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.112/bin/apache-tomcat-9.0.112.tar.gz

# 3. Extract
tar -xzf apache-tomcat-9.0.112.tar.gz

# 4. Move to /opt (standard location)
mv apache-tomcat-9.0.112 /opt/tomcat

# 5. Create tomcat user
groupadd -r tomcat
useradd -r -g tomcat -s /bin/false -d /opt/tomcat tomcat
chown -R tomcat:tomcat /opt/tomcat

# 6. Fix tomcat-users.xml (add user OUTSIDE comments)
cat > /opt/tomcat/conf/tomcat-users.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <user username="tomcat" password="admin@123" roles="manager-gui,manager-script"/>
</tomcat-users>
EOF

# 7. Allow remote access to Manager (comment out IP restriction)
sed -i 's/<Valve className="org.apache.catalina.valves.RemoteAddrValve"/<!-- & -->/' /opt/tomcat/webapps/manager/META-INF/context.xml
sed -i 's/<\/Valve>/<\/Valve> -->/' /opt/tomcat/webapps/manager/META-INF/context.xml

# 8. Create systemd service
cat > /etc/systemd/system/tomcat.service << 'EOF'
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# 9. Enable and start Tomcat
systemctl daemon-reload
systemctl enable tomcat
systemctl start tomcat

# 10. Open firewall
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload

# 11. Done!
echo "Tomcat installed! Access:"
echo "   Web: http://$(curl -s ifconfig.me):8080"
echo "   Manager: http://$(curl -s ifconfig.me):8080/manager/html"
echo "   User: tomcat | Pass: admin@123"
   ```
3. Wait 10 seconds → open in browser:
   
•	Tomcat Home: http://YOUR_IP:8080
•	Manager GUI: http://YOUR_IP:8080/manager/html → Login: tomcat / admin@123


Check status tomcat

   ```xml
   # Check status
   systemctl status tomcat

   # Check logs
   journalctl -u tomcat -f
   ```
4. Restart Tomcat: `systemctl restart tomcat`

5.   Pipeline Syntax → deploy: Deploy war/ear to container
   - WAR: `target/vprofile-v2.war`
   - Context path: `vprofile`
   - Tomcat URL: `http://<tomcat-ip>:8080`
   - Credentials: Credentials → Add → Username/Password → `tomcat` / `admin@123` → ID: `tomcat-creds``tomcat-creds`

Add generated step to pipeline  → it should look like this

    deploy adapters: [tomcat9(alternativeDeploymentContext: '', credentialsId: 'tomcat', path: '', url: 'http://<tomcat-ip>:8080')], contextPath: 'myapp', war: 'target/vprofile-v2.war'

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
