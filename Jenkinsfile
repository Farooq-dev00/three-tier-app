pipeline {
    agent any

    environment {
        AWS_REGION = 'eu-north-1'
        ECR_REGISTRY = '377418125520.dkr.ecr.eu-north-1.amazonaws.com'
        BACKEND_IMAGE = "${ECR_REGISTRY}/three-tier-app-backend"
        FRONTEND_IMAGE = "${ECR_REGISTRY}/three-tier-app-frontend"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        aws ecr get-login-password \
                        | docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t "$BACKEND_IMAGE:$IMAGE_TAG" ./backend
                    docker build -t "$FRONTEND_IMAGE:$IMAGE_TAG" ./frontend
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push "$BACKEND_IMAGE:$IMAGE_TAG"
                    docker push "$FRONTEND_IMAGE:$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    export IMAGE_TAG="$IMAGE_TAG"

                    docker compose down
                    docker compose pull
                    docker compose up -d
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    docker compose ps
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD pipeline failed. Check the Jenkins console log.'
        }
    }
}
