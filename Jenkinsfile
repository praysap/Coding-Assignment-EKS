pipeline {
  agent any
  options {
    timestamps()
  }

  environment {
    AWS_REGION        = 'us-west-2'
    CLUSTER_NAME      = 'three-tier-cluster'
    NAMESPACE         = 'three-tier'
    ECR_REPO_FRONTEND = 'mern-frontend'
    ECR_REPO_BACKEND  = 'mern-backend'
    ECR_REGISTRY      = '863541429677.dkr.ecr.us-west-2.amazonaws.com' // Replace with your AWS Account ID
  }

  stages {

    stage('Init AWS + Vars') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            echo "✅ AWS credentials and region set"
          }
        }
      }
    }

    stage('Checkout Code') {
      steps {
        git branch: 'main', url: 'https://github.com/praysap/Coding-Assignment-EKS'
      }
    }

    stage('Configure kubeconfig') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              set -e
              aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}
            """
          }
        }
      }
    }

    stage('Ensure ECR Repositories') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              set -e
              aws ecr describe-repositories --repository-names ${ECR_REPO_FRONTEND} >/dev/null 2>&1 || \
                aws ecr create-repository --repository-name ${ECR_REPO_FRONTEND}

              aws ecr describe-repositories --repository-names ${ECR_REPO_BACKEND} >/dev/null 2>&1 || \
                aws ecr create-repository --repository-name ${ECR_REPO_BACKEND}
            """
          }
        }
      }
    }

    stage('Login to ECR') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${ECR_REGISTRY}
            """
          }
        }
      }
    }

    stage('Build Frontend Image') {
      steps {
        sh """
          cd frontend
          docker build -t ${ECR_REGISTRY}/${ECR_REPO_FRONTEND}:latest .
        """
      }
    }

    stage('Build Backend Image') {
      steps {
        sh """
          cd backend
          docker build -t ${ECR_REGISTRY}/${ECR_REPO_BACKEND}:latest .
        """
      }
    }

    stage('Push Images to ECR') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              docker push ${ECR_REGISTRY}/${ECR_REPO_FRONTEND}:latest
              docker push ${ECR_REGISTRY}/${ECR_REPO_BACKEND}:latest
            """
          }
        }
      }
    }

    stage('Apply Manifests to EKS') {
      steps {
        script {
          withAWS(region: "${AWS_REGION}", credentials: 'as-eks-creds') {
            sh """
              echo "🚀 Applying backend deployment and service..."
              kubectl apply -f k8s/backend.yml -n ${NAMESPACE}

              echo "⏳ Waiting for backend pods to be ready..."
              kubectl rollout status deployment/backend-deployment -n ${NAMESPACE} --timeout=180s

              echo "🚀 Applying frontend deployment and service..."
              kubectl apply -f k8s/frontend.yml -n ${NAMESPACE}

              echo "⏳ Waiting for frontend pods to be ready..."
              kubectl rollout status deployment/frontend-deployment -n ${NAMESPACE} --timeout=180s
            """
          }
        }
      }
    }
  }
}

