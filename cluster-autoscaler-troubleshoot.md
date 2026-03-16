# EKS Cluster Autoscaler – Real Troubleshooting Steps:

## 1️⃣ Problem Observed

Symptoms in the cluster:

```bash
kubectl get pods
```

Output:

```
many pods → Pending
```

But nodes were still:

```bash
kubectl get nodes
```

```
3 nodes only
```

Autoscaler **was not scaling nodes**.

---

# 2️⃣ Verify Autoscaler Pod

First check if autoscaler is running.

```bash
kubectl get pods -n kube-system | grep autoscaler
```

Expected:

```
cluster-autoscaler-xxxxx Running
```

If not running → check logs.

---

# 3️⃣ Check Autoscaler Logs

```bash
kubectl logs -n kube-system deployment/cluster-autoscaler
```

Errors seen earlier were related to **RBAC permissions**.

Example:

```
cannot list resource "resourceclaims"
cannot list resource "deviceclasses"
```

Meaning:

➡ Autoscaler **did not have permission** to access some Kubernetes resources.

---

# 4️⃣ Check ClusterRole

Check autoscaler cluster role:

```bash
kubectl get clusterrole cluster-autoscaler -o yaml
```

We inspected the **rules section**.

---

# 5️⃣ Edit ClusterRole

We edited the RBAC role.

```bash
kubectl edit clusterrole cluster-autoscaler
```

Then added missing API permissions.

---

# 6️⃣ Added Missing API Groups

We added this block:

```yaml
- apiGroups:
  - resource.k8s.io
  resources:
  - resourceclaims
  - deviceclasses
  verbs:
  - get
  - list
  - watch
```

This was required for **newer Kubernetes versions (1.30+)**.

---

# 7️⃣ Verify Updated RBAC

After editing:

```bash
kubectl get clusterrole cluster-autoscaler -o yaml
```

Confirm it includes:

```
resource.k8s.io
resourceclaims
deviceclasses
```

---

# 8️⃣ Restart Cluster Autoscaler

After RBAC changes always restart autoscaler.

```bash
kubectl rollout restart deployment cluster-autoscaler -n kube-system
```

Verify:

```bash
kubectl get pods -n kube-system
```

Autoscaler pod should restart.

---

# 9️⃣ Verify Nodegroup Scaling Config

Check nodegroup scaling:

```bash
aws eks describe-nodegroup \
--cluster-name Healthcare-App-Cluster \
--nodegroup-name healthcare-ng \
--region ap-southeast-2 \
--query "nodegroup.scalingConfig"
```

Output:

```
desiredSize = 3
minSize = 3
maxSize = 5
```

Meaning autoscaler can scale **up to 5 nodes**.

---

# 🔟 Verify Pending Pods Trigger

Pods were pending:

```
autoscaler-test-xxxxx Pending
```

This means **cluster capacity insufficient** → autoscaler should scale.

---

# 1️⃣1️⃣ Verify Auto Scaling Group Activity

Check ASG activity in AWS Console.

We found this error:

```
VcpuLimitExceeded
You have requested more vCPU capacity than your current vCPU limit of 8
```

Meaning AWS blocked new instances.

---

# 1️⃣2️⃣ Understand the Root Cause

Instance type used:

```
t3.small
```

vCPU per node:

```
2 vCPU
```

Cluster calculation:

```
3 nodes = 6 vCPU
5 nodes = 10 vCPU
```

But AWS quota:

```
8 vCPU
```

So instance launch failed.

---

# 1️⃣3️⃣ Fix Option

Two possible fixes:

### Option A – Reduce max nodes

```
maxSize = 4
```

Command:

```bash
aws eks update-nodegroup-config \
--cluster-name Healthcare-App-Cluster \
--nodegroup-name healthcare-ng \
--region ap-southeast-2 \
--scaling-config minSize=3,maxSize=4,desiredSize=3
```

---

### Option B – Increase AWS Quota (Recommended)

Go to:

```
AWS Console
→ Service Quotas
→ EC2
```

Increase:

```
Running On-Demand Standard instances
```

Example:

```
8 → 32 vCPU
```

---

# 1️⃣4️⃣ Monitor Node Scaling

Use this command to watch nodes:

```bash
watch kubectl get nodes
```

Expected scaling:

```
3 → 4 → 5 nodes
```

---

# 1️⃣5️⃣ Test Autoscaling

Scale deployment:

```bash
kubectl scale deployment autoscaler-test --replicas=100
```

Flow:

```
Pods Pending
↓
Cluster Autoscaler detects
↓
ASG launches EC2
↓
Node joins cluster
↓
Pods Running
```

---

# Final Root Causes We Fixed

1️⃣ Missing **RBAC permissions**
2️⃣ Added **resource.k8s.io API group**
3️⃣ Restarted autoscaler
4️⃣ Checked nodegroup scaling config
5️⃣ Identified **AWS vCPU quota limit**

---

# Pro Tip (Interview Point)

Cluster Autoscaler troubleshooting usually involves checking:

```
1. Pending Pods
2. Autoscaler logs
3. RBAC permissions
4. ASG discovery tags
5. Nodegroup scaling limits
6. AWS quotas
```

If you remember these **6 steps**, you can debug autoscaling issues easily in production.

---


root@ip-10-0-8-83:~# kubectl get clusterrole cluster-autoscaler -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRole","metadata":{"annotations":{},"labels":{"k8s-addon":"cluster-autoscaler.addons.k8s.io","k8s-app":"cluster-autoscaler"},"name":"cluster-autoscaler"},"rules":[{"apiGroups":[""],"resources":["events","endpoints"],"verbs":["create","patch"]},{"apiGroups":[""],"resources":["pods/eviction"],"verbs":["create"]},{"apiGroups":[""],"resources":["pods/status"],"verbs":["update"]},{"apiGroups":[""],"resourceNames":["cluster-autoscaler"],"resources":["endpoints"],"verbs":["get","update"]},{"apiGroups":[""],"resources":["nodes"],"verbs":["watch","list","get","update"]},{"apiGroups":[""],"resources":["namespaces","pods","services","replicationcontrollers","persistentvolumeclaims","persistentvolumes"],"verbs":["watch","list","get"]},{"apiGroups":["extensions"],"resources":["replicasets","daemonsets"],"verbs":["watch","list","get"]},{"apiGroups":["policy"],"resources":["poddisruptionbudgets"],"verbs":["watch","list"]},{"apiGroups":["apps"],"resources":["statefulsets","replicasets","daemonsets"],"verbs":["watch","list","get"]},{"apiGroups":["storage.k8s.io"],"resources":["storageclasses","csinodes","csidrivers","csistoragecapacities","volumeattachments"],"verbs":["watch","list","get"]},{"apiGroups":["batch","extensions"],"resources":["jobs"],"verbs":["get","list","watch","patch"]},{"apiGroups":["coordination.k8s.io"],"resources":["leases"],"verbs":["create"]},{"apiGroups":["coordination.k8s.io"],"resourceNames":["cluster-autoscaler"],"resources":["leases"],"verbs":["get","update"]}]}
  creationTimestamp: "2026-03-16T09:40:26Z"
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  managedFields:
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:labels:
          .: {}
          f:k8s-addon: {}
          f:k8s-app: {}
    manager: kubectl-client-side-apply
    operation: Update
    time: "2026-03-16T09:40:26Z"
  - apiVersion: rbac.authorization.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:rules: {}
    manager: kubectl-edit
    operation: Update
    time: "2026-03-16T10:28:42Z"
  name: cluster-autoscaler
  resourceVersion: "31528"
  uid: e0a7c820-1baa-4bc9-9da2-8e8ab9b513b6
rules:
- apiGroups:
  - ""
  resources:
  - events
  - endpoints
  verbs:
  - create
  - patch
- apiGroups:
  - ""
  resources:
  - pods/eviction
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - pods/status
  verbs:
  - update
- apiGroups:
  - ""
  resourceNames:
  - cluster-autoscaler
  resources:
  - endpoints
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - watch
  - list
  - get
  - update
- apiGroups:
  - ""
  resources:
  - namespaces
  - pods
  - services
  - replicationcontrollers
  - persistentvolumeclaims
  - persistentvolumes
  verbs:
  - watch
  - list
  - get
- apiGroups:
  - extensions
  resources:
  - replicasets
  - daemonsets
  verbs:
  - watch
  - list
  - get
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - watch
  - list
- apiGroups:
  - apps
  resources:
  - statefulsets
  - replicasets
  - daemonsets
  verbs:
  - watch
  - list
  - get
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - csinodes
  - csidrivers
  - csistoragecapacities
  - volumeattachments
  verbs:
  - watch
  - list
  - get
- apiGroups:
  - batch
  - extensions
  resources:
  - jobs
  verbs:
  - get
  - list
  - watch
  - patch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - create
  - get
  - list
  - watch
  - update
- apiGroups:
  - coordination.k8s.io
  resourceNames:
  - cluster-autoscaler
  resources:
  - resourceclaims
  - deviceclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - resource.k8s.io
  resources:
  - resourceclaims
  - deviceclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - resource.k8s.io
  resources:
  - resourceslices
  verbs:
  - get
  - list
  - watch
root@ip-10-0-8-83:~#


---
# Test

```
kubectl create deployment autoscaler-test --image=nginx
kubectl scale deployment autoscaler-test --replicas=20
```

