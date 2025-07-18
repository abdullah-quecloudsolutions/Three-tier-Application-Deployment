pipeline {
  agent any
  tools {
    nodejs 'MyNodeJS'
  }
  environment {
    AWS_REGION = 'us-east-1'
    ECR_REGISTRY = '932757390465.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO_FRONTEND = 'tfp-eks-frontend'
    ECR_REPO_BACKEND = 'tfp-eks-backend'
    KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-credentials-id'
  }
  stages {
    stage('Build & Test Frontend') {
      steps {
        dir('frontend') {
          sh 'npm install'
          sh 'npm run build'
          sh 'npm test || true'
        }
      }
    }
    stage('Build & Test Backend') {
      steps {
        dir('backend') {
          sh 'npm install'
          sh 'npm test || true'
        }
      }
    }
    stage('Docker Build & Push') {
      when {
        branch 'main'
      }
      steps {
        script {
          sh 'aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
          dir('frontend') {
            sh "docker build -t $ECR_REPO_FRONTEND:latest ."
            sh "docker tag $ECR_REPO_FRONTEND:latest $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest"
            sh "docker push $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest"
          }
          dir('backend') {
            sh "docker build -t $ECR_REPO_BACKEND:latest ."
            sh "docker tag $ECR_REPO_BACKEND:latest $ECR_REGISTRY/$ECR_REPO_BACKEND:latest"
            sh "docker push $ECR_REGISTRY/$ECR_REPO_BACKEND:latest"
          }
        }
      }
    }
    stage('Deploy to EKS') {
      when {
        branch 'main'
      }
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
          sh 'kubectl apply -f k8s_manifests/ --namespace=app'
        }
      }
    }
  }
}
