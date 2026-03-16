This architecture will include:

* VPC
* Bastion
* EKS cluster
* Node autoscaling
* ECR
* Backend deployment
* Ingress + ALB
* Frontend hosting
* Domain setup
* Database
* ArgoCD GitOps

Core platform:

* Amazon Elastic Kubernetes Service

---

# 1. Install Required Tools

Install on **local machine or bastion host**.

Tools required:

* AWS CLI
* kubectl
* eksctl
* Docker
* Helm

Example installation (Linux):

```bash
sudo apt update
sudo apt install awscli docker.io -y
```

Install kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.29.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Install eksctl

```bash
curl --silent --location \
https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz \
| tar xz

sudo mv eksctl /usr/local/bin
```

Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

# 2. Configure AWS Credentials

Configure CLI access.

```bash
aws configure
```

Enter:

* Access key
* Secret key
* Default region

Example region:

* AWS Asia Pacific (Sydney) Region

---

# 3. Create VPC Architecture

Create one VPC for the platform, VPC and more.
--> Don't enable the auto assign public ip for pvt subnets

VPC CIDR

```
10.0.0.0/16
```

Subnets

| Type              | CIDR         |
| ----------------- | ------------ |
| Public subnet AZ1 | 10.0.1.0/24  |
| Public subnet AZ2 | 10.0.2.0/24  |
| Private app AZ1   | 10.0.11.0/24 |
| Private app AZ2   | 10.0.12.0/24 |
| Private DB AZ1    | 10.0.21.0/24 |
| Private DB AZ2    | 10.0.22.0/24 |

Components

* Internet Gateway
* NAT Gateway
* Route Tables

Architecture

```
VPC
 ├ Public Subnet
 │   ├ Bastion
 │   └ ALB
 │
 ├ Private App Subnet
 │   └ EKS Worker Nodes
 │
 └ Private DB Subnet
     └ Database
```

---

# 4. Create Bastion Host

Launch EC2 instance.

Service used:

* Amazon EC2

Configuration

* Instance type: t3.micro
* Subnet: Public subnet
* Security group: Allow SSH from your IP

Connect

```
ssh ec2-user@bastion-ip
```

Install tools on bastion

```
awscli
kubectl
eksctl
docker
```

---

# ADD the Tags for VPC components:

# Step 1 — Add Cluster Tag to ALL Subnets

Run this once:
```
aws ec2 create-tags \
--resources subnet-06ea5ab608efe0b3e subnet-0ac699bf561b6b6db subnet-05215f00dc1db4ed8 subnet-084755775252a1c91 subnet-02f65533c1ec14e30 subnet-07390f2084330b40c \
--tags Key=kubernetes.io/cluster/Healthcare-App-Cluster,Value=shared
```

#Step 2 — Tag Public Subnets

Run:
```
aws ec2 create-tags \
--resources subnet-06ea5ab608efe0b3e subnet-0ac699bf561b6b6db \
--tags Key=kubernetes.io/role/elb,Value=1
```

# Step 3 — Tag Private App Subnets

Run:

```
aws ec2 create-tags \
--resources subnet-05215f00dc1db4ed8 subnet-084755775252a1c91 \
--tags Key=kubernetes.io/role/internal-elb,Value=1
```

# Step 4 — DB Subnets

For these:

```
subnet-02f65533c1ec14e30
subnet-07390f2084330b40c
```
* Do nothing extra.

* Only the cluster tag from Step 1 is enough.


# 5. Create EKS Cluster

Create cluster configuration file.

`vi cluster.yaml`

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: Healthcare-App-Cluster
  region: ap-southeast-2
  version: "1.35"

vpc:
  id: vpc-0f87102e195a365f1

  subnets:
    public:
      public-1:
        id: subnet-06ea5ab608efe0b3e
      public-2:
        id: subnet-0ac699bf561b6b6db

    private:
      app-private-1:
        id: subnet-05215f00dc1db4ed8
      app-private-2:
        id: subnet-084755775252a1c91

managedNodeGroups:
  - name: healthcare-ng
    instanceType: t3.small
    desiredCapacity: 3
    minSize: 3
    maxSize: 5
    privateNetworking: true
    subnets:
      - subnet-05215f00dc1db4ed8
      - subnet-084755775252a1c91

```

Create cluster

```
eksctl create cluster -f cluster.yaml
```

This creates

* control plane
* node group
* security groups

Cluster service

* Amazon Elastic Kubernetes Service

---

# 6. Connect to EKS Cluster

Configure kubeconfig AND OIDC
```
eksctl utils associate-iam-oidc-provider \
    --region ap-southeast-2 \
    --cluster Healthcare-App-Cluster \
    --approve
```

```
aws eks update-kubeconfig \
--region ap-southeast-2 \
--name Healthcare-App-Cluster


```

Verify

```
kubectl get nodes
```

# If need to delete:

```
eksctl get nodegroup --cluster Healthcare-App-Cluster
eksctl delete nodegroup --cluster Healthcare-App-Cluster --name healthcare-app-ng
eksctl delete-cluster --name Healthcare-App-Cluster --region ap-southeast-2
```


---

# 7. Configure Node Autoscaling

Node autoscaling uses **Cluster Autoscaler**.

Install using Helm.

```
helm repo add autoscaler https://kubernetes.github.io/autoscaler
```

Deploy autoscaler

```
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
-n kube-system
```

Scaling logic

```
Pods increase
→ cluster autoscaler detects shortage
→ new EC2 nodes launched
```

Nodes run on

* Amazon EC2

---

# 8. Create Container Registry

Create repository using

* Amazon Elastic Container Registry

Create repository

```
aws ecr create-repository \
--repository-name healthcare-backend
```

Login

```
aws ecr get-login-password \
| docker login --username AWS --password-stdin <account>.dkr.ecr.ap-southeast-2.amazonaws.com
```

---

# 9. Build and Push Docker Image

Build image

```
docker build -t healthcare-backend .
```

Tag image

```
docker tag healthcare-backend:latest \
<account>.dkr.ecr.ap-southeast-2.amazonaws.com/healthcare-backend:latest
```

Push image

```
docker push \
<account>.dkr.ecr.ap-southeast-2.amazonaws.com/healthcare-backend:latest
```

---

# 10. Deploy Backend to Kubernetes

Create deployment file.

`backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: healthcare
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
        image: <ECR-IMAGE>
        ports:
        - containerPort: 3000
```

Deploy

```
kubectl apply -f backend-deployment.yaml
```

---

# 11. Create Backend Service

`backend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply

```
kubectl apply -f backend-service.yaml
```

---

# 12. Install ALB Ingress Controller

**ALB Ingress Controller** is actually the **correct production approach** for EKS. This avoids creating multiple load balancers for every service.

The controller integrates Kubernetes with:

* AWS Load Balancer Controller
  which provisions load balancers from
* Elastic Load Balancing
  inside your
* Amazon Elastic Kubernetes Service cluster.

---

# 1️⃣ Why ALB Ingress is Better

Without ingress:

```
service type = LoadBalancer
```

Every service creates **one ALB**.

Example:

```
frontend → ALB1
api → ALB2
admin → ALB3
```

This is **bad practice**.

---

With **ALB Ingress Controller**:

```
Single ALB
      │
Ingress
 ├── / → frontend
 ├── /api → backend
 └── /admin → admin
```

Only **one load balancer**.

---

# 2️⃣ Domain-Based Routing (Best for Healthcare App)

Instead of path routing you can use:

```
healthcare.com → frontend
api.healthcare.com → backend
admin.healthcare.com → admin
```

Ingress example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: healthcare-ingress
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: healthcare.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80

  - host: api.healthcare.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

---

# 3️⃣ Requirements Before Installing Controller

You must enable:

### OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
--cluster Healthcare-App-Cluster \
--approve
```

This enables **IAM roles for service accounts**.

---

# 4️⃣ Install AWS Load Balancer Controller

Download IAM policy:

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Create IAM policy:

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json
```

---

# 5️⃣ Create Service Account

```bash
eksctl create iamserviceaccount \
--cluster Healthcare-App-Cluster \
--namespace kube-system \
--name aws-load-balancer-controller \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```

---

# 6️⃣ Install Controller Using Helm

Install Helm if needed:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Add repo:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=Healthcare-App-Cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
```

---

# 7️⃣ Verify Controller

```bash
kubectl get pods -n kube-system
```

You should see:

```
aws-load-balancer-controller
```

---

# 8️⃣ What Happens Next

When you apply an ingress file:

```
kubectl apply -f ingress.yaml
```

The controller automatically creates:

```
Application Load Balancer
Target Groups
Listeners
Security Groups
```

in AWS.

---

# 9️⃣ Your Final Healthcare Architecture

```
Internet
     │
Application Load Balancer
     │
Ingress Controller
     │
Kubernetes Services
     │
Pods
     │
Database
```

Database hosted in:

* Amazon RDS

---

# 13. Create Ingress

`ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: healthcare-ingress
  annotations:
    kubernetes.io/ingress.class: alb
spec:
  rules:
  - host: api.healthcare.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
```

Apply

```
kubectl apply -f ingress.yaml
```

ALB will be created automatically.

---

# 14. Frontend Hosting

Frontend should be hosted using

* Amazon S3
* Amazon CloudFront

Steps

1 Create S3 bucket
2 Upload frontend build files
3 Enable static hosting
4 Create CloudFront distribution

---

# 15. Domain Configuration

Manage DNS using

* Amazon Route 53

Example DNS records

```
healthcare.com → CloudFront
api.healthcare.com → ALB
```

SSL certificates issued using

* AWS Certificate Manager

---

# 16. Database Setup

Create database using

* Amazon RDS

Configuration

* Private DB subnet
* Multi-AZ enabled
* Security group allowing EKS nodes

Example rule

```
Allow 3306 from EKS node security group
```

---

# 17. Install ArgoCD (GitOps)

Create namespace

```
kubectl create namespace argocd
```

Install ArgoCD

```
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check pods

```
kubectl get pods -n argocd
```

---

# 18. Access ArgoCD UI

Change service type

```
kubectl edit svc argocd-server -n argocd
```

Change

```
ClusterIP → LoadBalancer
```

Get URL

```
kubectl get svc argocd-server -n argocd
```

---

# 19. Deploy Applications with ArgoCD

Workflow

```
Developer pushes code
      ↓
CI builds Docker image
      ↓
Image pushed to ECR
      ↓
Kubernetes manifest updated in Git
      ↓
ArgoCD detects change
      ↓
Application deployed to EKS
```

GitOps tool used

* Argo CD

---

# Final Architecture

```
Users
 |
Route53
 |
CloudFront
 |
Frontend (S3)
 |
API Calls
 |
ALB
 |
Ingress
 |
EKS Cluster
 |
Backend Pods
 |
RDS
```

DevOps management

```
DevOps Engineer
      |
Bastion
      |
kubectl
      |
EKS
```

# NOTE: THE TROUBLESHOOTING STEPS ALSO MENTIONED IN OTHER PAGES.
# Prepared by:
*SHAIK MOULALI*
# DEVOPS ENGINEER
