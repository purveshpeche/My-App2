pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'purveshpeche/myapp'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/purveshpeche/My-App2.git'
            }
        }

        stage('Set Version') {
            steps {
                script {
                    env.VERSION = "v${env.BUILD_NUMBER}"
                    echo "Generated image version: ${env.VERSION}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:$VERSION .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $DOCKER_IMAGE:$VERSION
                        docker logout
                    '''
                }
            }
        }

        stage('Test SSH Connectivity') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@ec2-18-204-195-116.compute-1.amazonaws.com echo "✅ SSH connection working!"'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
ssh -o StrictHostKeyChecking=no ubuntu@ec2-18-204-195-116.compute-1.amazonaws.com << 'EOF'
set -e

# Ensure Docker is installed
if ! command -v docker &> /dev/null; then
    echo "Installing Docker..."

    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS=\$ID
    fi

    if [ "\$OS" = "ubuntu" ]; then
        sudo apt update -y
        sudo apt install -y docker.io
        sudo systemctl start docker
        sudo systemctl enable docker
    elif [ "\$OS" = "amzn" ]; then
        sudo yum update -y
        sudo yum install -y docker
        sudo systemctl start docker
        sudo systemctl enable docker
    else
        echo "Unsupported OS. Exiting."
        exit 1
    fi
fi

echo "✅ Docker is ready. Pulling image and deploying..."

# Pull, stop old container, run new
sudo docker pull $DOCKER_IMAGE:$VERSION
sudo docker stop myapp || true
sudo docker rm myapp || true
sudo docker run -d --name myapp -p 80:80 $DOCKER_IMAGE:$VERSION

echo "🚀 Deployed $DOCKER_IMAGE:$VERSION successfully."
EOF
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD pipeline completed successfully. Deployed version: $VERSION"
        }
        failure {
            echo "❌ CI/CD pipeline failed."
        }
    }
}

