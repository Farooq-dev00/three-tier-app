pipeline {
    agent any

    environment {
        AWS_REGION = 'eu-north-1'
        ECR_REGISTRY = '377418125520.dkr.ecr.eu-north-1.amazonaws.com'

        BACKEND_IMAGE = "${ECR_REGISTRY}/three-tier-app-backend"
        FRONTEND_IMAGE = "${ECR_REGISTRY}/three-tier-app-frontend"
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
                    } else {
                        env.ENV_NAME = 'main'
                    }

                    env.IMAGE_TAG = "${env.ENV_NAME}-${env.BUILD_NUMBER}"

                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Environment: ${env.ENV_NAME}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
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
                    docker build \
                        -t "$BACKEND_IMAGE:$IMAGE_TAG" \
                        ./backend

                    docker build \
                        -t "$FRONTEND_IMAGE:$IMAGE_TAG" \
                        ./frontend
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
