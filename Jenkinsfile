pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "sanjay5raj/cicd-demo"
    // Use forward slashes for Windows compatibility
    DOCKER_BUILDKIT = "1"
  }

  stages {

    stage('Clone') {
      steps {
        git branch: 'main', 
            url: 'https://github.com/dm-gamer/cicd-demo.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        // Using Windows batch command with proper variable expansion
        bat """
          docker build -t ${DOCKER_IMAGE}:latest .
          docker tag ${DOCKER_IMAGE}:latest ${DOCKER_IMAGE}:${BUILD_NUMBER}
        """
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          bat """
            echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin https://index.docker.io/v1/
            docker push ${DOCKER_IMAGE}:latest
            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
          """
        }
      }
    }

    stage('Update Kubernetes Deployment') {
      steps {
        // Update the image tag in deployment.yaml before applying
        bat """
          powershell -Command "(Get-Content k8s/deployment.yaml) -replace 'image: .*', 'image: ${DOCKER_IMAGE}:${BUILD_NUMBER}' | Set-Content k8s/deployment.yaml"
          kubectl apply -f k8s/deployment.yaml
          kubectl rollout status deployment/cicd-demo
        """
      }
    }

    stage('Verify Deployment') {
      steps {
        bat 'kubectl get pods'
        bat 'kubectl get deployment cicd-demo'
      }
    }

  }

  post {
    success { 
      echo 'Pipeline succeeded! App deployed successfully.'
      // Optional: Clean up old images
      bat 'docker system prune -f'
    }
    failure { 
      echo 'Pipeline failed. Check the logs above for details.'
    }
    always {
      // Optional: Logout from Docker Hub
      bat 'docker logout'
    }
  }
}