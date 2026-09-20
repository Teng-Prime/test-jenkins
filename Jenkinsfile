pipeline {
    agent any

    environment {
        // Make sure Jenkins can find Docker
        PATH = "/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"

        // Docker Hub
        DOCKER_IMAGE = "tengzzz/jenkins"

        // AWS EC2
        AWS_HOST = "16.176.27.153"
        CONTAINER_NAME = "jenkins-app"

        // Nginx inside Docker container
        APP_PORT = "80"
    }

    stages {

        // ========================================
        // 1. CHECKOUT
        // ========================================
        stage('Checkout Code') {
            steps {
                echo "========================================"
                echo "CHECKOUT CODE"
                echo "========================================"

                checkout scm

                echo "Source code checked out successfully."
            }
        }


        // ========================================
        // 2. CHECK DOCKER
        // ========================================
        stage('Check Docker') {
            steps {
                echo "========================================"
                echo "CHECK DOCKER"
                echo "========================================"

                sh '''
                    echo "PATH=$PATH"

                    echo "Current user:"
                    whoami

                    echo "Docker location:"
                    which docker

                    echo "Docker version:"
                    docker --version

                    echo "Docker status:"
                    docker ps
                '''
            }
        }


        // ========================================
        // 3. BUILD DOCKER IMAGE
        // ========================================
        stage('Build Docker Image') {
            steps {
                echo "========================================"
                echo "BUILD DOCKER IMAGE"
                echo "========================================"

                sh """
                    docker build \
                        -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        .

                    echo "Docker image built successfully."

                    docker images ${DOCKER_IMAGE}
                """
            }
        }


        // ========================================
        // 4. LOGIN + PUSH TO DOCKER HUB
        // ========================================
        stage('Push Image to Docker Hub') {
            steps {
                echo "========================================"
                echo "PUSH IMAGE TO DOCKER HUB"
                echo "========================================"

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-id',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "Logging into Docker Hub..."

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''

                    echo "Pushing build image..."

                    sh """
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """

                    echo "Pushing latest image..."

                    sh """
                        docker push ${DOCKER_IMAGE}:latest
                    """

                    echo "Docker image pushed successfully."
                }
            }
        }


        // ========================================
        // 5. TEST SSH CONNECTION TO AWS
        // ========================================
        stage('Test AWS SSH') {
            steps {
                echo "========================================"
                echo "TEST AWS SSH CONNECTION"
                echo "========================================"

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'aws-ec2',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        chmod 600 "$SSH_KEY"

                        echo "Connecting to AWS EC2..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            -i "$SSH_KEY" \
                            "$SSH_USER@$AWS_HOST" \
                            "echo 'AWS SSH connection successful!'"
                    '''
                }
            }
        }


        // ========================================
        // 6. DEPLOY TO AWS
        // ========================================
        stage('Deploy to AWS') {
            steps {
                echo "========================================"
                echo "DEPLOY TO AWS EC2"
                echo "========================================"

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'aws-ec2',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh """
                        chmod 600 "\$SSH_KEY"

                        echo "Connecting to AWS EC2..."

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            -i "\$SSH_KEY" \
                            "\$SSH_USER@${AWS_HOST}" "
                            
                                echo '----------------------------------------';
                                echo 'Connected to AWS EC2';
                                echo '----------------------------------------';

                                echo 'Docker version:';
                                docker --version;

                                echo '----------------------------------------';
                                echo 'Pulling Docker image...';
                                echo '----------------------------------------';

                                docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER};

                                echo '----------------------------------------';
                                echo 'Stopping old container...';
                                echo '----------------------------------------';

                                docker stop ${CONTAINER_NAME} || true;

                                echo '----------------------------------------';
                                echo 'Removing old container...';
                                echo '----------------------------------------';

                                docker rm ${CONTAINER_NAME} || true;

                                echo '----------------------------------------';
                                echo 'Starting new container...';
                                echo '----------------------------------------';

                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart unless-stopped \
                                    -p 80:${APP_PORT} \
                                    ${DOCKER_IMAGE}:${BUILD_NUMBER};

                                echo '----------------------------------------';
                                echo 'Deployment complete!';
                                echo '----------------------------------------';

                                echo 'Running containers:';

                                docker ps;
                            "
                    """
                }
            }
        }


        // ========================================
        // 7. VERIFY DEPLOYMENT
        // ========================================
        stage('Verify Deployment') {
            steps {
                echo "========================================"
                echo "VERIFY DEPLOYMENT"
                echo "========================================"

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'aws-ec2',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh """
                        chmod 600 "\$SSH_KEY"

                        ssh \
                            -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            -i "\$SSH_KEY" \
                            "\$SSH_USER@${AWS_HOST}" "
                            
                                echo 'Checking container...';

                                docker ps \
                                    --filter name=${CONTAINER_NAME};

                                echo '----------------------------------------';

                                echo 'Checking port 80...';

                                curl -I http://localhost:80 || true;

                                echo '----------------------------------------';

                                echo 'Deployment verification completed.';
                            "
                    """
                }
            }
        }
    }


    // ========================================
    // POST ACTIONS
    // ========================================
    post {

        success {
            echo """
========================================
       DEPLOYMENT SUCCESSFUL
========================================

Docker Image:
${DOCKER_IMAGE}:${BUILD_NUMBER}

Docker Hub:
${DOCKER_IMAGE}:latest

AWS EC2:
${AWS_HOST}

Container:
${CONTAINER_NAME}

Application:
http://${AWS_HOST}

Port:
80 -> ${APP_PORT}

========================================
"""
        }

        failure {
            echo """
========================================
         DEPLOYMENT FAILED
========================================

Build:
${BUILD_NUMBER}

Check the Jenkins Console Output
for the exact error.

========================================
"""
        }

        always {
            echo "Cleaning up Docker Hub login..."

            sh '''
                docker logout || true
            '''
        }
    }
}