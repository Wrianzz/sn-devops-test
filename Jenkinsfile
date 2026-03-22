pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "wrianzz" 
        GITHUB_USER    = "wrianzz"
        GITOPS_REPO    = "github.com/wrianzz/sn-devops-test.git"

        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'main', url: 'https://github.com/sncyber-ops/bookslib.git'
            }
        }

        // --- STAGE 1: BUILD IMAGE ---
        stage('Build Docker Images') {
            steps {
                script {
                    def services = ['frontend', 'auth-service', 'books-service', 'reviews-service']
                    
                    echo "Mengubah file .env frontend ke IP Cluster..."
                    sh """
                        echo 'VITE_AUTH_API=http://192.168.56.20:30081' > frontend/.env
                        echo 'VITE_BOOKS_API=http://192.168.56.20:30082' >> frontend/.env
                        echo 'VITE_REVIEWS_API=http://192.168.56.20:30083' >> frontend/.env
                    """
                    
                    for (service in services) {
                        def imageName = "${DOCKERHUB_USER}/bookslib-${service}"
                        echo "=== Membangun Image untuk ${service} ==="
                        sh "docker build -t ${imageName}:${IMAGE_TAG} ./${service}"
                    }
                }
            }
        }

        // --- STAGE 2: SHIFT-LEFT SECURITY (TRIVY) ---
        stage('Shift-Left Security Scan (Trivy)') {
            steps {
                script {
                    def services = ['frontend', 'auth-service', 'books-service', 'reviews-service']
                    
                    for (service in services) {
                        def imageName = "${DOCKERHUB_USER}/bookslib-${service}:${IMAGE_TAG}"
                        echo "=== Scanning Vulnerabilities for ${imageName} ==="
                        sh "trivy image --severity CRITICAL --exit-code 0 ${imageName}" 
                    }
                }
            }
        }

        // --- STAGE 3: PUSH KE REGISTRY ---
        stage('Push to Registry') {
            steps {
                script {
                    def services = ['frontend', 'auth-service', 'books-service', 'reviews-service']
                    
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        
                        for (service in services) {
                            def imageName = "${DOCKERHUB_USER}/bookslib-${service}"
                            echo "=== Push Image ${service} ke Registry ==="
                            
                            sh "docker push ${imageName}:${IMAGE_TAG}"
                            sh "docker tag ${imageName}:${IMAGE_TAG} ${imageName}:latest"
                            sh "docker push ${imageName}:latest"
                        }
                    }
                }
            }
        }

        // --- STAGE 4: UPDATE GITOPS ---
        stage('Update GitOps Manifests') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        sh """
                            git clone https://${GIT_USER}:${GIT_PASS}@${GITOPS_REPO} gitops-dir
                            cd gitops-dir
                            
                            sed -i 's|image: ${DOCKERHUB_USER}/bookslib-frontend:.*|image: ${DOCKERHUB_USER}/bookslib-frontend:${IMAGE_TAG}|g' k8s/frontend-deployment.yaml
                            sed -i 's|image: ${DOCKERHUB_USER}/bookslib-auth-service:.*|image: ${DOCKERHUB_USER}/bookslib-auth-service:${IMAGE_TAG}|g' k8s/auth-deployment.yaml
                            sed -i 's|image: ${DOCKERHUB_USER}/bookslib-books-service:.*|image: ${DOCKERHUB_USER}/bookslib-books-service:${IMAGE_TAG}|g' k8s/books-deployment.yaml
                            sed -i 's|image: ${DOCKERHUB_USER}/bookslib-reviews-service:.*|image: ${DOCKERHUB_USER}/bookslib-reviews-service:${IMAGE_TAG}|g' k8s/reviews-deployment.yaml
                            
                            git config user.email "jenkins-bot@wrianzz.me"
                            git config user.name "Jenkins CI Bot"
                            
                            git add k8s/
                            git commit -m "Auto-update all microservices to Build #${BUILD_NUMBER}"
                            git push origin main
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
            sh "docker logout"
        }
    }
}
