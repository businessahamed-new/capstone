pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '766851565676'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker build -t shopnow-backend:latest ./backend'
                sh 'docker build -t shopnow-frontend:latest ./frontend'
                sh 'docker build -t shopnow-admin:latest ./admin'
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION | \
                    docker login --username AWS --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Tag Docker Images') {
            steps {
                sh '''
                    docker tag shopnow-backend:latest $ECR_REGISTRY/shopnow-backend:latest
                    docker tag shopnow-frontend:latest $ECR_REGISTRY/shopnow-frontend:latest
                    docker tag shopnow-admin:latest $ECR_REGISTRY/shopnow-admin:latest
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push $ECR_REGISTRY/shopnow-backend:latest
                    docker push $ECR_REGISTRY/shopnow-frontend:latest
                    docker push $ECR_REGISTRY/shopnow-admin:latest
                '''
            }
        }
    }
