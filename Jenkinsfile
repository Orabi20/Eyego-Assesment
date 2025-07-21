pipeline {
  agent any

  environment {
    AWS_REGION = 'us-east-1'
    ECR_REPO = 'hello-eyego'
    ECR_REGISTRY = '289222951012.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_TAG = "${BUILD_NUMBER}"
    CLUSTER_NAME = 'my-eks-cluster'
    DEPLOYMENT_NAME = 'hello-eyego'
    DEPLOYMENT_FILE = 'deployment.yaml'
    SERVICE_FILE = 'service.yaml'
  }

  stages {
    // stage('Checkout') {
    //   steps {
    //     git 'https://github.com/YOUR_USERNAME/hello-eyego.git'
    //   }
    // }

    // stage('Build Docker Image') {
    //   steps {
    //     script {
    //       sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
    //     }
    //   }
    // }

    stage('Login to AWS ECR') {
      steps {
        script {
          sh """
            aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ECR_REGISTRY}
          """
        }
      }
    }

    // stage('Tag & Push Image to ECR') {
    //   steps {
    //     script {
    //       sh """
    //         docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
    //         docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
    //       """
    //     }
    //   }
    // }

    stage('Update K8s Deployment') {
      steps {
        dir('k8s') {
          script {
            sh """
              aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
              kubectl apply -f ${DEPLOYMENT_FILE}
              kubectl apply -f ${SERVICE_FILE}
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployment successful!"
    }
    failure {
      echo "❌ Deployment failed."
    }
  }
}
