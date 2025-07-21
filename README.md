# 🚀 Eyego DevOps Project: CI/CD with Jenkins, Docker, AWS ECR & EKS

## 📌 Project Description

This project demonstrates a complete CI/CD workflow using:

- Jenkins for Continuous Integration
- Docker for containerization
- AWS ECR for image storage
- AWS EKS (Elastic Kubernetes Service) for deployment
- ArgoCD for GitOps-based Continuous Deployment (optional)
- Kubernetes manifests stored under `k8s/`

---

## 🛠️ Technologies Used

- **Jenkins**
- **Docker**
- **AWS ECR**
- **AWS EKS**
- **Kubernetes**
- **ArgoCD** (Optional)
- **Shell scripting**

---

## 🧱 Project Structure

```
Eyego-Assesment/
├── Jenkinsfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── README.md
```

---

## ✅ Prerequisites

- Jenkins running with Docker and AWS CLI installed on the agent
- IAM user with ECR and EKS permissions
- AWS Credentials stored in Jenkins Credentials Manager:
  - `aws-access-key`
  - `aws-secret-key`
  - `aws-session-token`
  - or a single AWS credential ID like `aws`
- A running EKS cluster (e.g., `my-eks-cluster`)
- A namespace `eyego` in Kubernetes (created automatically if not exists)
- Jenkins plugins: Pipeline, Docker, AWS Credentials, Git, GitHub

---

## 🚦 CI/CD Workflow Overview

### 1. Clone the Project

```bash
git clone https://github.com/Orabi20/Eyego-Assesment.git
cd Eyego-Assesment
```

### 2. Jenkinsfile Stages

#### 🔐 Stage: Login to AWS ECR
- Configure AWS credentials dynamically
- Authenticate to AWS ECR
- Required for pushing Docker images

#### 🛠️ Stage: Build, Tag & Push Docker Image
- Build Docker image from Dockerfile
- Tag it with `BUILD_NUMBER`
- Push to AWS ECR

#### 🚀 Stage: Update Kubernetes Deployment
- Update `deployment.yaml` image to latest pushed tag
- Create namespace `eyego` if missing
- Apply Kubernetes manifests (`deployment.yaml` and `service.yaml`)
- Print ELB DNS to access the application

---

## 📄 Jenkinsfile Summary

```groovy
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

    stage('Build, Tag & Push Docker Image') {
      steps {
        sh '''
          docker build -t $ECR_REPO:$IMAGE_TAG .
          docker tag $ECR_REPO:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
        '''
      }
    }

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

              kubectl get namespace eyego || kubectl create namespace eyego

              sed -i "s|image:.*|image: $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG|" $DEPLOYMENT_FILE

              kubectl apply -n eyego -f $DEPLOYMENT_FILE
              kubectl apply -n eyego -f $SERVICE_FILE

              echo "🔗 Application DNS:"
              kubectl get svc -n eyego $DEPLOYMENT_NAME -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
            '''
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
```

---

## 🌐 Access the Application

After a successful deployment, your app will be accessible via the **LoadBalancer DNS**, which is printed in the Jenkins logs:

```
🔗 Application DNS:
<your-elb-dns>.elb.amazonaws.com
```

---

## 🧪 Bonus Tips

- You can integrate ArgoCD for GitOps deployment by pushing updated manifests to a Git repo watched by ArgoCD.
- Consider using `helm` for managing Kubernetes manifests.
- Secure your Jenkins with role-based access and credentials masking.

---

## 📬 Contact

**Author:** Ahmed Orabi  
**GitHub:** [@Orabi20](https://github.com/Orabi20)
