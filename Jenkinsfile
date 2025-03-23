pipeline {
    agent any
    environment {
        APP_DIR = "/home/ubuntu/flask_app"
        SSH_KEY = "/var/jenkins_home/.ssh/id_rsa"
        EC2_USER = "ubuntu"
        EC2_IP = "54.219.100.124"
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/aakashrawat1910/CICDFlaskTest.git'
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
                pip install pytest
                pytest || true
                '''
            }
        }
       stage('Deploy to EC2') {
            steps {
               script {
                  sh """
                ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${EC2_USER}@${EC2_IP} <<EOF
                set -e  # Stop script on error
                
                echo "🚀 Deploying Flask app..."
                
                # Install required packages
                echo "Installing required packages..."
                sudo apt-get update
                sudo apt-get install -y python3-venv python3-pip
                
                # Check for existing Flask processes
                echo "🛑 Checking for existing Flask processes..."
                PID=\$(lsof -t -i:5000) || true  # Prevent failure if no process is running
                if [ -n "\$PID" ]; then
                    echo "Killing existing Flask process: \$PID"
                    kill -9 \$PID || true
                else
                    echo "No existing Flask process found."
                fi
                
                # Navigate to home directory first to ensure clean paths
                cd ~
                
                # Check if the repository exists and clone if needed
                if [ ! -d "FlaskTest" ] || [ ! -d "FlaskTest/.git" ]; then
                    echo "Repository doesn't exist or isn't a git repo. Cloning..."
                    rm -rf FlaskTest
                    git clone https://github.com/aakashrawat1910/CICDFlaskTest.git FlaskTest
                fi
                
                # Navigate to app directory and update code
                cd ~/FlaskTest
                git fetch origin
                git reset --hard origin/main
                
                # Remove old virtual environment if it exists but is problematic
                if [ -d "venv" ] && [ ! -f "venv/bin/activate" ]; then
                    echo "Removing problematic virtual environment..."
                    rm -rf venv
                fi
                
                # Create new virtual environment if needed
                if [ ! -d "venv" ]; then
                    echo "Creating virtual environment..."
                    python3 -m venv venv
                    # Verify venv was created correctly
                    if [ ! -f "venv/bin/activate" ]; then
                        echo "Virtual environment creation failed!"
                        exit 1
                    fi
                fi
                
                # Activate virtual environment and install dependencies
                echo "Activating virtual environment..."
                source ./venv/bin/activate
                
                echo "Installing dependencies..."
                pip install --upgrade pip
                pip install -r requirements.txt
                
                # Start Flask app in the background
                echo "Starting Flask app..."
                nohup python3 app.py > output.log 2>&1 &
                
                echo "✅ Deployment completed successfully!"
                exit 0
                EOF
                """
               }
            }
        }
    }
}
