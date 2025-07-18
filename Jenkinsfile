pipeline {
  agent {
    docker {
      image 'node:16-alpine'
      args '-v /var/run/docker.sock:/var/run/docker.sock'
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
    stage('Setup') {
      steps {
        sh '''
          # Install required tools
          apk add --no-cache docker-cli curl unzip
          
          # Install AWS CLI
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip -q awscliv2.zip
          ./aws/install
          rm -rf awscliv2.zip aws/
          
          # Install kubectl
          curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl
          mv kubectl /usr/local/bin/
          
          # Verify installations
          node --version
          npm --version
          docker --version
          aws --version
          kubectl version --client
        '''
      }
    }
    
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
          sh 'npm test -- --passWithNoTests || true'
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
              # Configure AWS credentials
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws configure set region $AWS_REGION
              
              # Login to ECR
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
              
              # Build and push Docker image
              docker build -t $ECR_REPO_FRONTEND:latest .
              docker tag $ECR_REPO_FRONTEND:latest $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest
              docker push $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest
            '''
          }
        }
      }
    }
    
    stage('Docker Build & Push Backend') {
      steps {
        script {
          dir('backend') {
            sh '''
              # Configure AWS credentials
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws configure set region $AWS_REGION
              
              # Login to ECR
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
              
              # Build and push Docker image
              docker build -t $ECR_REPO_BACKEND:latest .
              docker tag $ECR_REPO_BACKEND:latest $ECR_REGISTRY/$ECR_REPO_BACKEND:latest
              docker push $ECR_REGISTRY/$ECR_REPO_BACKEND:latest
            '''
          }
        }
      }
    }
    
    stage('Deploy to EKS') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
          sh '''
            kubectl apply -f k8s_manifests/ --recursive --namespace=app
            kubectl wait --for=condition=available --timeout=300s deployment --all -n app
          '''
        }
      }
    }
  }
  
  post {
    always {
      cleanWs()
    }
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed. Check the logs for details.'
    }
  }
}
