pipeline {
  agent any
  tools {
    nodejs 'MyNodeJS'
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
    stage('Setup Environment') {
      steps {
        script {
          sh '''
            # Update package list
            apt-get update
            
            # Install Docker
            if ! command -v docker &> /dev/null; then
              echo "Installing Docker..."
              apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
              curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
              echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
              apt-get update
              apt-get install -y docker-ce-cli
            fi
            
            # Install AWS CLI
            if ! command -v aws &> /dev/null; then
              echo "Installing AWS CLI..."
              curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
              unzip -q awscliv2.zip
              ./aws/install
              rm -rf awscliv2.zip aws/
            fi
            
            # Install kubectl
            if ! command -v kubectl &> /dev/null; then
              echo "Installing kubectl..."
              curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
              chmod +x kubectl
              mv kubectl /usr/local/bin/
            fi
            
            # Verify installations
            docker --version || echo "Docker not available"
            aws --version
            kubectl version --client
            node --version
            npm --version
          '''
        }
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
              
              # Clean up local images
              docker rmi $ECR_REPO_FRONTEND:latest || true
              docker rmi $ECR_REGISTRY/$ECR_REPO_FRONTEND:latest || true
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
              
              # Clean up local images
              docker rmi $ECR_REPO_BACKEND:latest || true
              docker rmi $ECR_REGISTRY/$ECR_REPO_BACKEND:latest || true
            '''
          }
        }
      }
    }
    
    stage('Deploy to EKS') {
      steps {
        withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
          sh '''
            # Apply Kubernetes manifests
            kubectl apply -f k8s_manifests/ --recursive --namespace=app
            
            # Wait for deployment to be ready
            kubectl wait --for=condition=available --timeout=300s deployment --all -n app || true
            
            # Show deployment status
            kubectl get deployments -n app
            kubectl get pods -n app
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
