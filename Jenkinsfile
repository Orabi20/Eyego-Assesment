pipeline {
  agent any

  environment {
    AWS_REGION = 'us-east-1'
    ECR_REPO = 'hello-eyego'
    ECR_REGISTRY = '289222951012.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_TAG = "${BUILD_NUMBER}"
    CLUSTER_NAME = 'my-eks-cluster'
    NAMESPACE = 'eyego'
    SERVICE_NAME = 'hello-eyego-service'
    DEPLOYMENT_FILE = 'deployment.yaml'
    SERVICE_FILE = 'service.yaml'
  }

  stages {

    stage('Login to AWS ECR') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN')
        ]) {
          sh '''
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            aws configure set aws_session_token $AWS_SESSION_TOKEN
            aws configure set region $AWS_REGION

            aws sts get-caller-identity
            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          '''
        }
      }
    }

    // stage('Build Docker Image') {
    //   steps {
    //     sh '''
    //       docker build -t $ECR_REPO:$IMAGE_TAG .
    //     '''
    //   }
    // }

    // stage('Tag & Push Docker Image to ECR') {
    //   steps {
    //     sh '''
    //       docker tag $ECR_REPO:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
    //       docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
    //     '''
    //   }
    // }

    stage('Update K8s Deployment') {
      steps {
        withCredentials([
          string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN')
        ]) {
          dir('k8s') {
            sh '''
              aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
              aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
              aws configure set aws_session_token $AWS_SESSION_TOKEN
              aws configure set region $AWS_REGION

              aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
              
               kubectl get namespace eyego || kubectl create namespace $NAMESPACE
              
              kubectl apply -n $NAMESPACE -f $DEPLOYMENT_FILE
              kubectl apply -n $NAMESPACE -f $SERVICE_FILE

              kubectl get svc -n $NAMESPACE $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployment successful"
    }
    failure {
      echo "Deployment failed"
    }
  }
}
