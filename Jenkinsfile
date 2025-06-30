pipeline {
    agent any

    parameters {
        string(name: 'DEPLOY_VERSION', defaultValue: '', description: 'Deploy a specific Docker image version. Leave empty to skip deployment.')
    }

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
                    echo "Generated version: ${env.VERSION}"
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
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $DOCKER_IMAGE:$VERSION
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression { return params.DEPLOY_VERSION?.trim() }
            }
            steps {
                script {
                    echo "Deploying version: ${params.DEPLOY_VERSION}"
                }

                sshagent(['ec2-ssh-key']) {
                    sh """
ssh -o StrictHostKeyChecking=no ec2-user@ec2-18-204-195-116.compute-1.amazonaws.com << 'EOF'
sudo docker pull $DOCKER_IMAGE:${params.DEPLOY_VERSION}
sudo docker stop myapp || true
sudo docker rm myapp || true
sudo docker run -d --name myapp -p 80:80 $DOCKER_IMAGE:${params.DEPLOY_VERSION}
EOF
                    """
                }
            }
        }
    }
}

