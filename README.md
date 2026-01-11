# EKS_k8s-Cluster-creation-eksctl
## This repo contains the steps to create the EKS/K8s-Cluster-creation using eksctl

---
## ☸️ Step 4: Creation of EKS Cluster

### 4.1. Create IAM User

> ⚠️ **Important:** Never use Root Account to create EKS Cluster

Create a dedicated IAM user for EKS cluster management.

---

### 4.2. Attach Policies to the User

Attach the following AWS managed policies:

- ✅ AmazonEC2FullAccess
- ✅ AmazonEKS_CNI_Policy
- ✅ AmazonEKSClusterPolicy
- ✅ AmazonEKSWorkerNodePolicy
- ✅ AWSCloudFormationFullAccess
- ✅ IAMFullAccess

**Additionally, attach this inline policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
    }
  ]
}
```

---

### 4.3. Create Access Keys

Generate **Access Keys** for the IAM user created above.  
Save the **Access Key ID** and **Secret Access Key** securely.

---

### 4.4. Install AWS CLI
```bash
sudo apt update
curl -o awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```

**Configure AWS CLI:**
```bash
aws configure
```

Enter your Access Key ID, Secret Access Key, region (e.g., `us-east-1`), and output format when prompted.

---

## NOTE: IF you don't wish to use SECRETE_KEY & SECRETE_ACCESS_KEYS you can use the IAM ROle with ADMINISTRATOR policy and attach that policy to EC2 Instance.

---

### 4.5. Install kubectl
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
```

**Verify installation:**
```bash
kubectl version --short --client
```

---

### 4.6. Install eksctl
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

**Verify installation:**
```bash
eksctl version
```

---

### 4.7. Create EKS Cluster
```bash
eksctl create cluster --name Shaik-Ecom-cluster --region us-east-1 --node-type t3.small --zones us-east-1a,us-east-1b
```

> ⏳ **Note:** Cluster creation takes approximately 15-20 minutes

---
## OIDC
```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster Shaik-Ecom-cluster \
    --approve
```

# Update ./kube/config file

```bash
aws eks update-kubeconfig --name Shaik-Ecom-cluster
```
---

### 4.10. Delete Cluster (Optional)

If you need to delete the cluster:
```bash
eksctl delete cluster --name Shaik-Ecom-cluster --region us-east-1
```


