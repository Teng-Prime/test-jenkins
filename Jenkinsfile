```groovy
pipeline {
    agent any

    environment {
        // Docker Hub
        DOCKER_IMAGE = "tengzzz/jenkins"

        // Application
        CONTAINER_NAME = "jenkins-app"
        APP_PORT = "8080"

        // AWS EC2
        AWS_USER = "ec2-user"
        AWS_HOST = "16.176.27.153"
        AWS_PORT = "22"

        // Jenkins build number
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        // ==========================================
        // 1. CHECKOUT
        // ==========================================
        stage('Checkout') {
            steps {
                echo "Checking out source code..."

                checkout scm
            }
        }

        // ==========================================
        // 2. BUILD DOCKER IMAGE
        // ==========================================
        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."

                sh '''
                    docker build \
                        -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                '''
            }
        }

        // ==========================================
        // 3. TEST IMAGE
        // ==========================================
        stage('Test Docker Image') {
            steps {
                echo "Checking Docker image..."

                sh '''
                    docker image inspect ${DOCKER_IMAGE}:${IMAGE_TAG}

                    echo "Image created successfully:"
                    docker images ${DOCKER_IMAGE}
                '''
            }
        }

        // ==========================================
        // 4. LOGIN DOCKER HUB
        // ==========================================
        stage('Docker Hub Login') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        // ==========================================
        // 5. PUSH DOCKER IMAGE
        // ==========================================
        stage('Push Docker Image') {
            steps {

                echo "Pushing ${DOCKER_IMAGE}:${IMAGE_TAG}"

                sh '''
                    docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    docker push ${DOCKER_IMAGE}:latest
                '''
            }
        }

        // ==========================================
        // 6. DEPLOY AWS EC2
        // ==========================================
        stage('Deploy to AWS') {
            steps {

                sshagent(credentials: ['aws-ec2']) {

                    sh '''
                        echo "Connecting to AWS EC2..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -p ${AWS_PORT} \
                            ${AWS_USER}@${AWS_HOST} << 'REMOTE_COMMANDS'

                            set -e

                            echo "Connected to AWS EC2"

                            echo "Pulling latest image..."
                            docker pull ${DOCKER_IMAGE}:latest

                            echo "Stopping old container..."
                            docker stop ${CONTAINER_NAME} || true

                            echo "Removing old container..."
                            docker rm ${CONTAINER_NAME} || true

                            echo "Starting new container..."

                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                --restart unless-stopped \
                                -p 80:${APP_PORT} \
                                ${DOCKER_IMAGE}:latest

                            echo "Container started!"

                            docker ps --filter name=${CONTAINER_NAME}

                        REMOTE_COMMANDS
                    '''
                }
            }
        }

        // ==========================================
        // 7. VERIFY AWS DEPLOYMENT
        // ==========================================
        stage('Verify Deployment') {
            steps {

                sshagent(credentials: ['aws-ec2']) {

                    sh '''
                        echo "Verifying deployment..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -p ${AWS_PORT} \
                            ${AWS_USER}@${AWS_HOST} \
                            "docker ps --filter name=${CONTAINER_NAME}"
                    '''
                }
            }
        }
    }

    // ==========================================
    // POST
    // ==========================================
    post {

        success {
            echo '''
==========================================
       DEPLOYMENT SUCCESSFUL 🚀
==========================================

Docker Image:
tengzzz/jenkins:latest

AWS EC2:
16.176.27.153

Application:
http://16.176.27.153

Port:
80 → 8080

==========================================
'''
        }

        failure {
            echo '''
==========================================
       DEPLOYMENT FAILED ❌
==========================================

Check the Jenkins Console Output.

==========================================
'''
        }

        always {
            sh 'docker logout || true'
        }
    }
}
```
