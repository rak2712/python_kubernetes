pipeline {
    agent any

    environment {
        IMAGE_NAME = "rakshith3/hello-world-python"
        SERVICE_PORT = "5006"
        GIT_COMMIT = ''
        IMAGE_TAG = ''
        INGRESS_HOSTNAME = "hello-python.example.com"  // Replace with your domain
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        retry(2)
    }

    stages {
        stage('Checkout Source') {
            steps {
                // Use checkout with CloneOption for shallow clone instead of git step
                checkout([$class: 'GitSCM',
                          branches: [[name: 'refs/heads/main']],
                          userRemoteConfigs: [[url: 'https://github.com/rak2712/hello-world-python.git']],
                          extensions: [[$class: 'CloneOption', depth: 1, shallow: true]]])
            }
            post {
                success {
                    script {
                        GIT_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        IMAGE_TAG = "${GIT_COMMIT}"
                        echo "Using git commit SHA as image tag: ${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image ${IMAGE_NAME}:${IMAGE_TAG} and latest"
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .
                """
            }
        }

        stage('Security Scan') {
            steps {
                echo "Scanning Docker image for vulnerabilities"
                sh """
                    trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG} || true
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                echo "Pushing Docker images to registry"
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Verify Kubernetes Access') {
            steps {
                echo "Verifying Kubernetes cluster access"
                retry(3) {
                    sh 'kubectl get nodes'
                }
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                echo "Updating image tag in deployment manifest"
                sh """
                    sed -i 's|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|g' k8s/deployment.yaml
                """
            }
        }

        stage('Validate Kubernetes Manifests') {
            steps {
                echo "Validating manifests with dry-run"
                sh 'kubectl apply --dry-run=client -f k8s/deployment.yaml'
                sh 'kubectl apply --dry-run=client -f k8s/service.yaml'
                sh 'kubectl apply --dry-run=client -f k8s/ingress.yaml'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "Applying Kubernetes manifests"
                sh 'kubectl apply -f k8s/deployment.yaml'
                sh 'kubectl apply -f k8s/service.yaml'
                sh 'kubectl apply -f k8s/ingress.yaml'
            }
        }

        stage('Show Access Information') {
            steps {
                echo "Your app is accessible via the Ingress URL:"
                echo "http://${INGRESS_HOSTNAME}"
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace and pruning Docker images"
            cleanWs()
            sh 'docker image prune -f || true'
        }
        failure {
            mail to: 'team@example.com',
                 subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Build failed. Check logs: ${env.BUILD_URL}"
        }
    }
}
