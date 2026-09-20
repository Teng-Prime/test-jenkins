pipeline {
    agent any

    environment {
    PATH = "/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"

    DOCKER_IMAGE = "tengzzz/jenkins"
    AWS_USER = "ec2-user"
    AWS_HOST = "16.176.27.153"
    CONTAINER_NAME = "jenkins-app"
    APP_PORT = "8080"
}

    stages {

        stage('Checkout Code') {
            steps {
                echo "Checking out source code..."
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."

                sh """
                    docker build \
                        -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                """
            }
        }

        stage('Push Image') {
            steps {
                echo "Logging into Docker Hub..."

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-id',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''

                    echo "Pushing Docker image..."

                    sh """
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy to AWS') {
            steps {
                echo "Deploying to AWS EC2..."

                sshagent(credentials: ['aws-ec2']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            ${AWS_USER}@${AWS_HOST} "
                                echo 'Connected to AWS EC2';

                                echo 'Pulling Docker image...';
                                docker pull ${DOCKER_IMAGE}:${BUILD_NUMBER};

                                echo 'Stopping old container...';
                                docker stop ${CONTAINER_NAME} || true;

                                echo 'Removing old container...';
                                docker rm ${CONTAINER_NAME} || true;

                                echo 'Starting new container...';
                                docker run -d \
                                    --name ${CONTAINER_NAME} \
                                    --restart unless-stopped \
                                    -p 80:${APP_PORT} \
                                    ${DOCKER_IMAGE}:${BUILD_NUMBER};

                                echo 'Deployment complete!';
                                docker ps;
                            "
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo "Verifying deployment..."

                sshagent(credentials: ['aws-ec2']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            ${AWS_USER}@${AWS_HOST} \
                            "docker ps --filter name=${CONTAINER_NAME}"
                    """
                }
            }
        }
    }

}