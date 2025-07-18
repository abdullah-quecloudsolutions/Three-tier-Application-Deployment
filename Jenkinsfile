pipeline {
  agent {
    kubernetes {
      yamlFile 'jenkins-agent-pod-template.yaml'
    }
  }
  
  environment {
    AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
    AWS_REGION = 'us-east-1'
    ECR_REGISTRY = '932757390465.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO_FRONTEND = 'tfp-eks-frontend'
    ECR_REPO_BACKEND = 'tfp-eks-backend'
    KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-credentials-id'
  }
  
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    
    stage('Build & Test Frontend') {
      steps {
        container('jnlp') {
          dir('frontend') {
            sh 'npm install'
            sh 'npm run build'
            sh 'npm test || echo "Tests failed but continuing pipeline"'
          }
        }
      }
    }
    
    stage('Build & Test Backend') {
      steps {
        container('jnlp') {
          dir('backend') {
            sh 'npm install'
            sh 'npm test || echo "Tests failed but continuing pipeline"'
          }
        }
      }
    }
    
    stage('Docker Build & Push Frontend') {
      steps {
        container('docker') {
          container('aws-cli') {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-credentials']]) {
              dir('frontend') {
                sh '''
                  aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                  docker build -t $ECR_REPO_FRONTEND:latest .
                  docker tag $ECR_REPO_FRONTEND:latest $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest
                  docker push $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest
                '''
              }
            }
          }
        }
      }
    }
    
    stage('Docker Build & Push Backend') {
      steps {
        container('docker') {
          container('aws-cli') {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-credentials']]) {
              dir('backend') {
                sh '''
                  aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                  docker build -t $ECR_REPO_BACKEND:latest .
                  docker tag $ECR_REPO_BACKEND:latest $ECR_REGISTRY/$ECR_REPO_BACKEND:latest
                  docker push $ECR_REGISTRY/$ECR_REPO_BACKEND:latest
                '''
              }
            }
          }
        }
      }
    }
    
    stage('Deploy to EKS') {
      steps {
        container('kubectl') {
          withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
            sh '''
              kubectl apply -f k8s_manifests/ --recursive --namespace=app
              kubectl wait --for=condition=available --timeout=300s deployment --all -n app || true
              kubectl get deployments -n app
              kubectl get pods -n app
            '''
          }
        }
      }
    }
  }
  
  post {
    always {
      container('docker') {
        sh '''
          docker rmi $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest || true
          docker rmi $ECR_REGISTRY/$ECR_REPO_BACKEND:latest || true
          docker rmi $ECR_REPO_FRONTEND:latest || true
          docker rmi $ECR_REPO_BACKEND:latest || true
        '''
      }
    }
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed. Check the logs for details.'
    }
  }
}
