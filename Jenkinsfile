pipeline {
    agent any

    environment {
        IMAGE_NAME = "shanthigowd/trendapp"
        IMAGE_TAG = "${BUILD_NUMBER}"
        AWS_REGION = "ap-south-1"
        EKS_CLUSTER = "trend-task-shanthi"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Login to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh """
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Verify AWS Identity') {
            steps {
                sh '''
                    echo "===== AWS Identity ====="
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Update kubeconfig') {
            steps {
                sh """
                    aws eks update-kubeconfig \
                        --region ${AWS_REGION} \
                        --name ${EKS_CLUSTER}
                """
            }
        }

        stage('Verify Cluster Access') {
            steps {
                sh '''
                    echo "===== Current Context ====="
                    kubectl config current-context

                    echo "===== Kubectl Version ====="
                    kubectl version --client

                    echo "===== Cluster Nodes ====="
                    kubectl get nodes
                '''
            }
        }

        stage('Update Deployment Image') {
            steps {
                sh """
                    sed -i 's|image:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yml

                    echo "===== Updated deployment.yml ====="

                    cat deployment.yml
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                    kubectl apply -f ingress.yml
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== Pods ====="
                    kubectl get pods -o wide

                    echo "===== Services ====="
                    kubectl get svc

                    echo "===== Deployments ====="
                    kubectl get deployments

                    kubectl rollout status deployment/trend-app
                '''
            }
        }
    }

    post {

        success {
            echo "=================================="
            echo "Deployment Successful"
            echo "=================================="
        }

        failure {
            echo "=================================="
            echo "Deployment Failed"
            echo "=================================="
        }

        always {
            cleanWs()
        }
    }
}
