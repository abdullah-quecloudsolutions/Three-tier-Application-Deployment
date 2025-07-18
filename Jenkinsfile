pipeline {
  agent any
  tools {
    nodejs 'MyNodeJS'
    dockerTool 'MyDocker'
  }
  environment {
    AWS_REGION = 'us-east-1'
    ECR_REGISTRY = '932757390465.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO = 'tfp-eks'
    KUBECONFIG_CREDENTIALS_ID = 'kubeconfig-credentials-id'
  }
  stages {
    stage('Build Docker Image') {
      steps {
        script {
          sh "docker build -t $ECR_REPO:latest ."
        }
      }
    }
    stage('Push to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-jenkins-credentials']]) {
          script {
            sh "aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY"
            sh "docker tag $ECR_REPO:latest $ECR_REGISTRY/$ECR_REPO:latest"
            sh "docker push $ECR_REGISTRY/$ECR_REPO:latest"
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
