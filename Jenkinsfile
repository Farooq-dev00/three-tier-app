pipeline {
    agent any

    environment {
        REGISTRY = 'ghcr.io'
        IMAGE_PREFIX = 'ghcr.io/farooq-dev00/three-tier-app'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t ${IMAGE_PREFIX}-backend:${BUILD_NUMBER} ./backend
                    docker build -t ${IMAGE_PREFIX}-frontend:${BUILD_NUMBER} ./frontend
                '''
            }
        }

        stage('Login to GHCR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'ghcr',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_TOKEN'
                    )
                ]) {
                    sh '''
                        echo "$GHCR_TOKEN" | docker login ghcr.io \
                        -u "$GHCR_USER" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    docker push ${IMAGE_PREFIX}-backend:${BUILD_NUMBER}
                    docker push ${IMAGE_PREFIX}-frontend:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    export IMAGE_TAG=${BUILD_NUMBER}

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
            echo 'CI/CD pipeline failed.'
        }
    }
}
