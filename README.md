# Comprehensive Guide to Setting Up Jenkins CI/CD with Docker, SonarQube, and ArgoCD on Ubuntu

This guide provides a step-by-step walkthrough for setting up a Jenkins CI/CD pipeline integrated with Docker, SonarQube, and ArgoCD on an Ubuntu server.

---

## 1. Create an EC2 Instance
1. Launch an EC2 instance using Ubuntu OS and the `t2.large` instance type.
2. Open all inbound traffic for the instance (for demo purposes). For production, configure security rules based on your requirements.

---

## 2. Connect to the EC2 Instance
SSH into the EC2 instance:
```bash
ssh -i <your-key.pem> ubuntu@<your-public-ip>
```

---

## 3. Install Java
Install OpenJDK 17:
```bash
sudo apt update
sudo apt install openjdk-17-jre
```
Verify the Java installation:
```bash
java -version
```

---

## 4. Install Jenkins

### 4.1 Add Jenkins Repository and Install
Import the Jenkins key:
```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```
Add the Jenkins repository:
```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```
Update package list and install Jenkins:
```bash
sudo apt-get update
sudo apt-get install jenkins
```

### 4.2 Check Jenkins Process
Check if Jenkins is running:
```bash
ps -ef | grep jenkins
```

### 4.3 Open Jenkins Web Interface
Access Jenkins via the public IP address and port `8080` (e.g., `http://<your-public-ip>:8080`).

Retrieve the Jenkins unlock password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Unlock Jenkins, install recommended plugins, and set up an admin username and password.

### 4.4 Create and Configure a Jenkins Pipeline Job
1. Create a new job and select **Pipeline** as the job type.
2. Set up the GitHub repository URL and configure the `Jenkinsfile` path.

#### Configure GitHub Integration
1. Set up GitHub credentials in Jenkins.
2. Use the credentials to allow Jenkins to access your repository.

### 4.5 Install Docker Plugin in Jenkins
Install the Docker plugin through the Jenkins Plugin Manager.

---

## 5. Install SonarQube for Static Code Analysis

### 5.1 Install SonarQube on Ubuntu Server
Switch to the root user:
```bash
sudo su -
```
Add a user for SonarQube:
```bash
adduser sonarqube
```
Download and install SonarQube:
```bash
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip sonarqube-9.4.0.54424.zip
chmod -R 755 sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube sonarqube-9.4.0.54424
```
Start SonarQube:
```bash
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```
Access SonarQube via `http://<your-public-ip>:9000`.

Default credentials:
- Username: `admin`
- Password: `admin`

### 5.2 Configure SonarQube in Jenkins
1. Generate a SonarQube token from **My Account > Security** in the SonarQube dashboard.
2. Add the token as a **Secret Text** in Jenkins under **Manage Jenkins > Manage Credentials**.

---

## 6. Install Docker on Ubuntu Server
Update the package list and install Docker:
```bash
sudo apt update
sudo apt install docker.io
```
Add Jenkins and Ubuntu users to the Docker group:
```bash
usermod -aG docker jenkins
usermod -aG docker ubuntu
```
Restart Docker:
```bash
systemctl restart docker
```

---

## 7. Set Up Maven and Docker in Jenkins Pipeline
1. Install Maven and Docker tools on the Jenkins server.
2. Configure the pipeline to use Maven and Docker for building the project.

---

## 8. Set Up ArgoCD for Kubernetes Deployment

### 8.1 Install ArgoCD Using Operators
1. Install the ArgoCD Operator from OperatorHub:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
spec: {}
```
Apply the configuration:
```bash
kubectl apply -f argo-cd.yaml
```
Check running pods:
```bash
kubectl get pods -n operators
```

### 8.2 Configure ArgoCD with GitHub
Sync the GitHub repository with ArgoCD. Set up the GitHub repository URL, cluster name, and other necessary configurations.

### 8.3 Access ArgoCD Web Interface
Access ArgoCD via `http://<your-public-ip>:8080` and log in with default credentials. Change the password upon first login.

---

## 9. Complete CI/CD Pipeline in Jenkins

Integrate Docker, SonarQube, ArgoCD, and GitHub into your Jenkins pipeline. Below is an example `Jenkinsfile`:

```groovy
pipeline {
  agent {
    docker {
      image 'ramagiri/maven-docker-agent:v1'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
      }
    }
    stage('Build and Test') {
      steps {
        sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
      }
    }
    stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://<your-sonarqube-ip>:9000"
      }
      steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
          sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "jithendarramagiri1998/ultimate-cicd:${env.BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
          sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
          def dockerImage = docker.image("${DOCKER_IMAGE}")
          docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
            dockerImage.push()
          }
        }
      }
    }
    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "Jenkins-Zero-To-Hero"
        GIT_USER_NAME = "************"
      }
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config user.email "jithendar*******gmail.com"
            git config user.name "***************"
            BUILD_NUMBER=${BUILD_NUMBER}
            sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
            git add java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
          '''
        }
      }
    }
  }
}
```

---

## Conclusion

With this setup, you now have a fully automated CI/CD pipeline integrating Jenkins, SonarQube, Docker, and Argo
