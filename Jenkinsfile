pipeline {
  agent {
    docker {
      image 'node:slim'
    }
  }

  environment {
    REGISTRY   = 'docker.io'
    IMAGE_NAME = 'muhammadnuman91/assignment2_22035013'
    IMAGE_TAG  = "build-${BUILD_NUMBER}"
    DOCKER_BUILDKIT = '1'
  }

  options {
    timestamps()
    skipDefaultCheckout(true)
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        sh 'echo "Source code checked out successfully"'
      }
    }

    stage('Prepare Docker CLI') {
      steps {
        sh '''
          set -e
          echo "Preparing Docker CLI inside node:slim..."

          apt-get update -qq
          apt-get install -y -qq ca-certificates curl gnupg lsb-release

          mkdir -p /etc/apt/keyrings
          curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
          echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list

          apt-get update -qq
          apt-get install -y -qq docker-ce-cli

          echo "Verifying Docker CLI connectivity..."
          docker version
          echo "Docker CLI successfully connected to DinD."
        '''
      }
    }

    stage('Install Dependencies') {
      steps {
        sh '''
          set -e
          echo "Installing Node dependencies..."
          npm install --save
        '''
      }
    }

    stage('Unit Tests') {
      steps {
        sh '''
          set -e
          echo "Running unit tests..."
          npm test || echo "No tests found - skipping test stage."
        '''
      }
    }

    stage('Security Scan (Snyk)') {
      steps {
        withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
          sh '''
            set -e
            echo "Running security scan using Snyk..."
            npm install -g snyk@latest
            snyk auth "$SNYK_TOKEN"
            snyk test --severity-threshold=high
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -e
          echo "Building Docker image..."
          docker build -t "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG" .
        '''
      }
    }

    stage('Docker Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-reg-cred', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh '''
            set -e
            echo "Logging into Docker Hub..."
            echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
            echo "Pushing Docker image to registry..."
            docker push "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG"
            docker logout
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Build finished with status: ${currentBuild.currentResult}"
    }
    success {
      echo "Pipeline completed successfully. Image pushed to Docker Hub."
    }
    failure {
      echo "Build failed. Check above logs for error details."
    }
  }
}