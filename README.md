# 🦖 Project Humangasaurus: EKS Migration

**Project Humangasaurus** documents the migration of the "Stranger Things" application from local Docker containers to a distributed **Microservices Architecture** on **AWS Elastic Kubernetes Service (EKS)**.

---

## 🏗️ Architecture



* **Frontend:** Nginx container serving HTML/JS (Deployed on EKS).
* **Backend:** Python Flask API (Deployed on EKS).
* **Database:** AWS RDS MySQL (External to Cluster).
* **Orchestration:** Kubernetes (EKS) managing availability and scaling.

---

## 🛠️ Prerequisites

* **AWS Account:** Access to the AWS Console (Free Tier recommended).
* **Docker Hub:** Account required to push your custom container images.
* **EC2 "Command Center":** A dedicated management instance to avoid local configuration conflicts.

---

## 🚀 Phase 1: The "Command Center" Setup

We use a "Manager" EC2 instance to control the cluster.

1.  **Launch EC2 Instance:**
    * **OS:** Ubuntu 22.04 LTS
    * **Type:** `t2.micro`
    * **IAM Role:** Attach a role with `AdministratorAccess`.
2.  **Install Tools:** SSH into the instance and run:

```bash
sudo apt update && sudo apt install unzip curl -y

# 1. Install AWS CLI
curl "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 2. Install Docker
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
sudo chmod 666 /var/run/docker.sock

# 3. Install Kubectl
curl -O [https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl](https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl)
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin

# 4. Install Eksctl
curl --silent --location "[https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname](https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname) -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 5. Install MySQL Client
sudo apt install mysql-client -y
🛢️ Phase 2: Database Infrastructure (RDS)
Create Database:

Service: AWS RDS (MySQL Free Tier).

Identifier: humangasaurus-db

Credentials: admin / strangerpassword (or your preferred secure password).

Initial DB Name: LEAVE BLANK (Crucial).

Networking:

Edit RDS Security Group to allow Port 3306 from the Manager EC2 Security Group.

⚠️ Critical Schema Initialization: Connect to RDS from the Manager and run the schema script. Note the column name changes.

Bash

mysql -h <YOUR-RDS-ENDPOINT> -u admin -p
SQL

CREATE DATABASE stranger_db;
USE stranger_db;

-- RENAMED 'question' to 'question_text' to fix backend "Unknown column" error
CREATE TABLE questions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question_text VARCHAR(255)
);

CREATE TABLE answers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question_id INT,
    answer_text VARCHAR(255)
);
EXIT;
☸️ Phase 3: Cluster Creation
Create the EKS cluster using eksctl. We use t3.small to prevent control plane timeouts.

Bash

eksctl create cluster \
  --name humangasaurus-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.small \
  --nodes 2 \
  --managed
🧠 Phase 4: Backend Deployment
Build & Push Image:

Bash

cd humangasaurus/backend
docker build -t <your-dockerhub-id>/humangasaurus-backend:latest .
docker push <your-dockerhub-id>/humangasaurus-backend:latest
Deploy to K8s: Ensure k8s/db-secret.yaml contains your RDS credentials.

Bash

kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/backend.yaml
Get LoadBalancer URL:

Bash

kubectl get svc backend-service
Copy the EXTERNAL-IP (e.g., a082...elb.amazonaws.com).

🎨 Phase 5: Frontend Configuration
We must link the Frontend to the new Backend URL and fix Nginx crashing issues.

Create frontend/default.conf:

Nginx

server {
    listen 80;
    server_name localhost;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
Update frontend/Dockerfile:

Dockerfile

FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY default.conf /etc/nginx/conf.d/
COPY . /usr/share/nginx/html
Update frontend/index.html: Replace the API variable with the Backend LoadBalancer URL from Phase 4.

JavaScript

const API = "http://<YOUR-BACKEND-EXTERNAL-IP>:5000/api";
Build & Deploy:

Bash

# Build & Push
docker build -t <your-dockerhub-id>/humangasaurus-frontend:latest .
docker push <your-dockerhub-id>/humangasaurus-frontend:latest

# Deploy
kubectl apply -f k8s/frontend.yaml
🌉 Phase 6: Networking Bridge (The Fix)
The Issue: The EKS Cluster (New VPC) cannot talk to the RDS Database (Default VPC) by default. The Lab Workaround:

Go to AWS Console -> RDS -> Modify.

Set Public Access to YES.

Edit RDS Security Group Inbound Rules:

Add Rule: MySQL (3306) -> Source: 0.0.0.0/0 (Anywhere).

🔄 How It Works (Architecture Flow)
User Request: You open the Frontend LoadBalancer URL in Chrome.

Static Serve: AWS ELB forwards traffic to the Frontend Pod (Nginx), which sends index.html to your browser.

Client-Side Logic: Your browser executes the JavaScript. When you click "Submit", your browser sends a request directly to the Backend LoadBalancer URL.

Logic Layer: Backend ELB forwards the request to the Python/Flask Pod.

Data Layer: The Backend Pod connects to RDS MySQL over the public internet (using the Phase 6 fix) to insert/read data.

🧹 Cleanup
To avoid AWS charges, delete the cluster when finished:

Bash

eksctl delete cluster --region=ap-south-1 --name=humangasaurus-cluster

---

### 🦖 Brief Summary of "How It Works"
To help you brush up quickly next time:

* **Decoupling:** Unlike the simple Docker setup where containers talked internally via `localhost` or Docker Network, here the components are physically separated.
* **The "Split Brain":** The **Frontend** is just a "dumb" file server. It doesn't talk to the backend inside the cluster. It tells **your browser** to talk to the backend.
* **The "Public Bridge":** Because EKS creates its own isolated network (VPC), it cannot see your RDS database in the default network. We solved this by making the RDS "Public" (like a website) so the EKS pods could reach it over the open internet.
* **Immutability:** Every time we changed code (like the Nginx config), we had to rebuild the Docker image and push it to Hub. Kubernetes doesn't know about code changes on your laptop; it only knows about Images in the Hub.
