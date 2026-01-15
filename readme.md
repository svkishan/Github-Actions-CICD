STEP 1: Create AWS EC2 Ubuntu Instance
1. Launch EC2

AMI: Ubuntu 22.04

Instance type: t2.medium (important for Minikube)

Storage: 20 GB

Security Group:

SSH → Port 22

HTTP → Port 80

App → Port 3000

2. Connect to EC2
ssh ubuntu@<EC2_PUBLIC_IP>

 STEP 2: Install Basic Tools on EC2
Update system
sudo apt update -y

Install Docker
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker


Verify:

docker --version

Install kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/


Verify:

kubectl version --client

Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube


Start Minikube:

minikube start --driver=docker


Check:

kubectl get nodes


 Kubernetes is ready

 STEP 3: Create a Simple App (Node.js)
Create project
mkdir cicd-app && cd cicd-app
npm init -y
npm install express

Create index.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello from CI/CD Pipeline 🚀');
});

app.listen(3000, () => {
  console.log('App running on port 3000');
});

Update package.json
"scripts": {
  "start": "node index.js",
  "test": "echo \"No tests yet\""
}

 STEP 4: Dockerize the App
Create Dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]


Test locally:

docker build -t cicd-app .
docker run -p 3000:3000 cicd-app


Open browser:

http://EC2_PUBLIC_IP:3000

STEP 5: Kubernetes Manifests
Create folder
mkdir k8s

deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cicd-app
  template:
    metadata:
      labels:
        app: cicd-app
    spec:
      containers:
      - name: cicd-app
        image: DOCKER_USERNAME/cicd-app:latest
        ports:
        - containerPort: 3000

service.yaml
apiVersion: v1
kind: Service
metadata:
  name: cicd-app-service
spec:
  type: NodePort
  selector:
    app: cicd-app
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30007

 STEP 6: Push Code to GitHub
git init
git add .
git commit -m "Initial CI/CD app"
git branch -M main
git remote add origin https://github.com/<USERNAME>/cicd-app.git
git push -u origin main

 STEP 7: Docker Hub Secrets (IMPORTANT)

Go to GitHub Repo → Settings → Secrets → Actions

Add:

DOCKER_USERNAME

DOCKER_PASSWORD

⚙ STEP 8: GitHub Actions Workflow
Create folder
mkdir -p .github/workflows

Create ci-cd.yml
name: CI-CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: self-hosted

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Build Docker Image
      run: docker build -t ${{ secrets.DOCKER_USERNAME }}/cicd-app:latest .

    - name: Login to Docker Hub
      run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

    - name: Push Image
      run: docker push ${{ secrets.DOCKER_USERNAME }}/cicd-app:latest

    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml

 STEP 9: Setup Self-Hosted GitHub Runner (KEY STEP)

On EC2:

GitHub → Repo → Settings → Actions → Runners

Click New self-hosted runner

Copy commands and run them on EC2

Example:

./run.sh


 Runner connected to GitHub

🚀 STEP 10: Trigger the Pipeline

Make a small change:

git commit -am "Trigger pipeline"
git push


Go to:

GitHub → Actions → Watch pipeline run 

 STEP 11: Access App from Kubernetes
kubectl get svc


Open:

http://EC2_PUBLIC_IP:30007


 Your CI/CD pipeline is LIVE
