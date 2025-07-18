pipeline {
  agent any
  tools {
    nodejs 'MyNodeJS'
    dockerTool 'MyDocker'
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
    stage('Docker Build & Push Frontend') {
      steps {
        script {
          dir('frontend') {
            sh '''
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
            '''
            sh "docker build -t $ECR_REPO_FRONTEND:latest ."
            sh "docker tag $ECR_REPO_FRONTEND:latest $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest"
            sh "docker push $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest"
          }
        }
      }
    }
    stage('Docker Build & Push Backend') {
      steps {
        script {
          dir('backend') {
            sh '''
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
            '''
            sh "docker build -t $ECR_REPO_BACKEND:latest ."
            sh "docker tag $ECR_REPO_BACKEND:latest $ECR_REGISTRY/$ECR_REPO_BACKEND:latest"
            sh "docker push $ECR_REGISTRY/$ECR_REPO_BACKEND:latest"
          }
        }
      }
    }
    stage('Deploy to EKS') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
          sh 'kubectl apply -f k8s_manifests/ --recursive --namespace=app'
        }
      }
    }
  }
}
