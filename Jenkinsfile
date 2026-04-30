pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "sanjay5raj/cicd-demo"
    // Remove the incorrect KUBECONFIG path
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

    stage('Setup Kubernetes Config') {
      steps {
        script {
          // Method 1: Using kubeconfig file from Jenkins credentials
          withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
            bat """
              set KUBECONFIG=${KUBECONFIG_FILE}
              kubectl config view
              kubectl cluster-info
            """
          }
        }
      }
    }

    stage('Update Kubernetes Deployment') {
      steps {
        script {
          withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
            bat """
              set KUBECONFIG=${KUBECONFIG_FILE}
              powershell -Command "(Get-Content k8s\\deployment.yaml) -replace 'image:.*', 'image: ${DOCKER_IMAGE}:${BUILD_NUMBER}' | Set-Content k8s\\deployment.yaml"
              kubectl apply -f k8s/deployment.yaml
              kubectl rollout status deployment/cicd-demo
            """
          }
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        script {
          withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
            bat """
              set KUBECONFIG=${KUBECONFIG_FILE}
              kubectl get pods
              kubectl get deployment cicd-demo
            """
          }
        }
      }
    }

  }

  post {
    success {
      echo 'Pipeline succeeded! App deployed successfully.'
      bat 'docker system prune -f'
    }
    failure {
      echo 'Pipeline failed. Check the logs above for details.'
    }
    always {
      bat 'docker logout'
    }
  }
}