# EasyCRUD DevOps Deployment Documentation

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [EC2 Setup](#ec2-setup)
- [Docker Setup](#docker-setup)
- [Kubernetes (EKS)](#kubernetes-eks)
- [Ingress Setup](#ingress-setup)
- [Jenkins Setup](#jenkins-setup)
- [CI/CD Flow](#cicd-flow)
- [How to Use This Project](#how-to-use-this-project)
- [Next Improvements](#next-improvements)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Key Lessons](#key-lessons)

---

## Overview

This document provides a complete step-by-step guide to deploy the EasyCRUD application using:

- AWS EC2
- Docker
- Jenkins (CI/CD)
- Kubernetes (EKS)
- Ingress (NGINX)

---

## Architecture

| Component | Purpose |
|---|---|
| EC2 (Jenkins Server) | CI/CD pipeline execution |
| EC2 (App Build Server) | Build Docker images |
| AWS EKS Cluster | Run application (frontend + backend) |
| DockerHub | Store container images |
| Ingress (NGINX) | External traffic routing |

---

## EC2 Setup

### Instances Used

| Instance | Purpose | Recommended Type |
|---|---|---|
| Jenkins Server | CI/CD | c7i-flex-large |
| Build Server | Docker builds + kubectl | c7i-flex-large |

### Basic Setup (run on both EC2 instances)

```bash
sudo apt update -y
sudo apt install docker.io git -y
sudo usermod -aG docker ubuntu
```

> **Note:** Logout and login again after running the above.

---

## Docker Setup

### Backend Dockerfile

```dockerfile
FROM openjdk:17-jdk-slim
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

### Frontend Dockerfile

```dockerfile
FROM node:18 AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

### Build & Push Images

```bash
docker build -t <dockerhub-username>/backend:tag .
docker push <dockerhub-username>/backend:tag
```

---

## Kubernetes (EKS)

### Create Cluster

Use AWS Console or Terraform.

### Connect to Cluster

```bash
aws eks update-kubeconfig --region us-east-1 --name cluster
kubectl get nodes
```

### Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: <dockerhub>/backend:tag
        ports:
        - containerPort: 8081
```

### Frontend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <dockerhub>/frontend:tag
        ports:
        - containerPort: 80
```

### Services

```yaml
kind: Service
apiVersion: v1
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 8081
```

---

## Ingress Setup

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ultron-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8081
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## Jenkins Setup

### Install Jenkins

```bash
sudo apt install openjdk-17-jdk -y
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo apt install jenkins -y
```

### Jenkinsfile (CI/CD Pipeline)

```groovy
pipeline {
    agent any

    environment {
        DOCKER_USER = "your-dockerhub"
        BACKEND_IMAGE = "${DOCKER_USER}/backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "${DOCKER_USER}/frontend:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/mrstark316-lgtm/EasyCurd.git'
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                cd backend
                mvn clean package -DskipTests
                docker build --no-cache -t $BACKEND_IMAGE .

                cd ../frontend
                docker build --no-cache -t $FRONTEND_IMAGE .
                '''
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $BACKEND_IMAGE
                    docker push $FRONTEND_IMAGE
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                kubectl set image deployment/backend backend=$BACKEND_IMAGE
                kubectl set image deployment/frontend frontend=$FRONTEND_IMAGE
                '''
            }
        }
    }
}
```

---

## CI/CD Flow

```
Developer → GitHub → Jenkins → DockerHub → Kubernetes → Live App
```

---

## How to Use This Project

1. Clone the repository
2. Setup EC2 servers
3. Setup Docker & Jenkins
4. Configure EKS cluster
5. Apply Kubernetes YAMLs
6. Run the Jenkins pipeline

---

## Next Improvements

- [ ] HTTPS setup
- [ ] Domain integration
- [ ] Monitoring (Prometheus + Grafana)
- [ ] Auto-scaling

---

## Troubleshooting Guide

### 1. Maven Build Fails — Java Version Mismatch

**Error:** `release version 17 not supported`

**Cause:** JDK mismatch on EC2.

**Fix:**
```bash
sudo apt install openjdk-17-jdk -y
java -version
```

---

### 2. Frontend Docker Build Fails

**Error:** `COPY --from=build /app/build not found`

**Cause:** Wrong build output directory — Vite uses `dist`, not `build`.

**Fix:**
```dockerfile
COPY --from=build /app/dist /usr/share/nginx/html
```

---

### 3. Ingress Not Working

**Error:** App not accessible / 404

**Cause:** Missing ingress controller or wrong class.

**Fix:**
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Ensure `ingressClassName: nginx` is set in your Ingress manifest.

---

### 4. API Not Working — Whitelabel Error

**Error:** Spring Boot Whitelabel page

**Cause:** Wrong endpoint (`/students` vs `/users`).

**Fix:**
```
Correct endpoint: /api/users
```

---

### 5. Frontend Not Loading Data

**Cause:** Frontend calling wrong API or wrong `BASE_URL`.

**Fix:**
```js
const apiUrl = '/api';
```

---

### 6. Localhost Issue in Production

**Error:** API calls hitting `localhost`

**Cause:** Hardcoded fallback URL.

**Fix:**
```js
getApiUrl() {
  return '/api';
}
```

---

### 7. Jenkins Pipeline Fails — Missing Credentials

**Error:** `docker-creds not found`

**Fix:** Go to **Jenkins → Manage Credentials** and add DockerHub credentials with ID: `docker-creds`.

---

### 8. Frontend Works Manually but Breaks via Pipeline

**Cause:** Code changes not pushed to GitHub before running the pipeline.

**Fix:**
```bash
git add .
git commit -m "fix frontend config"
git push origin main
```

---

### 9. Old Frontend Still Loading

**Cause:** Docker or browser cache serving stale content.

**Fix:**
```bash
docker build --no-cache -t frontend:v2 .
kubectl set image deployment/frontend frontend=frontend:v2
```

For browser cache:
```
Hard refresh → Ctrl + Shift + R
```

---

### 10. Wrong Image Running in Kubernetes

**Fix:**
```bash
kubectl describe pod <pod-name> | grep Image
```

Ensure the correct tag is being used — never rely on `latest` in production.

---

## Key Lessons

- Never assume API endpoints — always verify
- Always push code to GitHub before running the pipeline
- Avoid `latest` tag in production deployments
- Use `/api` prefix with ingress routing
- Always verify which container image is actually running

---

## 🧑‍💻 Author
Ashif Makandar

## ⚙️ Tech Stack
AWS EKS | Jenkins | Docker | Kubernetes | NGINX | React | Spring Boot


*Project status: Fully functional and production-ready baseline.*
