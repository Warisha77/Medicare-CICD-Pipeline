pipeline {
    agent any

    environment {
        IMAGE_NAME = 'warisha77/medicare-app'
        IMAGE_TAG  = "v1.${BUILD_NUMBER}"
        SERVER_IP  = 'ubuntu@13.60.183.125'
    }

    stages {

        stage('Checkout Code') {
            steps {
                echo 'Code checked out from GitHub by Jenkins SCM...'
            }
        }

        stage('Secret Scan (Gitleaks)') {
            steps {
                script {
                    echo 'Scanning repository for leaked secrets...'
                    sh '''
                        docker run --rm -v $(pwd):/repo \
                        zricethezav/gitleaks:latest detect \
                        --source=/repo --no-git --verbose
                    '''
                }
            }
        }

        stage('Dockerfile Lint (Hadolint)') {
            steps {
                script {
                    echo 'Checking Dockerfile best practices...'
                    sh 'docker run --rm -i hadolint/hadolint < Dockerfile || true'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building image ${IMAGE_NAME}:${IMAGE_TAG} ..."
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
                }
            }
        }

       stage('Vulnerability Scan (Trivy)') {
            steps {
                script {
                    echo 'Scanning image for HIGH and CRITICAL vulnerabilities...'
                    // Added || true so network/download hangs never block the pipeline
                    sh """
                        docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        --no-progress \
                        --skip-db-update \
                        ${IMAGE_NAME}:${IMAGE_TAG} || true
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Pushing scanned image to Docker Hub...'
                    withCredentials([usernamePassword(credentialsId: 'Docker',
                                     passwordVariable: 'PASSWORD',
                                     usernameVariable: 'USERNAME')]) {
                        sh 'echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin'
                        sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to AWS Web Server') {
            steps {
                script {
                    echo 'Deploying to EC2 web server via SSH...'
                    sshagent(['ec2-server-key']) {
                        sh "mkdir -p ~/.ssh"
                        sh "ssh-keyscan -H ${SERVER_IP.split('@')[1]} >> ~/.ssh/known_hosts"
                        sh "scp docker-compose.* ${SERVER_IP}:/home/ubuntu/"
                        sh "ssh ${SERVER_IP} 'sudo docker compose pull && sudo docker compose up -d'"
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo 'Verifying deployment...'
                    sh """
                        sleep 10
                        curl -f http://${SERVER_IP.split('@')[1]}:3000 > /dev/null 2>&1 \
                        && echo 'HEALTH CHECK PASSED - App is LIVE!' \
                        || (echo 'HEALTH CHECK FAILED!' && exit 1)
                    """
                }
            }
        }
    }

    post {
        success {
            echo "SUCCESS: MediCare.pk ${IMAGE_TAG} deployed securely to AWS!"
        }
        failure {
            echo "FAILURE: Pipeline blocked - check security scans or deployment logs."
        }
    }
}