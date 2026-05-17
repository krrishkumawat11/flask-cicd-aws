# DevOps Project: Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

**Author:** Krrish Kumawat &nbsp;|&nbsp; **Date:** May 2026

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-Python-000000?style=for-the-badge&logo=flask&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-Database-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-Version%20Control-181717?style=for-the-badge&logo=github&logoColor=white)

---

## Project Overview

In this project, I built a **fully automated CI/CD pipeline** to deploy a 2-tier Flask + MySQL application on **AWS EC2** using **Jenkins**, **Docker**, and **Docker Compose**. It integrates GitHub for version control and ensures seamless, hands-free deployments on every code push via GitHub Webhooks.

---

## Table of Contents

1. [Architecture Diagram](#architecture-diagram)
2. [Tech Stack](#tech-stack)
3. [Step 1: AWS EC2 Instance Setup](#step-1-aws-ec2-instance-setup)
4. [Step 2: Install Dependencies](#step-2-install-dependencies)
5. [Step 3: Jenkins Installation & Setup](#step-3-jenkins-installation--setup)
6. [Step 4: GitHub Repository Configuration](#step-4-github-repository-configuration)
7. [Step 5: Jenkins Pipeline Creation](#step-5-jenkins-pipeline-creation)
8. [Step 6: GitHub Webhook Setup](#step-6-github-webhook-setup)
9. [Conclusion](#conclusion)

---

## Architecture Diagram

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same AWS EC2)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| **AWS EC2** | Cloud server to host Jenkins + Application |
| **Jenkins** | CI/CD automation server |
| **Docker** | Containerization of Flask app |
| **Docker Compose** | Multi-container orchestration (Flask + MySQL) |
| **GitHub** | Source code management |
| **GitHub Webhooks** | Auto-trigger Jenkins pipeline on push |
| **Flask** | Python web framework |
| **MySQL** | Relational database |
| **Ubuntu 22.04** | OS on EC2 instance |

---

## Step 1: AWS EC2 Instance Setup

1. Navigate to **AWS EC2 Console** → Launch Instance
2. Select **Ubuntu 22.04 LTS** AMI — Free Tier eligible
3. Choose **t2.micro** instance type
4. Create a new key pair → download `.pem` file
5. Configure **Security Group** with these inbound rules:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | Your IP |
| Custom TCP | TCP | 8080 | Anywhere (Jenkins) |
| Custom TCP | TCP | 3000 | Anywhere (Flask App) |

6. Set storage to **20 GB**
7. Click **Launch Instance**

---

## Step 2: Install Dependencies

Connect to EC2 via **EC2 Instance Connect** (browser-based SSH), then run:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Java (required for Jenkins)
sudo apt install openjdk-17-jdk -y
java -version

# Set up Docker's apt repository.

# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

# Install the Docker packages.

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo systemctl start docker

# Add swap memory (for t2.micro)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

---

## Step 3: Jenkins Installation & Setup

```bash
# Update the Debian apt repositories, install OpenJDK 21

sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version

# Add Jenkins Repository and Install

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins

# Grant Jenkins Docker permissions
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart jenkins
```

**Access Jenkins at:** `http://<your-ec2-public-ip>:8080`

Get initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- Paste password → Install suggested plugins → Create admin user

---

## Step 4: GitHub Repository Configuration

Make sure your repo contains these files:

### `app.py`
```python
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def home():
    return "<h1>Flask CI/CD Pipeline - Running on AWS!</h1><p>Deployed via Jenkins</p>"

@app.route('/health')
def health():
    return {"status": "healthy"}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### `Dockerfile`
```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### `docker-compose.yml`
```yaml
version: '3.8'
services:
  flask-app:
    build: .
    ports:
      - "3000:5000"
    depends_on:
      - mysql
    environment:
      - DB_HOST=mysql
      - DB_USER=root
      - DB_PASSWORD=rootpassword
      - DB_NAME=flaskdb
    restart: always

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: flaskdb
    volumes:
      - mysql-data:/var/lib/mysql
    restart: always

volumes:
  mysql-data:
```

### `Jenkinsfile`
```groovy
pipeline {
    agent any

    stages {
        stage('Clone') {
            steps {
                echo 'Cloning repository...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building Docker image...'
                sh 'docker-compose build'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh 'docker-compose down || true'
                sh 'docker-compose up -d'
            }
        }
    }

    post {
        success {
            echo 'Pipeline successful! App is running.'
        }
        failure {
            echo 'Pipeline failed! Check logs.'
        }
    }
}
```

---

## Step 5: Jenkins Pipeline Creation

1. Jenkins Dashboard → **New Item**
2. Name: `flask-cicd-pipeline` → Select **Pipeline** → OK
3. Configure:
   - **GitHub project** → Enter repo URL
   - **GitHub hook trigger for GITScm polling** (for auto-deploy)
   - **Pipeline Definition:** Pipeline script from SCM
   - **SCM:** Git
   - **Repository URL:** `https://github.com/krrishkumawat11/flask-cicd-aws.git`
   - **Credentials:** Add GitHub token
   - **Branch:** `*/main`
   - **Script Path:** `Jenkinsfile`
4. Click **Save** → **Build Now**

---

## Step 6: GitHub Webhook Setup

1. Go to your GitHub repo → **Settings → Webhooks → Add webhook**
2. Fill in:
   - **Payload URL:** `http://<your-ec2-ip>:8080/github-webhook/`
   - **Content type:** `application/json`
   - **Events:** Just the push event 
   - **Active:** 
3. Click **Add webhook**

Now every `git push` will **automatically trigger** the Jenkins pipeline! 

---

## Conclusion

The CI/CD pipeline is now **fully operational**. Any `git push` to the `main` branch automatically:

1. Triggers Jenkins via GitHub Webhook
2. Clones the latest code
3. Builds Docker image
4. Deploys Flask + MySQL containers

**Live App:** `http://<your-ec2-ip>:3000`  
**Jenkins Dashboard:** `http://<your-ec2-ip>:8080`

## Infrastructure Diagram

---<img width="1000" height="1600" alt="infra_final" src="https://github.com/user-attachments/assets/01555f7e-a4e7-422f-acbc-e4929c610c8e" />


## Key Learnings

- Setting up Jenkins CI/CD pipeline from scratch on AWS EC2
- Containerizing a multi-tier app with Docker & Docker Compose
- Automating deployments using GitHub Webhooks
- Managing AWS EC2 security groups and instance configuration
- Writing declarative Jenkinsfile pipeline-as-code

---

## Topics

`jenkins` `docker` `docker-compose` `flask` `mysql` `aws` `ec2` `cicd` `devops` `github-webhooks` `automation` `containerization` `devops-project`
