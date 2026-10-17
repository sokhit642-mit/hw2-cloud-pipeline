pipeline {
    agent any

    environment {
        AWS_REGION     = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '251528148716'
        ECR_REPO_NAME  = 'my-app-h2'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "v${BUILD_NUMBER}"
        MANAGER_IP     = '32.236.227.255'
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} ."
                sh "docker tag ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO_NAME}:latest"
            }
        }

        stage('Push to Amazon ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-ecr-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                        docker run --rm \
                          -e AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}" \
                          -e AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}" \
                          -e AWS_DEFAULT_REGION="${AWS_REGION}" \
                          amazon/aws-cli ecr get-login-password --region "${AWS_REGION}" | docker login --username AWS --password-stdin "${ECR_REGISTRY}"
                        docker push "${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG}"
                        docker push "${ECR_REGISTRY}/${ECR_REPO_NAME}:latest"
                    """
                }
            }
        }

        stage('Rolling Update on Swarm') {
            steps {
                sshagent(['swarm-manager-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${MANAGER_IP} "
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker service update --image ${ECR_REGISTRY}/${ECR_REPO_NAME}:${IMAGE_TAG} --with-registry-auth my-stack_web
                        "
                    """
                }
            }
        }
    }
}