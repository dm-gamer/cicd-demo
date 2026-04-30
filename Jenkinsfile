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
        sh 'docker build -t $DOCKER_IMAGE:latest .'
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
          sh 'docker push $DOCKER_IMAGE:latest'
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh 'kubectl apply -f k8s/deployment.yaml'
        sh 'kubectl rollout restart deployment/cicd-demo'
      }
    }

  }

  post {
    success { echo 'Pipeline succeeded! App deployed.' }
    failure { echo 'Pipeline failed. Check logs.' }
  }
}