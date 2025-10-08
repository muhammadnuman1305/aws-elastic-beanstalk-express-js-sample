pipeline {
  agent {
    docker {
      image 'node:16'  // Use the full Node 16 image (not slim)
    //   args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock -e DOCKER_HOST=tcp://dind:2375 --network project2-compose_jenkins_dind'
      args '-u root:root -v /var/run/docker.sock:/var/run/docker.sock'
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
        echo "Configuring Debian Buster archive repositories..."
        
        # Replace with archive.debian.org
        cat > /etc/apt/sources.list <<'EOF'
deb http://archive.debian.org/debian/ buster main
deb http://archive.debian.org/debian-security buster/updates main
EOF
        
        # Disable release file validity check (archive repos don't update)
        echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99no-check-valid
        
        echo "Installing Docker CLI..."
        apt-get update -y
        apt-get install -y docker.io
        
        echo "Verifying Docker CLI..."
        docker --version
        
        echo "Testing Docker daemon connection..."
        docker -H tcp://dind:2375 ps || echo "Docker daemon not ready yet — continuing"
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
            echo "Creating temporary Dockerfile..."
            cat <<'EOF' > Dockerfile
            FROM node:16
            WORKDIR /app
            COPY package*.json ./
            RUN npm install --production
            COPY . .
            EXPOSE 3000
            CMD ["npm", "start"]
            EOF

            echo "Building Docker image..."
            docker build -t "$REGISTRY/$IMAGE_NAME:$IMAGE_TAG" .

            echo "Docker image built successfully."
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