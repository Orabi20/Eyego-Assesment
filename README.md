# Eyego Assesment

## Assesment Description

This Assesment demonstrates a complete CI/CD workflow using:

- Jenkins for Continuous Integration
- Docker for containerization
- AWS ECR for image storage
- AWS EKS (Elastic Kubernetes Service) for deployment
- Kubernetes manifests stored under `k8s/`

---

## Technologies Used

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

## CI/CD Workflow Overview

### 1. simple web app (Node.js)

```app.js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => res.send('Hello Eyego'));

app.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

```package.json
{
  "name": "hello-eyego",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

### 2. Jenkinsfile Stages

#### Stage: Login to AWS ECR
- Configure AWS credentials dynamically
- Authenticate to AWS ECR
- Required for pushing Docker images

  <img width="448" height="152" alt="image" src="https://github.com/user-attachments/assets/f8afba8f-551a-43ec-9183-77de0634a573" />


#### Stage: Build, Tag & Push Docker Image
- Build Docker image from Dockerfile
- Tag it with `BUILD_NUMBER`
- Push to AWS ECR

<img width="959" height="310" alt="image" src="https://github.com/user-attachments/assets/49071b98-195c-4fee-974a-91cfc5484e9f" />


#### Stage: Update Kubernetes Deployment
- Update `deployment.yaml` image to latest pushed tag
- Create namespace `eyego` if missing
- Apply Kubernetes manifests (`deployment.yaml` and `service.yaml`)
- Print ELB DNS to access the application

  <img width="497" height="152" alt="image" src="https://github.com/user-attachments/assets/0710bce7-87d6-4d9e-a58f-717191cb9764" />

  <img width="956" height="86" alt="image" src="https://github.com/user-attachments/assets/72a0cf8b-9006-44a3-990f-f0b2b0b5078d" />


  <img width="954" height="41" alt="image" src="https://github.com/user-attachments/assets/91fe30b8-50f1-4343-8b3f-93fe6e3f6536" />



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
<img width="959" height="317" alt="image" src="https://github.com/user-attachments/assets/09cd8531-e19f-41bf-94bf-9ed3cc9d2a7a" />

---

## Access the Application

After a successful deployment, the app will be accessible via the **LoadBalancer DNS**, which is printed in the Jenkins logs:

```
a49e9e6f51e684e9580e11fb335efc31-815840059.us-east-1.elb.amazonaws.com
```
<img width="608" height="29" alt="image" src="https://github.com/user-attachments/assets/9a9a96b6-c87f-4b48-9d67-da422ba50096" />


<img width="958" height="254" alt="image" src="https://github.com/user-attachments/assets/943e57bf-95d1-42f5-9bf9-f601c0172b25" />




---
