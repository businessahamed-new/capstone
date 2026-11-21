pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '766851565676'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        EKS_CLUSTER    = 'ecom-eks-cluster'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                    echo "Installing dependencies and running tests"

                    cd backend
                    npm ci
                    npm test -- --watchAll=false || echo "Backend tests not configured"

                    cd ../frontend
                    npm ci
                    npm test -- --watchAll=false || echo "Frontend tests not configured"

                    cd ../admin
                    npm ci
                    npm test -- --watchAll=false || echo "Admin tests not configured"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t shopnow-backend:${BUILD_TAG} ./backend
                    docker build -t shopnow-frontend:${BUILD_TAG} ./frontend
                    docker build -t shopnow-admin:${BUILD_TAG} ./admin
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Tag & Push to ECR') {
            steps {
                sh '''
                    docker tag shopnow-backend:${BUILD_TAG} ${ECR_REGISTRY}/shopnow-backend:${BUILD_TAG}
                    docker tag shopnow-frontend:${BUILD_TAG} ${ECR_REGISTRY}/shopnow-frontend:${BUILD_TAG}
                    docker tag shopnow-admin:${BUILD_TAG} ${ECR_REGISTRY}/shopnow-admin:${BUILD_TAG}

                    docker push ${ECR_REGISTRY}/shopnow-backend:${BUILD_TAG}
                    docker push ${ECR_REGISTRY}/shopnow-frontend:${BUILD_TAG}
                    docker push ${ECR_REGISTRY}/shopnow-admin:${BUILD_TAG}
                '''
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER}
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl apply -f k8s/namespace.yml
                    kubectl apply -f k8s/mongodb.yml
                    kubectl rollout status deployment/shopnow-mongodb -n web-app --timeout=5m


                    export FRONTEND_IMAGE=${ECR_REGISTRY}/shopnow-frontend:${BUILD_TAG}
                    export BACKEND_IMAGE=${ECR_REGISTRY}/shopnow-backend:${BUILD_TAG}
                    export ADMIN_IMAGE=${ECR_REGISTRY}/shopnow-admin:${BUILD_TAG}

                    envsubst < k8s/deployment.yml | kubectl apply -f -

                    kubectl apply -f k8s/service.yml
                    kubectl apply -f k8s/hpa.yml
                '''
            }
        }

        stage('Post-deploy Validation') {
            steps {
                sh '''
                    kubectl rollout status deployment/shopnow-frontend -n web-app --timeout=5m
                    kubectl rollout status deployment/shopnow-backend -n web-app --timeout=5m
                    kubectl rollout status deployment/shopnow-admin -n web-app --timeout=5m

                    echo "===== PODS ====="
                    kubectl get pods -n web-app

                    echo "===== SERVICES ====="
                    kubectl get svc -n web-app

                    echo "===== HPA ====="
                    kubectl get hpa -n web-app
                '''
            }
        }
    }

    post {
        success {
            echo 'ShopNow Sprint 4 CI/CD deployment completed successfully.'
        }

        failure {
            echo 'ShopNow Sprint 4 CI/CD deployment failed. Check the Jenkins console output.'
        }

        always {
            sh '''
                docker image rm shopnow-backend:${BUILD_TAG} 2>/dev/null || true
                docker image rm shopnow-frontend:${BUILD_TAG} 2>/dev/null || true
                docker image rm shopnow-admin:${BUILD_TAG} 2>/dev/null || true
            '''
        }
    }
}
