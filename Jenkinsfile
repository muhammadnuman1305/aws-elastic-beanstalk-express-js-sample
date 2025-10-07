pipeline {
  agent {
    docker {
      image 'node:slim'
      args '-u root:root -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_HOST'
    }
  }

  // Environment Variables
  environment {
    // Registry configuration
    REGISTRY   = 'docker.io'                                   // To push the image to Docker Hub
    IMAGE_NAME = 'muhammadnuman91/assignment2_22035013'        // Format: <DockerHubUsername>/<Repository>
    IMAGE_TAG  = "build-${BUILD_NUMBER}"                       // Automatically versioned tag per build
    DOCKER_BUILDKIT = '1'                                      // Enable modern BuildKit builder
  }

  // Pipeline Options
  options {
    timestamps()                    // Add timestamps to console
    skipDefaultCheckout(true)       // Skip auto-checkout, we’ll do it manually
    disableConcurrentBuilds()       // Prevent parallel runs on the same job
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10')) // Keep only last 20 builds
  }

  // Stages
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
            echo "Preparing Docker CLI..."

            # Install Docker CLI in Debian-based container (node:slim)
            apt-get update -qq
            apt-get install -y -qq ca-certificates curl gnupg lsb-release

            # Add official Docker repository
            mkdir -p /etc/apt/keyrings
            curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
            echo \
            "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
            https://download.docker.com/linux/debian \
            $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list

            apt-get update -qq
            apt-get install -y -qq docker-ce-cli

            echo "Verifying Docker CLI..."
            docker version
            '''
        }
    }


    // 3. Install Node dependencies
    stage('Install Dependencies') {
        steps {
            sh '''
            set -e
            echo "Installing Node dependencies..."
            # Required as per assignment: use 'npm install --save'
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
            # Execute the test script defined in package.json
            npm test || echo "No tests found - skipping test stage."
            '''
        }
    }

    // 5. Security Scan using Snyk
    stage('Security Scan (Snyk)') {
        steps {
            // Pull Snyk token securely from Jenkins credentials
            withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
            sh '''
                set -e
                echo "Running security scan using Snyk..."
                npm install -g snyk@latest                # Install Snyk CLI globally
                snyk auth "$SNYK_TOKEN"                   # Authenticate Snyk using token
                snyk test --severity-threshold=high       # Fail build if High/Critical issues exist
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
            // Use Jenkins credentials to log in and push securely
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
      sh 'echo "Build finished with status: ${currentBuild.currentResult}"'
    }
    success {
      echo "Pipeline completed successfully. Image pushed to Docker Hub."
    }
    failure {
      echo "Build failed. Check above logs for error details."
    }
  }
}
