When you create an **EKS cluster using `eksctl`**, you can configure the **nodegroup autoscaling** using the parameters:

* `--nodes-min` → Minimum number of nodes
* `--nodes-max` → Maximum number of nodes
* `--nodes` → Desired nodes at creation time

### Example Command

```bash
eksctl create cluster \
--name healthcare-app-cluster \
--region ap-southeast-2 \
--node-type t3.small \
--nodes 2 \
--nodes-min 1 \
--nodes-max 4 \
--zones ap-southeast-2a,ap-southeast-2b
```

### Explanation

* `--nodes 2` → Initial nodes created
* `--nodes-min 1` → Autoscaler minimum nodes
* `--nodes-max 4` → Autoscaler maximum nodes
* `--node-type t3.small` → Instance type
* `--zones ap-southeast-2a,ap-southeast-2b` → Worker nodes distributed across AZs

⚠️ Important: This only sets **Auto Scaling Group limits**.
To actually **scale nodes automatically based on pod demand**, you must install **Kubernetes Cluster Autoscaler** in your **Amazon Elastic Kubernetes Service** cluster.

---

## Better Approach (Production)

Create cluster **without nodegroup** and then create nodegroup separately.

### 1️⃣ Create Cluster

```bash
eksctl create cluster \
--name healthcare-app-cluster \
--region ap-southeast-2 \
--without-nodegroup
```

### 2️⃣ Create Nodegroup with Autoscaling

```bash
eksctl create nodegroup \
--cluster healthcare-app-cluster \
--region ap-southeast-2 \
--name ecom-ng \
--node-type t3.small \
--nodes 2 \
--nodes-min 1 \
--nodes-max 5 \
--node zones ap-southeast-2a,ap-southeast-2b
```

---

## Verify Nodegroup

```bash
eksctl get nodegroup --cluster healthcare-app-cluster
```
```
eksctl get nodegroup --cluster healthcare-app-cluster

eksctl delete nodegroup --cluster healthcare-app-cluster --name healthcare-app-ng

eksctl delete-cluster --name healthcare-app-cluster --region ap-southeast-2
```

---
