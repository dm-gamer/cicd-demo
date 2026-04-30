pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "sanjay5raj/cicd-demo"
    // Windows-specific paths
    KUBECONFIG = "${USERPROFILE}\\.kube\\config"
    MINIKUBE_HOME = "${USERPROFILE}\\.minikube"
  }

  stages {

    stage('Clone Repository') {
      steps {
        git branch: 'main',
            url: 'https://github.com/dm-gamer/cicd-demo.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        bat """
          echo Building Docker image...
          docker build -t ${DOCKER_IMAGE}:latest .
          docker tag ${DOCKER_IMAGE}:latest ${DOCKER_IMAGE}:${BUILD_NUMBER}
          docker images | findstr ${DOCKER_IMAGE}
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
            echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
            docker push ${DOCKER_IMAGE}:latest
            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
            echo Docker push completed successfully
          """
        }
      }
    }

    stage('Setup Minikube Context') {
      steps {
        bat """
          echo === Setting up Minikube for Windows ===
          
          :: Start Minikube if not running
          minikube status || minikube start --driver=docker
          
          :: Configure kubectl to use minikube
          kubectl config use-context minikube
          
          :: Verify cluster access
          kubectl cluster-info
          kubectl get nodes
          
          :: Enable necessary addons
          minikube addons enable ingress
          minikube addons enable dashboard
          
          :: Show current context
          kubectl config current-context
        """
      }
    }

    stage('Prepare Kubernetes Manifests') {
      steps {
        bat """
          echo === Checking Kubernetes manifests ===
          if not exist "k8s\\" (
            echo Creating k8s directory...
            mkdir k8s
          )
          
          :: List manifest files
          if exist "k8s\\deployment.yaml" (
            echo Found deployment.yaml
            type k8s\\deployment.yaml
          ) else (
            echo ERROR: deployment.yaml not found in k8s directory!
            echo Creating sample deployment.yaml...
            (
              echo apiVersion: apps/v1
              echo kind: Deployment
              echo metadata:
              echo   name: cicd-demo
              echo   labels:
              echo     app: cicd-demo
              echo spec:
              echo   replicas: 2
              echo   selector:
              echo     matchLabels:
              echo       app: cicd-demo
              echo   template:
              echo     metadata:
              echo       labels:
              echo         app: cicd-demo
              echo     spec:
              echo       containers:
              echo       - name: cicd-demo
              echo         image: ${DOCKER_IMAGE}:${BUILD_NUMBER}
              echo         ports:
              echo         - containerPort: 80
              echo         resources:
              echo           limits:
              echo             memory: "256Mi"
              echo             cpu: "250m"
              echo ---
              echo apiVersion: v1
              echo kind: Service
              echo metadata:
              echo   name: cicd-demo-service
              echo spec:
              echo   selector:
              echo     app: cicd-demo
              echo   ports:
              echo   - protocol: TCP
              echo     port: 80
              echo     targetPort: 80
              echo     nodePort: 30080
              echo   type: NodePort
            ) > k8s\\deployment.yaml
          )
        """
      }
    }

    stage('Update Kubernetes Deployment') {
      steps {
        bat """
          echo === Updating deployment image ===
          
          :: Update the image tag in deployment.yaml (PowerShell method)
          powershell -Command "
            \$file = 'k8s\\deployment.yaml';
            if (Test-Path \$file) {
              \$content = Get-Content \$file -Raw;
              \$content = \$content -replace 'image: [^\\s]+', 'image: ${DOCKER_IMAGE}:${BUILD_NUMBER}';
              Set-Content \$file -Value \$content -NoNewline;
              Write-Host 'Updated deployment.yaml with image: ${DOCKER_IMAGE}:${BUILD_NUMBER}';
            } else {
              Write-Error 'deployment.yaml not found!';
              exit 1;
            }
          "
          
          :: Show updated manifest
          echo === Updated Deployment Manifest ===
          type k8s\\deployment.yaml
          
          :: Apply the deployment
          echo === Applying Kubernetes manifests ===
          kubectl apply -f k8s\\deployment.yaml
          
          :: Wait for rollout to complete
          echo === Waiting for rollout to complete ===
          kubectl rollout status deployment/cicd-demo --timeout=5m
          
          :: If rollout fails, get detailed info
          if %errorlevel% neq 0 (
            echo Rollout failed! Getting details...
            kubectl describe deployment cicd-demo
            kubectl get pods
            kubectl logs -l app=cicd-demo --tail=50
            exit /b 1
          )
        """
      }
    }

    stage('Verify Deployment') {
      steps {
        bat """
          echo === Verifying Kubernetes Deployment ===
          
          :: Get deployment status
          kubectl get deployment cicd-demo
          kubectl get replicaset -l app=cicd-demo
          
          :: Get pods status
          echo === Pods ===
          kubectl get pods -o wide
          
          :: Get services
          echo === Services ===
          kubectl get svc
          
          :: Get endpoints
          echo === Endpoints ===
          kubectl get endpoints
          
          :: Check pod logs
          echo === Recent Pod Logs ===
          kubectl logs -l app=cicd-demo --tail=20 --all-containers=true || echo No logs available yet
          
          :: Get service URL (for Minikube)
          echo === Service Access URL ===
          minikube service cicd-demo-service --url || echo Service URL not available
          
          :: Test if pods are running
          set POD_COUNT=0
          for /f %%i in ('kubectl get pods -l app=cicd-demo --no-headers ^| find /c /v ""') do set POD_COUNT=%%i
          echo Running pods count: %POD_COUNT%
          
          if %POD_COUNT% equ 0 (
            echo ERROR: No pods are running!
            kubectl describe pods
            exit /b 1
          )
        """
      }
    }

    stage('Test Application') {
      steps {
        bat """
          echo === Testing Application ===
          
          :: Port forward for testing (run in background)
          start /B kubectl port-forward service/cicd-demo-service 8080:80
          
          :: Wait for port-forward to establish
          timeout /t 5 /nobreak
          
          :: Test the endpoint
          powershell -Command "
            try {
              \$response = Invoke-WebRequest -Uri http://localhost:8080 -TimeoutSec 10;
              Write-Host 'Application responded with status:' \$response.StatusCode;
              if (\$response.StatusCode -eq 200) {
                Write-Host 'Application test PASSED';
                exit 0;
              } else {
                Write-Host 'Application test FAILED with status:' \$response.StatusCode;
                exit 1;
              }
            } catch {
              Write-Host 'Application test FAILED:' \$_.Exception.Message;
              exit 1;
            }
          "
          
          :: Clean up port-forward
          taskkill /F /IM kubectl.exe
        """
      }
    }
  }

  post {
    success {
      echo '========================================='
      echo 'Pipeline succeeded! App deployed successfully to Minikube'
      echo '========================================='
      script {
        // Get Minikube service URL for access
        bat """
          echo Access your application at:
          minikube service list
          echo Or run: minikube service cicd-demo-service
        """
      }
      bat 'docker system prune -f || echo Cleanup skipped'
    }
    
    failure {
      echo '========================================='
      echo 'Pipeline failed. Check the logs above for details.'
      echo '========================================='
      script {
        // Get debug information on failure
        bat """
          echo === Debug Information ===
          echo Docker images:
          docker images | findstr ${DOCKER_IMAGE}
          
          echo === Kubernetes Events ===
          kubectl get events --sort-by='.lastTimestamp' | tail -n 20
          
          echo === Pod Details ===
          kubectl describe pods
          
          echo === Minikube Status ===
          minikube status
        """
      }
    }
    
    always {
      bat """
        echo Cleaning up...
        docker logout 2>nul || echo Already logged out
        echo Pipeline finished at %DATE% %TIME%
      """
    }
  }
}