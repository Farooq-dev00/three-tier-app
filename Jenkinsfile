pipeline {
    agent any

    environment {
        AWS_REGION = 'eu-north-1'
        ECR_REGISTRY = '377418125520.dkr.ecr.eu-north-1.amazonaws.com'

        BACKEND_IMAGE = "${ECR_REGISTRY}/three-tier-app-backend"
        FRONTEND_IMAGE = "${ECR_REGISTRY}/three-tier-app-frontend"

        DOCKERHUB_BACKEND = 'umarfarooq00/three-tier-app-backend'
        DOCKERHUB_FRONTEND = 'umarfarooq00/three-tier-app-frontend'

        GHCR_BACKEND = 'ghcr.io/farooq-dev00/three-tier-app-backend'
        GHCR_FRONTEND = 'ghcr.io/farooq-dev00/three-tier-app-frontend'
    }

    stages {

        stage('Check Branch') {
            steps {
                script {
                    if (!(env.BRANCH_NAME in ['dev', 'stg', 'prod'])) {
                        currentBuild.result = 'NOT_BUILT'
                        error("This pipeline is only configured for dev, stg, and prod branches.")
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Environment') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'dev') {
                        env.ENV_NAME = 'dev'
                    } else if (env.BRANCH_NAME == 'stg') {
                        env.ENV_NAME = 'stg'
                    } else if (env.BRANCH_NAME == 'prod') {
                        env.ENV_NAME = 'prod'
                    }

                    env.IMAGE_TAG = "${env.ENV_NAME}-${env.BUILD_NUMBER}"

                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Environment: ${env.ENV_NAME}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Login to Registries') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    ),
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    ),
                    usernamePassword(
                        credentialsId: 'ghcr-credentials',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_TOKEN'
                    )
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        echo "Logging into Amazon ECR..."
                        aws ecr get-login-password \
                        | docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"

                        echo "Logging into Docker Hub..."
                        echo "$DOCKERHUB_TOKEN" \
                        | docker login \
                        --username "$DOCKERHUB_USER" \
                        --password-stdin

                        echo "Logging into GitHub Container Registry..."
                        echo "$GHCR_TOKEN" \
                        | docker login \
                        ghcr.io \
                        --username "$GHCR_USER" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    echo "Building backend image..."

                    docker build \
                        -t "$BACKEND_IMAGE:$IMAGE_TAG" \
                        ./backend

                    echo "Building frontend image..."

                    docker build \
                        -t "$FRONTEND_IMAGE:$IMAGE_TAG" \
                        ./frontend
                '''
            }
        }

        stage('Tag Images') {
            steps {
                sh '''
                    echo "Tagging backend image..."

                    docker tag \
                        "$BACKEND_IMAGE:$IMAGE_TAG" \
                        "$DOCKERHUB_BACKEND:$IMAGE_TAG"

                    docker tag \
                        "$BACKEND_IMAGE:$IMAGE_TAG" \
                        "$GHCR_BACKEND:$IMAGE_TAG"

                    echo "Tagging frontend image..."

                    docker tag \
                        "$FRONTEND_IMAGE:$IMAGE_TAG" \
                        "$DOCKERHUB_FRONTEND:$IMAGE_TAG"

                    docker tag \
                        "$FRONTEND_IMAGE:$IMAGE_TAG" \
                        "$GHCR_FRONTEND:$IMAGE_TAG"
                '''
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    echo "Pushing backend to ECR..."
                    docker push "$BACKEND_IMAGE:$IMAGE_TAG"

                    echo "Pushing frontend to ECR..."
                    docker push "$FRONTEND_IMAGE:$IMAGE_TAG"

                    echo "Pushing backend to Docker Hub..."
                    docker push "$DOCKERHUB_BACKEND:$IMAGE_TAG"

                    echo "Pushing frontend to Docker Hub..."
                    docker push "$DOCKERHUB_FRONTEND:$IMAGE_TAG"

                    echo "Pushing backend to GHCR..."
                    docker push "$GHCR_BACKEND:$IMAGE_TAG"

                    echo "Pushing frontend to GHCR..."
                    docker push "$GHCR_FRONTEND:$IMAGE_TAG"
                '''
            }
        }

        stage('Deploy') {
            when {
                expression {
                    return env.BRANCH_NAME in ['dev', 'stg', 'prod']
                }
            }

            steps {
                sh '''
                    export IMAGE_TAG="$IMAGE_TAG"

                    docker compose \
                        -p "three-tier-${ENV_NAME}" \
                        down

                    docker compose \
                        -p "three-tier-${ENV_NAME}" \
                        pull

                    docker compose \
                        -p "three-tier-${ENV_NAME}" \
                        up -d
                '''
            }
        }

        stage('Verify') {
            when {
                expression {
                    return env.BRANCH_NAME in ['dev', 'stg', 'prod']
                }
            }

            steps {
                sh '''
                    docker compose \
                        -p "three-tier-${ENV_NAME}" \
                        ps
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully for ${BRANCH_NAME}"
        }

        failure {
            echo "CI/CD pipeline failed for ${BRANCH_NAME}. Check the Jenkins console log."
        }
    }
}
