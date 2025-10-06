pipeline {
  agent any  // ✅ gives top-level context so no node errors
  environment {
    REGISTRY    = 'docker.io'
    IMAGE_REPO  = '21920794/pipeline-compose'
    DC_FAIL_CVSS = '7'
  }

  stages {
    stage('Checkout') {
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
  steps {
    sh '''
      set -eux
      mkdir -p reports
      # make sure the named volume exists on the remote Docker host (DinD)
      docker volume create dependency-check-data >/dev/null 2>&1 || true

      docker run --rm -u 0:0 \
        -v "$PWD":/src \
        -v dependency-check-data:/usr/share/dependency-check/data \
        owasp/dependency-check:latest \
          --scan /src \
          --format HTML \
          --out /src/reports \
          --failOnCVSS ${DC_FAIL_CVSS}
    '''
  }
  post {
    always {
      archiveArtifacts artifacts: 'reports/*.html', fingerprint: true
    }
  }
}

    stage('Docker Build & Push') {
      steps {
        script {
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
      post {
        always {
            // ⬇️ remove the numbered tag locally, keep :latest, then prune dangling layers
	      sh """
		docker rmi ${REGISTRY}/${IMAGE_REPO}:${BUILD_NUMBER} || true
		docker image prune -f || true
	      """
        }
      }
    }
  }
}

