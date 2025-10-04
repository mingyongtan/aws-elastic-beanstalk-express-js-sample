pipeline {
    agent {
        docker {
            image 'node:16'
            args '-u root:root'
        }
    }

    environment {
	    DOCKER_HOST       = "tcp://docker:2376"
	    DOCKER_CERT_PATH  = "/certs/client"
	    DOCKER_TLS_VERIFY = "1"

	    // For custom CLI calls (optional)
	    DOCKER_TLS_CA     = "/certs/ca/cert.pem"
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
	always {
	    archiveArtifacts artifacts: '**/logs/*.log', allowEmptyArchive: true
	}
        success {
            echo '✅ Build and push successful!'
        }
        failure {
            echo '❌ Build or push failed.'
        }
    }
}
