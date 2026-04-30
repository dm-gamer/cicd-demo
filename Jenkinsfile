pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "sanjay5raj/cicd-demo"
  }

  stages {

    stage('Clone') {
      steps {
        git branch: 'main', url: 'https://github.com/dm-gamer/cicd-demo.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        bat 'docker build -t %DOCKER_IMAGE%:latest .'
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          bat 'echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin'
          bat 'docker push %DOCKER_IMAGE%:latest'
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        bat 'kubectl apply -f k8s/deployment.yaml'
        bat 'kubectl rollout restart deployment/cicd-demo'
      }
    }

  }

  post {
    success { echo 'Pipeline succeeded! App deployed.' }
    failure { echo 'Pipeline failed. Check logs.' }
  }
}