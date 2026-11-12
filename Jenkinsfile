
pipeline {
  agent any

  environment {
    AWS_REGION     = 'us-east-1'
    AWS_ACCOUNT_ID = '699038657587'
    ECR_REPOSITORY = 'digital-marketing'
    IMAGE_TAG      = "${BUILD_NUMBER}"
    ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    IMAGE_NAME     = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
  }

  stages {
    stage('Git Checkout') {
      steps {
        git branch: 'master',
            url: 'https://github.com/anilkumarvanum9-debug/carwebsite.git'
      }
    }

    
stage('SonarQube Analysis') {
    steps {
        script {
            def scannerHome = tool(
                name: 'SonarScanner',
                type: 'hudson.plugins.sonar.SonarRunnerInstallation'
            )

            withSonarQubeEnv('SonarQube') {
                sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=digital-marketing \
                    -Dsonar.sources=. \
                    -Dsonar.exclusions=node_modules/**
                """
            }
        }
    }
}


    stage('OWASP Dependency Check') {
      steps {
        sh '''
          dependency-check \
            --project "digital-marketing" \
            --scan . \
            --format HTML \
            --out dependency-report
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
      }
    }

    stage('Trivy Security Scan') {
      steps {
        sh '''
          trivy image \
            --severity HIGH,CRITICAL \
            --exit-code 1 \
            ${IMAGE_NAME}:${IMAGE_TAG}
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region ${AWS_REGION} |
            docker login --username AWS --password-stdin ${ECR_REGISTRY}
          docker push ${IMAGE_NAME}:${IMAGE_TAG}
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          aws eks update-kubeconfig \
            --region ${AWS_REGION} \
            --name devsecops-eks

          sed "s|IMAGE_PLACEHOLDER|${IMAGE_NAME}:${IMAGE_TAG}|g" \
            deployment.yaml | kubectl apply -f -

          kubectl apply -f service.yaml
          kubectl rollout status deployment/digital-marketing --timeout=300s
        '''
      }
    }

    stage('Verify Deployment') {
      steps {
        sh '''
          kubectl get deployments
          kubectl get pods
          kubectl get services
        '''
      }
    }
  }

  post {
    always {
      echo 'DevSecOps pipeline execution completed.'
    }
    success {
      echo 'Application successfully deployed to EKS.'
    }
    failure {
      echo 'Pipeline failed. Check Jenkins console output.'
    }
  }
}

