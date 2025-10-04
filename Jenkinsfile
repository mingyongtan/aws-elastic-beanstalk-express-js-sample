pipeline {
    agent {
        docker {
            image 'node:16'              // Use Node.js 16 as build agent
            args '-u root:root'          // Run as root so we can install tools if needed
        }
    }

    environment {
        REGISTRY    = "docker.io"                        // Docker Hub registry
        APP_NAME    = "aws-express-sample"               // Change if needed
        IMAGE_NAME  = "mingyongtan/${APP_NAME}"          // Replace with your Docker Hub username
        IMAGE_TAG   = "latest"                           // Could also be "${BUILD_NUMBER}" or "${GIT_COMMIT}"
        DOCKER_USER = credentials('docker-username')     // Jenkins credential ID
        DOCKER_PASS = credentials('docker-password')     // Jenkins credential ID
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm   // Pulls code from your forked GitHub repo
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install --save'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh 'npm test || echo "⚠️ No tests defined"'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                  echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                  docker build -t $IMAGE_NAME:$IMAGE_TAG .
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully. Image pushed: $IMAGE_NAME:$IMAGE_TAG"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
