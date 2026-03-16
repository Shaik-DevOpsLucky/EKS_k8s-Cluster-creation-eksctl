# Kubernetes Cluster Autoscaler on AWS EKS

## Introduction

In Kubernetes environments, workloads can dynamically increase or decrease depending on traffic and application demand. Managing cluster capacity manually can lead to resource shortages or unnecessary infrastructure costs.

**Cluster Autoscaler** is a Kubernetes component that automatically adjusts the number of worker nodes in a cluster when workloads require more resources or when nodes are underutilized.

It integrates with cloud provider auto scaling mechanisms (such as AWS Auto Scaling Groups) to dynamically add or remove nodes based on Kubernetes scheduling needs.

This project demonstrates how to deploy **Cluster Autoscaler on an AWS EKS cluster** using the official Kubernetes autoscaler manifest and configure it to automatically scale worker nodes.

---

# What is Cluster Autoscaler?

Cluster Autoscaler is a Kubernetes controller that automatically adjusts the number of nodes in a cluster.

It performs two primary actions:

• **Scale Up** – Adds new nodes when pods cannot be scheduled due to insufficient resources.
• **Scale Down** – Removes underutilized nodes when workloads decrease.

Cluster Autoscaler works by monitoring the Kubernetes scheduler and interacting with the underlying cloud infrastructure.

---

# How Cluster Autoscaler Works

The Cluster Autoscaler continuously watches the Kubernetes cluster for pending pods and unused nodes.

### Scale-Up Process

1. A user deploys a workload.
2. Kubernetes tries to schedule pods on existing nodes.
3. If resources are insufficient, pods remain in **Pending** state.
4. Cluster Autoscaler detects the pending pods.
5. It increases the capacity of the AWS Auto Scaling Group.
6. A new node is created and joins the cluster.
7. Kubernetes scheduler places the pending pods on the new node.

### Scale-Down Process

1. Cluster Autoscaler monitors node utilization.
2. If a node is underutilized for a certain period:
3. Pods are safely rescheduled to other nodes.
4. The node is terminated to reduce cost.

---

# Architecture Flow

```
User Deploys Pods
        │
        ▼
Kubernetes Scheduler
        │
        ▼
Pods Pending (Insufficient Resources)
        │
        ▼
Cluster Autoscaler Detects Pending Pods
        │
        ▼
AWS Auto Scaling Group Scale Up
        │
        ▼
New EC2 Node Created
        │
        ▼
Node Joins EKS Cluster
        │
        ▼
Pods Scheduled Successfully
```

---

# Why Do We Need Cluster Autoscaler?

Without autoscaling, cluster administrators must manually increase or decrease node capacity.

Problems without autoscaling:

• Resource shortages during traffic spikes
• Over-provisioning leading to higher costs
• Manual infrastructure management
• Poor resource utilization

Cluster Autoscaler solves these problems by dynamically managing cluster capacity.

---

# Why Can't We Use AWS Auto Scaling Policies Alone?

AWS Auto Scaling Groups support scaling policies based on metrics like CPU or network usage. However, Kubernetes scheduling decisions are not based on these metrics alone.

Key limitations of AWS scaling policies:

| AWS ASG Scaling                           | Cluster Autoscaler               |
| ----------------------------------------- | -------------------------------- |
| Scales based on CPU/metrics               | Scales based on pending pods     |
| No awareness of Kubernetes scheduler      | Fully integrated with Kubernetes |
| May scale even when pods don't need nodes | Adds nodes only when required    |
| Cannot safely drain nodes                 | Gracefully evicts pods           |

Cluster Autoscaler understands **Kubernetes scheduling constraints**, which AWS scaling policies cannot.

Therefore, Cluster Autoscaler is the recommended solution for Kubernetes clusters.

---

# Prerequisites

Before installing Cluster Autoscaler ensure the following:

• AWS EKS cluster is created
• Worker node group is created
• Node groups are backed by **Auto Scaling Groups**
• Proper ASG tags are configured
• `kubectl` is configured to access the cluster

Required ASG Tags:

```
k8s.io/cluster-autoscaler/enabled = true
k8s.io/cluster-autoscaler/<cluster-name> = owned
```

Example:

```
k8s.io/cluster-autoscaler/enabled=true
k8s.io/cluster-autoscaler/dev-eks-cluster=owned
```

---

# Step 1 – Deploy Cluster Autoscaler Manifest

Apply the official Kubernetes autoscaler manifest.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

---

# Step 2 – Add Required Annotation

Add the `safe-to-evict` annotation to prevent autoscaler pods from being evicted during scaling operations.

```
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

---

# Step 3 – Edit Autoscaler Deployment

Edit the deployment configuration.

```
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

Update the container command section.

Replace:

```
<YOUR CLUSTER NAME>
```

Add the following parameters:

```
--balance-similar-node-groups
--skip-nodes-with-system-pods=false
```

Example configuration:

```
spec:
  containers:
  - command:
    - ./cluster-autoscaler
    - --v=4
    - --stderrthreshold=info
    - --cloud-provider=aws
    - --skip-nodes-with-local-storage=false
    - --expander=least-waste
    - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
**  - --balance-similar-node-groups
    - --skip-nodes-with-system-pods=false**
```

Save the file to apply the changes.

---

# Step 4 – Update Autoscaler Image Version

Update the Cluster Autoscaler image to match your Kubernetes version.

```
kubectl -n kube-system set image deployment.apps/cluster-autoscaler \
cluster-autoscaler=us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.15.n
```

You can also use newer registry versions such as:

```
registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
```
# NOTE: ALWAYS CHECK AND find the latest Cluster Autoscaler version that matches your cluster’s Kubernetes major and minor version follow the below link.
---
https://github.com/kubernetes/autoscaler/releases/tag/cluster-autoscaler-1.35.0

https://github.com/kubernetes/autoscaler/releases?source=post_page-----4aab8c89f9a1---------------------------------------

----
# reference:
https://katharharshal1.medium.com/kubernetes-cluster-autoscaling-ca-using-aws-eks-4aab8c89f9a1
# ✅ Rule You Must Follow (Official Kubernetes Rule)

> **Cluster Autoscaler version MUST match the Kubernetes minor version.**

Meaning:

| Kubernetes (EKS) Version | Cluster Autoscaler Version |
| ------------------------ | -------------------------- |
| 1.33                     | v1.33.x                    |
| 1.34                     | v1.34.x                    |
| **1.35**                 | ✅ **v1.35.x**              |

❌ Using older versions (like v1.15, v1.27, etc.) will cause:

* autoscaler not detecting node groups
* scale-up failures
* API incompatibility errors
* silent scaling issues

---

# ✅ Correct Image for EKS 1.35

Use:
Can manually update or use the below command

```bash
**registry.k8s.io/autoscaling/cluster-autoscaler:v1.35.0**
```

Update image:

```bash
kubectl -n kube-system set image deployment/cluster-autoscaler \
cluster-autoscaler=registry.k8s.io/autoscaling/cluster-autoscaler:v1.35.0
```

# Verify pods are running or not:
```
Your deployment **does NOT contain AWS region environment variables**, so the autoscaler cannot call AWS APIs. That’s why it is crashing.

Your current container section looks like this:

```yaml
containers:
- command:
  - ./cluster-autoscaler
  - --v=4
  - --stderrthreshold=info
  - --cloud-provider=aws
```

But it is **missing this**:

```yaml
env:
- name: AWS_REGION
  value: ap-southeast-2
- name: AWS_DEFAULT_REGION
  value: ap-southeast-2
```

---

# ✅ Fix it in 1 command (Best way)

Instead of editing manually, run this **patch command**:

```bash
kubectl -n kube-system patch deployment cluster-autoscaler \
--type='json' \
-p='[{"op":"add","path":"/spec/template/spec/containers/0/env","value":[{"name":"AWS_REGION","value":"ap-southeast-2"},{"name":"AWS_DEFAULT_REGION","value":"ap-southeast-2"}]}]'
```

---

# ✅ After running patch

Restart the deployment:

```bash
kubectl -n kube-system rollout restart deployment cluster-autoscaler
```

---

# ✅ Verify

Check pod:

```bash
kubectl get pods -n kube-system | grep autoscaler
```

Expected:

```
cluster-autoscaler-xxxxx   1/1   Running
```

---

# ✅ Verify logs

```bash
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

Now you should see something like:

```
Building aws cloud provider
Regenerating instance to ASG map
```

No more **Missing Region error**.

---
# ✅ How to Verify Your Cluster Version

```bash
kubectl version --short
```

or

```bash
aws eks describe-cluster \
--name <cluster-name> \
--query "cluster.version"
```

Expected output:

```
1.35
```

---

# ✅ How to Verify Autoscaler Version

```bash
kubectl -n kube-system describe pod <autoscaler-pod-name> | grep Image
```

---

# ⚠️ Common Mistake (Very Important)

Many tutorials still use:

```
us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler
```

👉 This registry is **deprecated**.

✅ Always use:

```
registry.k8s.io/autoscaling/cluster-autoscaler
```

---

# ✅ Compatibility Checklist for EKS 1.35

Make sure you have:

* ✅ ASG tags configured
* ✅ IAM role permissions (IRSA recommended)
* ✅ Autoscaler version = 1.35.x
* ✅ Node group min/max configured
* ✅ Correct cluster name in autodiscovery flag

---

---

# Step 5 – Verify Installation

Check if the autoscaler pod is running.

```
kubectl get pods -n kube-system | grep autoscaler
```

---

# Step 6 – Monitor Autoscaler Logs

View autoscaler logs.

```
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

Logs will show scaling decisions such as:

• Detecting pending pods
• Increasing node group size
• Removing underutilized nodes

---

# Testing Autoscaling

Deploy a workload that requires more resources than available nodes.

Example:

```
kubectl run nginx-test --image=nginx --replicas=20
```

Observe cluster scaling:

```
kubectl get nodes
```

You should see new nodes being created automatically.

---

# Benefits of Cluster Autoscaler

• Automatic infrastructure scaling
• Better resource utilization
• Reduced cloud costs
• Improved application availability
• Fully integrated with Kubernetes scheduling

---

# Conclusion

Cluster Autoscaler is an essential component for running scalable Kubernetes workloads. It dynamically adjusts cluster capacity by adding nodes when workloads increase and removing unused nodes when demand decreases.

Using Cluster Autoscaler with AWS EKS ensures that applications always have the resources they need while optimizing infrastructure cost.

---
# PREPARED BY:
*SHAIK MOULALI*
# *DEVOPS ENGINEER*
