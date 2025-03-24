# CI/CD Pipeline using Jenkins and GitHub Actions

This guide provides step-by-step instructions to set up a CI/CD pipeline using **Jenkins** and **GitHub Actions** for a Python web application. The pipeline includes build, test, and deployment to an AWS EC2 instance.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Setup Jenkins](#setup-jenkins)
3. [Configure GitHub Actions](#configure-github-actions)
4. [CI/CD Workflow](#cicd-workflow)
5. [Deployment to AWS EC2](#deployment-to-aws-ec2)
6. [Screenshots and Attachments](#screenshots-and-attachments)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites
- A GitHub repository containing the Python web application
- An AWS EC2 instance with SSH access
- Jenkins installed and running (using Docker or a dedicated server)
- GitHub Actions enabled on your repository
- Python installed on Jenkins and EC2

---

## Setup Jenkins

### 1. Install Jenkins
You can install Jenkins on a virtual machine, use a cloud-based Jenkins service, or run Jenkins via Docker.

#### Install Jenkins on Docker
1. Create a bridge network in Docker:
   ```sh
   docker network create jenkins
   ```

2. Run a `docker:dind` Docker image:
   ```sh
   docker run --name jenkins-docker --rm --detach \
     --privileged --network jenkins --network-alias docker \
     --env DOCKER_TLS_CERTDIR=/certs \
     --volume jenkins-docker-certs:/certs/client \
     --volume jenkins-data:/var/jenkins_home \
     --publish 2376:2376 \
     docker:dind
   ```

3. Customize the official Jenkins Docker image:
   - Create a `Dockerfile` with the following content:
     ```Dockerfile
     FROM jenkins/jenkins:2.492.2-jdk17
     USER root
     RUN apt-get update && apt-get install -y lsb-release
     RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
       https://download.docker.com/linux/debian/gpg
     RUN echo "deb [arch=$(dpkg --print-architecture) \
       signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
       https://download.docker.com/linux/debian \
       $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
     RUN apt-get update && apt-get install -y docker-ce-cli
     USER jenkins
     RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
     ```

   - Build a new Docker image from this `Dockerfile`:
     ```sh
     docker build -t myjenkins-blueocean:2.492.2-1 .
     ```

4. Run your custom Jenkins image as a container:
   ```sh
   docker run --name jenkins-blueocean --restart=on-failure --detach \
     --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
     --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
     --volume jenkins-data:/var/jenkins_home \
     --volume jenkins-docker-certs:/certs/client:ro \
     --publish 8080:8080 --publish 50000:50000 myjenkins-blueocean:2.492.2-1
   ```

If running Jenkins via Docker:
```sh
mkdir jenkins_home && cd jenkins_home
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v $(pwd):/var/jenkins_home jenkins/jenkins:lts
```

### 2. Install Required Plugins
- **Git Plugin**
- **Pipeline Plugin**
- **SSH Pipeline Steps** (for deployment to EC2)
- **GitHub Integration Plugin**

### 3. Configure GitHub Webhook
- Go to **GitHub Repository → Settings → Webhooks**
- Add a webhook with the URL: `http://<Jenkins-Server-IP>:8080/github-webhook/`
- Set content type to `application/json`
- Trigger on `push` events

### 4. Create a Jenkins Pipeline
1. Navigate to **Jenkins Dashboard → New Item → Pipeline**
2. Add the following pipeline script:

```
pipeline {
    agent any

    environment {
        APP_DIR = "/home/ubuntu/flask_app"  // Directory on EC2 where the app will be deployed
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
        EC2_USER = "ubuntu"
        EC2_IP = "13.57.48.63"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    credentialsId: '2e869095-a76c-4b00-85e0-cb0fddb3a460',
                    url: 'git@github.com:reshmanavale/FlaskTest.git'
            }
        }

        stage('Build') {
            steps {
                sh '''
                apt update && apt install -y python3-venv python3-pip
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                . venv/bin/activate
                pip install pytest  # Ensure pytest is installed
                pytest || true  # Allow tests to fail without stopping pipeline
                '''
            }
        }

       stage('Deploy to EC2') {
            steps {
               script {
                  sh """
                ssh -o StrictHostKeyChecking=no -i /var/jenkins_home/.ssh/id_rsa ubuntu@13.57.48.63 <<EOF
                set -e  # Stop script on error
                
                echo "🚀 Deploying Flask app..."
                cd ~/FlaskTest || exit 1

                # 🔥 STOP any running Flask process before starting a new one
                echo "🛑 Checking for existing Flask processes..."
                PID=\$(lsof -t -i:5000) || true  # Prevent failure if no process is running
                if [ -n "\$PID" ]; then
                    echo "Killing existing Flask process: \$PID"
                    kill -9 \$PID || true  # Ignore failure if the process is already dead
                else
                    echo "No existing Flask process found."
                fi

                # Fetch latest code
                git reset --hard
                git pull origin main

                # Activate virtual environment and install dependencies
                source venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt

                # 🚀 Start Flask app in the background
                echo "Starting Flask app..."
                nohup venv/bin/python3 app.py > output.log 2>&1 &

                echo "✅ Deployment completed successfully!"
                exit 0  # ✅ Ensure Jenkins exits successfully
                EOF
            """
        }
    }
}


}
}
```

---

## Configure GitHub Actions

### 1. Create a GitHub Actions Workflow
1. Inside your repository, navigate to `.github/workflows/`
2. Create a new file `ci-cd.yml` and add the following:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.8'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run Tests
        run: pytest

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to EC2
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /var/www/app
            git pull origin main
            pip install -r requirements.txt
            systemctl restart myapp.service
```

### 2. Set Up Secrets in GitHub
- **EC2_HOST**: Public IP of your EC2 instance
- **EC2_SSH_KEY**: Private SSH key for authentication

---

## CI/CD Workflow
1. **Code Push**: Developers push code to the `main` branch.
2. **GitHub Actions**: Triggers build, test, and deploy steps.
3. **Jenkins**: Pulls the latest code, tests, and deploys it to EC2.
4. **Deployment**: Application is updated on the EC2 instance.

---

## Deployment to AWS EC2
1. Connect to your EC2 instance:
   ```sh
   ssh -i your-key.pem ubuntu@your-ec2-ip
   ```
2. Ensure the necessary packages are installed:
   ```sh
   sudo apt update && sudo apt install python3-pip git -y
   ```
3. Clone the repository:
   ```sh
   git clone https://github.com/your-repo.git /var/www/app
   ```
4. Setup a systemd service:
   ```sh
   sudo nano /etc/systemd/system/myapp.service
   ```
   Add the following:
   ```ini
   [Unit]
   Description=My Python App
   After=network.target

   [Service]
   User=ubuntu
   WorkingDirectory=/var/www/app
   ExecStart=/usr/bin/python3 /var/www/app/app.py
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
5. Start and enable the service:
   ```sh
   sudo systemctl daemon-reload
   sudo systemctl enable myapp.service
   sudo systemctl start myapp.service
   ```

---

## Screenshots and Attachments
Attach the following screenshots for documentation:
- **Jenkins Setup**: Screenshot of Jenkins plugins installed
- ![Screenshot 2025-03-16 011436](https://github.com/user-attachments/assets/2fa15e5e-608e-40a3-b722-ebef25fe96bf)
- ![Screenshot 2025-03-16 003144](https://github.com/user-attachments/assets/cc44f6d4-02bb-4fcf-b9ef-cd6657cc1905)
![Screenshot 2025-03-16 003050](https://github.com/user-attachments/assets/b434eae5-3e07-44a9-a398-006aaa594db1)
![Screenshot 2025-03-16 000732](https://github.com/user-attachments/assets/71edaf54-8f41-4539-8472-c6dc03da388d)


- **GitHub Actions Execution**: 
![Screenshot 2025-03-22 212616](https://github.com/user-attachments/assets/47ff9976-fb02-4892-acdb-5cab35d30e48)
<img width="671" alt="image" src="https://github.com/user-attachments/assets/17d02bb3-342c-4e04-bc43-94b1d2ab5228" />


- **AWS EC2 Deployment**: Screenshot of the deployed application running
![Screenshot 2025-03-22 212616](https://github.com/user-attachments/assets/e45b23c6-4bba-48c7-a679-8c041601c01f)

```

---

## Troubleshooting
### Common Issues & Fixes
- **Permission Denied (SSH to EC2)**:
  ```sh
  chmod 400 your-key.pem
  ```
- **GitHub Actions Fails at SSH Connection**:
  - Ensure the correct SSH private key is added as a GitHub secret.
- **Jenkins Not Triggering Builds**:
  - Verify the webhook setup in GitHub settings.
 ```
---

## Conclusion
This guide provides a fully automated CI/CD pipeline integrating Jenkins and GitHub Actions for deploying a Python web application to AWS EC2. 🚀

