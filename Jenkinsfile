pipeline {
    agent any
    environment {
        DOCKER_IMAGE = "phegley/delivery"
        KUBE_CONFIG = "/home/ubuntu/.kube/config"
        AWS_REGION = "us-east-1"
        EKS_CLUSTER_NAME = "order-cluster"
        
    }
    stages {
        stage('Checkout Code') {
            steps {
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/ericphegley/delivery-project.git']])
            }
        }
        stage('Build & Test') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    // Ensure Docker BuildKit is enabled
                    env.DOCKER_BUILDKIT = '1'

                    // Build Docker image
                    sh 'docker build -t $DOCKER_IMAGE:latest .'

                    // Login to DockerHub
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD'
                    }

                    // Push the Docker image to DockerHub
                    sh 'docker push $DOCKER_IMAGE:latest'
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',  // Jenkins credential ID for AWS keys
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]) {
                        sh """
                            # Configure kubectl for EKS
                            aws eks update-kubeconfig \
                              --region ${AWS_REGION} \
                              --name ${EKS_CLUSTER_NAME}
        
                            # Apply manifests
                            kubectl apply -f k8s-manifests/deployment.yaml
                            kubectl apply -f k8s-manifests/service.yaml
                        """
                    }
                }
            }
        }
    }
}
