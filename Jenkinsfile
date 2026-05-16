// ============================================================
// SmartPark - Jenkinsfile (Declarative Pipeline)
// WHERE TO PLACE: root of your SmartPark project (same level as Dockerfile)
// HOW TO USE: In Jenkins → New Item → Pipeline → choose "Pipeline script from SCM"
//             Point it to your GitHub repo. Jenkins reads this file automatically.
// ============================================================

pipeline {
    agent any

    // ── Environment Variables ─────────────────────────────────
    environment {
        DOCKERHUB_USERNAME   = 'priyanshusingh18'    // Replace this
        IMAGE_NAME           = 'smartpark'
        IMAGE_TAG            = "${BUILD_NUMBER}"             // e.g., smartpark:42
        DOCKER_IMAGE         = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
        DOCKER_IMAGE_LATEST  = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest"
        K8S_NAMESPACE        = 'smartpark'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')  // Set in Jenkins
        GITHUB_REPO          = 'https://github.com/Priyanshu-Singh1/SmartPark.git'
    }

    // ── Pipeline Options ──────────────────────────────────────
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 15, unit: 'MINUTES')
        timestamps()
    }

    stages {

        // ── STAGE 1: Pull latest code ────────────────────────
        stage('📥 Checkout Code') {
            steps {
                echo "Pulling latest code from GitHub..."
                checkout scm
                sh 'git log --oneline -5'
            }
        }

        // ── STAGE 2: Run basic PHP tests ─────────────────────
        stage('🧪 Test Application') {
            steps {
                echo "Running PHP syntax checks..."
                sh '''
                    # Check PHP syntax on all PHP files
                    find . -name "*.php" -exec php -l {} \\;
                    echo "✅ All PHP syntax checks passed!"
                '''
            }
        }

        // ── STAGE 3: Build Docker image ───────────────────────
        stage('🐳 Build Docker Image') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE}"
                sh '''
                    docker build -t $DOCKER_IMAGE -t $DOCKER_IMAGE_LATEST .
                    docker images | grep smartpark
                '''
            }
        }

        // ── STAGE 4: Push to Docker Hub ───────────────────────
        stage('📦 Push to Docker Hub') {
            steps {
                echo "Pushing image to Docker Hub..."
                sh '''
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                    docker push $DOCKER_IMAGE
                    docker push $DOCKER_IMAGE_LATEST
                    docker logout
                    echo "✅ Image pushed: $DOCKER_IMAGE"
                '''
            }
        }

        // ── STAGE 5: Deploy to Kubernetes ────────────────────
        stage('☸️ Deploy to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes cluster..."
                sh '''
                    # Update image tag in deployment
                    kubectl set image deployment/smartpark-app \
                        smartpark-php=$DOCKER_IMAGE \
                        -n $K8S_NAMESPACE

                    # Wait for rollout to complete
                    kubectl rollout status deployment/smartpark-app \
                        -n $K8S_NAMESPACE \
                        --timeout=300s

                    echo "✅ Deployment successful!"
                '''
            }
        }

        // ── STAGE 6: Verify deployment ────────────────────────
        stage('✅ Verify Deployment') {
            steps {
                sh '''
                    echo "=== Pod Status ==="
                    kubectl get pods -n $K8S_NAMESPACE -l app=smartpark-app

                    echo "=== Service Status ==="
                    kubectl get svc -n $K8S_NAMESPACE

                    echo "=== Deployment Status ==="
                    kubectl get deployment smartpark-app -n $K8S_NAMESPACE
                '''
            }
        }
    }

    // ── Post-pipeline actions ─────────────────────────────────
    post {
        success {
            echo "🎉 SmartPark pipeline completed successfully! Build #${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Pipeline failed at stage. Check logs above."
            // Rollback on failure
            sh '''
                kubectl rollout undo deployment/smartpark-app -n smartpark || true
                echo "⏪ Rolled back to previous version"
            '''
        }
        always {
            // Clean up local Docker images to save disk space
            sh 'docker rmi $DOCKER_IMAGE || true'
        }
    }
}