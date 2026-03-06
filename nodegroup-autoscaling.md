When you create an **EKS cluster using `eksctl`**, you can configure the **nodegroup autoscaling** using the parameters:

* `--nodes-min` → Minimum number of nodes
* `--nodes-max` → Maximum number of nodes
* `--nodes` → Desired nodes at creation time

### Example Command

```bash
eksctl create cluster \
--name Shaik-Ecom-cluster \
--region us-east-1 \
--node-type t3.small \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--zones us-east-1a,us-east-1b
```

### Explanation

* `--nodes 2` → Initial nodes created
* `--nodes-min 1` → Autoscaler minimum nodes
* `--nodes-max 4` → Autoscaler maximum nodes
* `--node-type t3.small` → Instance type
* `--zones us-east-1a,us-east-1b` → Worker nodes distributed across AZs

⚠️ Important: This only sets **Auto Scaling Group limits**.
To actually **scale nodes automatically based on pod demand**, you must install **Kubernetes Cluster Autoscaler** in your **Amazon Elastic Kubernetes Service** cluster.

---

## Better Approach (Production)

Create cluster **without nodegroup** and then create nodegroup separately.

### 1️⃣ Create Cluster

```bash
eksctl create cluster \
--name Shaik-Ecom-cluster \
--region us-east-1 \
--without-nodegroup
```

### 2️⃣ Create Nodegroup with Autoscaling

```bash
eksctl create nodegroup \
--cluster Shaik-Ecom-cluster \
--region us-east-1 \
--name ecom-ng \
--node-type t3.small \
--nodes 2 \
--nodes-min 1 \
--nodes-max 5 \
--zones us-east-1a,us-east-1b
```

---

## Verify Nodegroup

```bash
eksctl get nodegroup --cluster Shaik-Ecom-cluster
```

---
