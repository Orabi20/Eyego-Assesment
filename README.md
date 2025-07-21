# Eyego Assesment

## Assesment Description

This Assesment demonstrates a complete CI/CD workflow using:

- Jenkins for Continuous Integration
- Docker for containerization
- AWS ECR for image storage
- AWS EKS (Elastic Kubernetes Service) for deployment
- Kubernetes manifests stored under `k8s/`

---

## 🛠️ Technologies Used

- **Jenkins**
- **Docker**
- **AWS ECR**
- **AWS EKS**
- **Kubernetes**

---

## Project Structure

```
Eyego-Assesment/
├── app.js
├── package.json
├── Dockerfile
├── Jenkinsfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── README.md
```

---

## Prerequisites

- Jenkins running with Docker and AWS CLI installed on the agent
- IAM user with ECR and EKS permissions
- AWS Credentials stored in Jenkins Credentials Manager:
  - `aws-access-key`
  - `aws-secret-key`
  - `aws-session-token`
- A running EKS cluster

---

## 🚦 CI/CD Workflow Overview

### 1. Clone the Project

```bash
git clone https://github.com/Orabi20/Eyego-Assesment.git
cd Eyego-Assesment
```

### 2. Jenkinsfile Stages

#### Stage: Login to AWS ECR
- Configure AWS credentials dynamically
- Authenticate to AWS ECR
- Required for pushing Docker images

#### Stage: Build, Tag & Push Docker Image
- Build Docker image from Dockerfile
- Tag it with `BUILD_NUMBER`
- Push to AWS ECR

#### Stage: Update Kubernetes Deployment
- Update `deployment.yaml` image to latest pushed tag
- Create namespace `eyego` if missing
- Apply Kubernetes manifests (`deployment.yaml` and `service.yaml`)
- Print ELB DNS to access the application

---

## Jenkinsfile Summary

```groovy
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

            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          docker build -t $ECR_REPO:$IMAGE_TAG .
        '''
      }
    }

    stage('Tag & Push Docker Image to ECR') {
      steps {
        sh '''
          docker tag $ECR_REPO:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Update K8s Deployment') {
      steps {
         dir('k8s') {
            sh '''
              aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
              
              kubectl get namespace eyego || kubectl create namespace $NAMESPACE

              sed -i "s|image:.*|image: $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG|" $DEPLOYMENT_FILE
              
              kubectl apply -n $NAMESPACE -f $DEPLOYMENT_FILE
              kubectl apply -n $NAMESPACE -f $SERVICE_FILE

              kubectl get svc -n $NAMESPACE $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
            '''
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
```

---

## Access the Application

After a successful deployment, the app will be accessible via the **LoadBalancer DNS**, which is printed in the Jenkins logs:

```
a49e9e6f51e684e9580e11fb335efc31-815840059.us-east-1.elb.amazonaws.com
```

---
