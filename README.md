# project01EKS


Great! Let's start step by step from scratch to build a CI/CD pipeline for deploying a Flask application to AWS EKS using Jenkins, Docker, AWS ECR, and Kubernetes. 🚀

🛠️ Step 1: Set Up AWS Environment
Before we begin, ensure you have:
✅ AWS Account
✅ IAM User with Admin Access
✅ AWS CLI Installed
✅ kubectl Installed
✅ eksctl Installed

1️⃣ Configure AWS CLI
Run:

bash
Copy
Edit
aws configure
Enter your AWS Access Key, Secret Key, Region, and Output format.

2️⃣ Create an EKS Cluster
bash
Copy
Edit
eksctl create cluster --name flask-eks-cluster --region us-east-1 --nodegroup-name flask-nodes --nodes 2
⚡ This will take 10-15 minutes to create a Kubernetes cluster with 2 worker nodes.

3️⃣ Verify EKS Cluster
bash
Copy
Edit
aws eks update-kubeconfig --region us-east-1 --name flask-eks-cluster
kubectl get nodes
You should see the nodes in Ready state.

🛠️ Step 2: Install Jenkins on AWS EC2
1️⃣ Launch an EC2 Instance
Instance Type: t2.medium (for better performance)
Security Group: Allow 22 (SSH), 8080 (Jenkins), 5000 (Flask App), 3000 (Grafana)
Key Pair: Create and download

2️⃣ Connect to EC2
bash
Copy
Edit
ssh -i your-key.pem ec2-user@your-ec2-public-ip

3️⃣ Install Jenkins
bash
Copy
Edit
sudo yum update -y
sudo yum install java-11-amazon-corretto -y
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

4️⃣ Get Jenkins Admin Password
bash
Copy
Edit
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Copy the password and access Jenkins:

arduino
Copy
Edit
http://your-ec2-public-ip:8080
Complete the Jenkins setup by installing suggested plugins.

🛠️ Step 3: Create & Containerize Flask App

1️⃣ Create Project Structure
bash
Copy
Edit
mkdir flask-app && cd flask-app
bash
Copy
Edit
touch app.py requirements.txt Dockerfile
mkdir templates && touch templates/index.html

2️⃣ Write Flask App
Edit app.py:

python
Copy
Edit
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/")
def home():
    return render_template("index.html")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
Edit templates/index.html:

html
Copy
Edit
<!DOCTYPE html>
<html>
<head>
    <title>Flask App on AWS EKS</title>
</head>
<body>
    <h1>Hello, Jenkins CI/CD Pipeline!</h1>
</body>
</html>
Edit requirements.txt:

ini
Copy
Edit
Flask==2.2.2
Install dependencies:

bash
Copy
Edit
pip install -r requirements.txt

3️⃣ Create Dockerfile
Dockerfile
Copy
Edit
FROM python:3.9

WORKDIR /app

COPY . /app

RUN pip install -r requirements.txt

EXPOSE 5000

CMD ["python", "app.py"]

4️⃣ Build & Push Docker Image to AWS ECR
Create AWS ECR Repository
bash
Copy
Edit
aws ecr create-repository --repository-name flask-app
Authenticate Docker with ECR
bash
Copy
Edit
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin your-aws-account-id.dkr.ecr.us-east-1.amazonaws.com
Build & Push Docker Image
bash
Copy
Edit
docker build -t flask-app .
docker tag flask-app:latest your-aws-account-id.dkr.ecr.us-east-1.amazonaws.com/flask-app:latest
docker push your-aws-account-id.dkr.ecr.us-east-1.amazonaws.com/flask-app:latest
🛠️ Step 4: Deploy Flask App to AWS EKS

1️⃣ Create deployment.yaml
yaml
Copy
Edit
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: your-aws-account-id.dkr.ecr.us-east-1.amazonaws.com/flask-app:latest
        ports:
        - containerPort: 5000
        
2️⃣ Create service.yaml
yaml
Copy
Edit
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
    
3️⃣ Apply to EKS
bash
Copy
Edit
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
Verify deployment:

bash
Copy
Edit
kubectl get pods
kubectl get svc
Check the EXTERNAL-IP and open in browser! 🎉

🛠️ Step 5: Configure Jenkins Pipeline

1️⃣ Create Jenkinsfile
groovy
Copy
Edit
pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        EKS_CLUSTER = "flask-eks-cluster"
        ECR_REPO = "your-aws-account-id.dkr.ecr.us-east-1.amazonaws.com/flask-app"
        DOCKER_IMAGE = "${ECR_REPO}:latest"
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/your-github-username/flask-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }

        stage('Push Docker Image to AWS ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Update Kubernetes Deployment') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
                    sh "kubectl set image deployment/flask-app flask-app=${DOCKER_IMAGE} --record"
                }
            }
        }
    }
}

2️⃣ Push Jenkinsfile to GitHub
bash
Copy
Edit
git add Jenkinsfile
git commit -m "Added Jenkinsfile for AWS EKS deployment"
git push origin main

3️⃣ Create Jenkins Pipeline Job
New Pipeline Job → SCM → Git
Enter GitHub URL → Branch: main
Save & Build Now ✅
🎯 Final Check
✔️ Flask app is deployed on AWS EKS
✔️ Jenkins automates CI/CD pipeline
✔️ App updates with new commits

Let me know if you need any clarifications! 🚀🔥




To delete all services, deployments, and resources related to your Flask application in AWS EKS, follow these steps carefully.

🛠️ Step 1: Delete Kubernetes Resources
Run the following commands to delete services, deployments, and pods:

bash
Copy
Edit
kubectl delete service flask-service
kubectl delete deployment flask-app
To confirm deletion:

bash
Copy
Edit
kubectl get all
If the service still exists, you may need to force delete:

bash
Copy
Edit
kubectl delete service flask-service --force --grace-period=0


🛠️ Step 2: Delete Namespace (If Created)
If your resources were inside a specific namespace, delete it:

bash
Copy
Edit
kubectl delete namespace your-namespace



🛠️ Step 3: Delete EKS Cluster
⚠️ Warning: This will remove the entire cluster!

bash
Copy
Edit
eksctl delete cluster --name flask-eks-cluster
This will delete:
✅ EKS cluster
✅ Node groups
✅ Associated AWS resources

To confirm:

bash
Copy
Edit
aws eks list-clusters


🛠️ Step 4: Delete ECR Repository (Optional)
If you no longer need the Docker images in AWS ECR:

bash
Copy
Edit
aws ecr delete-repository --repository-name flask-app --force
To list repositories before deleting:

bash
Copy
Edit
aws ecr describe-repositories


🛠️ Step 5: Verify Everything is Deleted
Run:

bash
Copy
Edit
kubectl get all
eksctl get cluster
aws ecr describe-repositories
Everything should be empty or not found.

Let me know if you face any issues! 🚀









