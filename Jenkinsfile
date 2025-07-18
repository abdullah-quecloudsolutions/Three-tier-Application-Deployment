pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          serviceAccountName: jenkins
          containers:
          - name: node
            image: node:16
            command: ["sleep", "infinity"]
            volumeMounts:
            - name: docker-sock
              mountPath: /var/run/docker.sock
          - name: docker
            image: docker:24
            command: ["sleep", "infinity"]
            volumeMounts:
            - name: docker-sock
              mountPath: /var/run/docker.sock
          - name: aws-cli
            image: amazon/aws-cli:latest
            command: ["sleep", "infinity"]
          - name: kubectl
            image: bitnami/kubectl:latest
            command: ["sleep", "infinity"]
          volumes:
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
              type: Socket
      '''
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
        container('node') {
          dir('frontend') {
            sh 'npm install'
            sh 'npm run build'
            sh 'npm test -- --passWithNoTests || true'
          }
        }
      }
    }
    
    stage('Build & Test Backend') {
      steps {
        container('node') {
          dir('backend') {
            sh 'npm install'
            sh 'npm test || true'
          }
        }
      }
    }
    
    stage('Docker Build & Push Frontend') {
      steps {
        container('docker') {
          dir('frontend') {
            sh '''
              # Login to ECR using aws-cli container
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
        container('docker') {
          dir('backend') {
            sh '''
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
