pipeline {
    agent any
    environment {
        APP_DIR = "/home/ubuntu/flask_app"
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
        EC2_USER = "ubuntu"
        EC2_IP = "13.57.8.246"
    }
    stages {
        stage('Checkout Code') {
            steps {
                dir("${APP_DIR}") {
                    git branch: 'main',
                        url: 'https://github.com/aakashrawat1910/CICDFlaskTest.git'
                }
            }
        }
        stage('Build') {
            steps {
                dir("${APP_DIR}") {
                    sh '''
                    apt update && apt install -y python3-venv python3-pip
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    '''
                }
            }
        }
        stage('Test') {
            steps {
                dir("${APP_DIR}") {
                    sh '''
                    . venv/bin/activate
                    pip install pytest
                    pytest || true
                    '''
                }
            }
        }
        stage('Deploy to EC2') {
            steps {
                script {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${EC2_USER}@${EC2_IP} <<EOF
                    set -e  # Stop on error

                    # Create application directory if it doesn't exist
                    mkdir -p ${APP_DIR}

                    # Navigate to application directory
                    cd ${APP_DIR}

                    # Pull the latest code from the repository
                    git pull origin main

                    # Set up virtual environment and install dependencies
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt

                    # Run the Flask application
                    nohup python3 -m app &

                    exit
                    EOF
                    """
                }
            }
        }
    }
}
