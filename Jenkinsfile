pipeline {
    agent any

    environment {
        // Docker
        DOCKER_REGISTRY = 'ajiththomas10'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // Kubernetes (VERY IMPORTANT on Windows)
        KUBECONFIG = 'C:\\ProgramData\\Jenkins\\.kube\\config'
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/ajiththomas333/k85.git',
                    credentialsId: 'git'
            }
        }

        stage('Install Backend Dependencies') {
            steps {
                dir('backend') {
                    bat 'npm install'
                }
            }
        }

        stage('Install Frontend Dependencies') {
            steps {
                dir('frontend') {
                    bat 'npm install'
                }
            }
        }

        stage('Build & Push Docker Images') {
            parallel {

                stage('Build & Push Backend Image') {
                    steps {
                        script {
                            dir('backend') {
                                def backendImage = docker.build(
                                    "${DOCKER_REGISTRY}/mern-backend:${IMAGE_TAG}"
                                )
                                docker.withRegistry(
                                    'https://registry.hub.docker.com',
                                    'dockercred'
                                ) {
                                    backendImage.push()
                                    backendImage.push('latest')
                                }
                            }
                        }
                    }
                }

                stage('Build & Push Frontend Image') {
                    steps {
                        script {
                            dir('frontend') {
                                def frontendImage = docker.build(
                                    "${DOCKER_REGISTRY}/mern-frontend:${IMAGE_TAG}"
                                )
                                docker.withRegistry(
                                    'https://registry.hub.docker.com',
                                    'dockercred'
                                ) {
                                    frontendImage.push()
                                    frontendImage.push('latest')
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Verify Kubernetes Connectivity') {
            steps {
                echo '🔍 Verifying Kubernetes API access from Jenkins...'
                bat '''
                echo ===== kubeconfig being used =====
                kubectl config view --minify

                echo ===== cluster-info =====
                kubectl cluster-info

                echo ===== nodes =====
                kubectl get nodes -o wide
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo '🚀 Deploying application to Kubernetes...'

                bat """
                powershell -Command "(Get-Content k8s/backend-deployment.yaml) `
                  -replace 'your-registry/mern-backend:latest','${DOCKER_REGISTRY}/mern-backend:${IMAGE_TAG}' `
                  | Set-Content k8s/backend-deployment.yaml"

                powershell -Command "(Get-Content k8s/frontend-deployment.yaml) `
                  -replace 'your-registry/mern-frontend:latest','${DOCKER_REGISTRY}/mern-frontend:${IMAGE_TAG}' `
                  | Set-Content k8s/frontend-deployment.yaml"
                """

                bat '''
                kubectl apply -f k8s/
                kubectl rollout status deployment/backend-deployment
                kubectl rollout status deployment/frontend-deployment
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '✅ Verifying Kubernetes deployment status...'
                bat '''
                kubectl get pods -o wide
                kubectl get services

                kubectl wait --for=condition=ready pod -l app=backend --timeout=300s
                kubectl wait --for=condition=ready pod -l app=frontend --timeout=300s
                '''
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
        always {
            bat 'docker system prune -f'
        }
    }
}
