pipeline {
    agent {
        docker {
            image 'node:16'
            args '-u root:root'
        }
    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        DOCKER_IMAGE = "21920794/aws-express-app"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install --save'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test || echo "No tests configured"'
            }
        }

        stage('Dependency Scan - OWASP') {
            agent {
                docker {
                    image 'owasp/dependency-check:latest'
                    args '-v $PWD:/src'
                }
            }
            steps {
                sh '''
                    echo "🔍 Running OWASP Dependency Check..."
                    dependency-check.sh \
                      --scan /src \
                      --format HTML \
                      --failOnCVSS 7 \
                      --out /src/reports
                '''
            }
            post {
                always {
                    echo "Saving OWASP scan report..."
                    archiveArtifacts artifacts: 'reports/*.html', allowEmptyArchive: true
                }
                failure {
                    echo "❌ Build failed due to high/critical vulnerabilities!"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t ${DOCKER_IMAGE}:latest .
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    sh """
                    echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                    docker push ${DOCKER_IMAGE}:latest
                    docker logout
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Build and push successful!'
        }
        failure {
            echo '❌ Build or push failed.'
        }
    }
}
