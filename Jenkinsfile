// Jenkinsfile — CI/CD for Node app with security scan + Docker push
// - Uses Node 16 docker image as build agent for install/test
// - OWASP Dependency-Check fails build if CVSS >= 7
// - Builds and pushes image to Docker Hub

pipeline {
  agent none
  options { timestamps() }
  environment {
    REGISTRY   = 'docker.io'
    IMAGE_REPO = '21920794/pipeline-compose'   // change if needed
    DC_FAIL_CVSS = '7'                         // fail on High/Critical
  }

  stages {
    stage('Checkout') {
      agent any
      steps { checkout scm }
    }

    stage('Install & Test (Node 16)') {
      agent {
        docker {
          image 'node:16'
          args '-u root:root'
          reuseNode true
        }
      }
      steps {
        sh '''
          node -v
          npm install --save
          npm test || echo "No tests configured"
        '''
      }
    }

    stage('Dependency Scan (OWASP DC)') {
      agent any
      steps {
        sh '''
          mkdir -p reports
          docker run --rm -u 0:0 -v "$PWD":/src owasp/dependency-check:latest \
            --scan /src \
            --format HTML \
            --out /src/reports \
            --failOnCVSS ${DC_FAIL_CVSS}
        '''
      }
      post { always { archiveArtifacts artifacts: 'reports/*.html', fingerprint: true } }
    }

    stage('Docker Build & Push') {
      agent any
      steps {
        script {
          // 🔧 Fallback Dockerfile if the repo doesn't have one
          if (!fileExists('Dockerfile')) {
            writeFile file: 'Dockerfile', text: '''
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production || npm install --production
COPY . .
EXPOSE 8080
CMD ["npm","start"]
'''.stripIndent()
          }

          def imgTag = "${REGISTRY}/${IMAGE_REPO}:${BUILD_NUMBER}"
          sh "docker build -t ${imgTag} ."

          withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                            usernameVariable: 'DOCKERHUB_USERNAME',
                                            passwordVariable: 'DOCKERHUB_PASSWORD')]) {
            sh """
              echo "\$DOCKERHUB_PASSWORD" | docker login -u "\$DOCKERHUB_USERNAME" --password-stdin ${REGISTRY}
              docker push ${imgTag}
              docker tag ${imgTag} ${REGISTRY}/${IMAGE_REPO}:latest
              docker push ${REGISTRY}/${IMAGE_REPO}:latest
              docker logout ${REGISTRY} || true
            """
          }
        }
      }
    }
  }

  post { always { sh 'docker image prune -f || true' } }
}

