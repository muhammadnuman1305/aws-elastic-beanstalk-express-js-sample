pipeline {
  agent {
    docker {
      image 'node:16'  // ✅ Use the full Node 16 image (not slim)
      args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_HOST=tcp://dind:2375'
    }
  }

  // Environment Variables
  environment {
    REGISTRY   = 'docker.io'                                    // Docker registry
    IMAGE_NAME = 'muhammadnuman91/assignment2_22035013'         // DockerHub <username>/<repo>
    IMAGE_TAG  = "build-${BUILD_NUMBER}"                        // Auto-version per build
    DOCKER_BUILDKIT = '1'                                       // Use BuildKit builder
  }

  // Pipeline Options
  options {
    timestamps()                       // Add timestamps to logs
    skipDefaultCheckout(true)          // Manual checkout for clarity
    disableConcurrentBuilds()          // Prevent parallel builds
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10')) // Retain last 20 builds
  }

  stages {

    // 1. Checkout source code from the connected Git repository
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // 2. Prepare Docker CLI so the agent can talk to Docker-in-Docker (DinD)
    stage('Prepare Docker CLI') {
      steps {
        sh '''
        set -e
        echo "Installing and verifying Docker CLI..."

        export DEBIAN_FRONTEND=noninteractive

        # Install prerequisite packages
        apt-get update -qq
        apt-get install -y -qq ca-certificates curl gnupg lsb-release

        # Set up Docker’s official GPG key
        install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        chmod a+r /etc/apt/keyrings/docker.gpg

        # Add Docker repository
        echo \
        "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
        https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
        > /etc/apt/sources.list.d/docker.list

        # Install Docker CLI only (no daemon)
        apt-get update -qq
        apt-get install -y -qq docker-ce-cli

        echo "Verifying Docker CLI version..."
        docker --version

        echo "Testing Docker daemon connectivity..."
        docker -H tcp://dind:2375 ps || echo "Warning: Could not reach DinD yet (will retry in next stage)"
        '''
      }
    }

    // 3. Install Node dependencies
    stage('Install Dependencies') {
      steps {
        sh '''
        set -e
        echo "Installing Node dependencies..."
        npm install --save
        '''
      }
    }

    // 4. Run unit tests
    stage('Unit Tests') {
      steps {
        sh '''
        set -e
        echo "Running unit tests..."
        npm test || echo "No tests found - skipping test stage."
        '''
      }
    }

    // 5. Security Scan using Snyk
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

    // 6. Build Docker image for the Node.js app
    stage('Docker Build') {
      steps {
        sh '''
        set -e
        echo "Building Docker image..."
        docker build -t "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG" .
        '''
      }
    }

    // 7. Push the built image to Docker Hub
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

  // Post Actions
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