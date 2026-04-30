pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "sanjay5raj/cicd-demo"
    KUBECONFIG = "C:\\ProgramData\\Jenkins\\.jenkins\\config"
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
        bat 'docker build -t %DOCKER_IMAGE%:latest .'
        bat 'docker tag %DOCKER_IMAGE%:latest %DOCKER_IMAGE%:%BUILD_NUMBER%'
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          bat 'echo %DOCKER_PASS%| docker login -u %DOCKER_USER% --password-stdin'
          bat 'docker push %DOCKER_IMAGE%:latest'
          bat 'docker push %DOCKER_IMAGE%:%BUILD_NUMBER%'
        }
      }
    }

    stage('Update Kubernetes Deployment') {
      steps {
        bat 'powershell -Command "(Get-Content k8s\\deployment.yaml) -replace \'image:.*\', \'image: %DOCKER_IMAGE%:%BUILD_NUMBER%\' | Set-Content k8s\\deployment.yaml"'
        bat 'kubectl apply -f k8s/deployment.yaml --validate=false'
        bat 'kubectl rollout status deployment/cicd-demo'
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