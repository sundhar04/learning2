pipeline {
    agent any
    triggers {
        githubPush()
        pollSCM('H/5 * * * *') // Poll every 5 minutes
    }
    environment {
        EC2_HOST = "ec2-18-175-250-133.eu-west-2.compute.amazonaws.com"
        EC2_USER = "ubuntu"
        APP_DIR  = "learning2"
        IMAGE_NAME = "sundhar04/githubimage:latest"
        CONTAINER_NAME = "my_app_container"
    }
    stages {
        stage("Checkout") {
            steps {
                // Checkout the repository - this tells Jenkins which repo to monitor
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [[$class: 'CleanBeforeCheckout']],
                    submoduleCfg: [],
                    userRemoteConfigs: [[
                        url: 'https://github.com/sundhar04/learning2.git',
                        credentialsId: 'github-credentials'
                    ]]
                ])
            }
        }
        stage("Deploy to EC2") {
            steps {
                sshagent(['ec2-ssh-key']) {
                    withCredentials([usernamePassword(credentialsId: 'githubcredd', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
ssh -o StrictHostKeyChecking=no \$EC2_USER@\$EC2_HOST << "EOF"
set -e
cd /home/ubuntu

echo "Checking for Docker"
if ! command -v docker &> /dev/null; then
    echo "[!] Docker not found. Installing Docker..."
    sudo apt-get update -y
    sudo apt-get install -y ca-certificates curl gnupg lsb-release
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo \\
      "deb [arch=\$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \\
      https://download.docker.com/linux/ubuntu \\
      \$(lsb_release -cs) stable" | \\
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update -y
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    sudo usermod -aG docker ubuntu
    echo "Docker installed successfully"
else
    echo "Docker is already installed"
fi

echo "Cloning or updating repo"
if [ -d "$APP_DIR" ]; then
    cd "$APP_DIR" && git pull origin main && cd ..
else
    git clone https://\$GIT_USERNAME:\$GIT_PASSWORD@github.com/sundhar04/learning2.git
fi

echo "Cleaning old containers and images"
sudo docker stop "$CONTAINER_NAME" || true
sudo docker rm "$CONTAINER_NAME" || true
sudo docker rmi "$IMAGE_NAME" || true

echo "Building Docker image"
cd "$APP_DIR"
sudo docker build -t "$IMAGE_NAME" .

echo "Freeing port 8080 from any running container"
EXISTING_CONTAINER=\$(sudo docker ps -q --filter "publish=8080")
if [ ! -z "\$EXISTING_CONTAINER" ]; then
    echo "Found container using port 8080: \$EXISTING_CONTAINER"
    sudo docker stop \$EXISTING_CONTAINER
    sudo docker rm \$EXISTING_CONTAINER
fi

echo "Running Docker container"
sudo docker run -d --name "$CONTAINER_NAME" -p 8080:5000 "$IMAGE_NAME"
echo "Deployment complete. App running on port 8080"
EOF
                        """
                    }
                }
            }
        }
    }
    post {
        success {
            echo "deployment successful"
        }
        failure {
            echo "deployment failed"
        }
    }
}
