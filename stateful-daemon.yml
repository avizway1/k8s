## 13 - DaemonSet in Kubernetes

### What is a DaemonSet?

A **DaemonSet** ensures that a **specific Pod runs on every (or selected) node** in your Kubernetes cluster.
It's ideal for running **background services** that should exist on **each node**, such as:

* Log collectors (e.g. Fluent Bit, Filebeat)
* Monitoring agents (e.g. Prometheus Node Exporter)
* Network plugins (e.g. Calico, Cilium)
* Security agents (e.g. Falco)

---

### Key Characteristics

* One Pod per node (unless node selectors/affinity is applied)
* Automatically runs on **newly added nodes**
* Deletes Pods when nodes are removed
* Often used for **infra-level workloads**

---

### 📄 Sample: DaemonSet with NGINX

This example runs an NGINX container on **every node** in the cluster.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemon
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx-daemon
  template:
    metadata:
      labels:
        app: nginx-daemon
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

---

### 🛠️ How to Deploy

```bash
kubectl apply -f nginx-daemonset.yaml
kubectl get daemonsets
kubectl get pods -o wide  # Observe 1 pod per node
```

---

### 🔍 Use nodeSelector to Limit to Specific Nodes

```yaml
spec:
  template:
    spec:
      nodeSelector:
        dedicated: monitoring
```

Make sure your target nodes are labeled accordingly:

```bash
kubectl label node <node-name> dedicated=monitoring
```

---
## 🧱 14 - StatefulSet in Kubernetes
---

### 📌 What is a StatefulSet?

A **StatefulSet** is a Kubernetes controller used to manage **stateful applications**. It ensures that Pods:

* Have a **persistent identity** (name, network)
* Get **stable, unique storage**
* Are **created and terminated in order**

Unlike Deployments, which manage stateless replicas, StatefulSets are designed for apps that **maintain state across Pod restarts**, like databases and clustered systems.

---

### ⚙️ Key Features

| Feature                | Description                                            |
| ---------------------- | ------------------------------------------------------ |
| **Stable Pod names**   | Pods get predictable names like `web-0`, `web-1`       |
| **Ordered deployment** | Pods start one by one in order                         |
| **Persistent volumes** | Each Pod gets its own `PersistentVolumeClaim`          |
| **Stable DNS**         | DNS names like `web-0.nginx.default.svc.cluster.local` |

---

### 🧠 Use Cases

* Databases (MySQL, PostgreSQL, Cassandra)
* Message queues (Kafka, RabbitMQ)
* Storage systems (ElasticSearch, HDFS)
* Any clustered app that requires **identity + persistence**

---

### 📡 Headless Service (Required)

A Headless Service in Kubernetes is a special kind of service without a ClusterIP. That means it does not load-balance traffic, but instead directly exposes the individual Pods behind it.

A headless service is needed to allow stable DNS for each Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None  # Important!
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

---

---

## ⚙️ How to Enable Amazon EBS CSI Driver on EKS (for StatefulSets & PVCs)

This guide walks you through enabling EBS-backed volumes on your EKS cluster using the **Amazon EBS CSI Driver**.

---

### ✅ Step 1: Connect `kubectl` to Your EKS Cluster

---

### 🔐 Step 2: Associate IAM OIDC Provider (One-time setup)

Set your cluster name:

```bash
cluster_name=ekswithavinash
```

```bash
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

---

### 🧾 Step 3: Create IAM Role for EBS CSI Driver

```bash
eksctl create iamserviceaccount   \
  --name ebs-csi-controller-sa   \
  --namespace kube-system        \
  --cluster ekswithavinash       \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy  \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

✅ **Explanation**:

* Creates the role without binding it to the service account yet (`--role-only`).
* Attaches `AmazonEBSCSIDriverPolicy` with correct permissions.

---

### 🔗 Step 4: Export Role ARN

```bash
export SERVICE_ACCOUNT_ROLE_ARN=arn:aws:iam::501170964283:role/AmazonEKS_EBS_CSI_DriverRole
```

Ensure the ARN matches the role you created.

---

### 📦 Step 5: Install the EBS CSI Driver Add-on

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster ekswithavinash \
  --service-account-role-arn $SERVICE_ACCOUNT_ROLE_ARN \
  --force
```

✅ **Description**:

* Deploys the Amazon EBS CSI driver as a managed add-on.
* Associates it with your custom IAM role for secure EBS operations.

---

### ✅ Final Validation

```bash
kubectl get daemonset ebs-csi-node -n kube-system
kubectl get pods -n kube-system
```

You should see the `ebs-csi-node` DaemonSet running on each node.

---

---

### 📄 Sample StatefulSet YAML



```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  # Must match the headless service
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: gp2  # Make sure gp2 exists in your EKS
        resources:
          requests:
            storage: 1Gi

```

---

### 🚀 Deploy It

```bash
kubectl apply -f nginx-headless-service.yaml
kubectl apply -f nginx-statefulset.yaml
```

Check Pod names:

```bash
kubectl get pods -o wide
```

You’ll see:

```
web-0
web-1
web-2
```

---

### 🔍 Verify Volume Stability

Even if a Pod is deleted (e.g., `web-1`), it will be re-created **with the same volume** and hostname.

---
---
## 🧱 15 - ConfigMap in Kubernetes
---
### 📌 What is a ConfigMap?

A **ConfigMap** is used to **store non-sensitive configuration data** in key-value format, outside of your container image.

You can inject this data into Pods as:

* Environment variables
* Command-line arguments
* Mounted files

---

### 🧠 Why use ConfigMap?

* Keep images **generic**, configuration **external**
* Change config without rebuilding container
* Share common config across multiple Pods

---

### 🛠️ Step-by-Step: Basic Example

Perfect — here's a **very simple, minimal working example** of using a ConfigMap in Kubernetes.

---

### ✅ 1. Create a ConfigMap

```bash
kubectl create configmap my-config --from-literal=GREETING="Hello from ConfigMap"
```

This creates a ConfigMap named `my-config` with a key-value pair:

```
GREETING=Hello from ConfigMap
```

---

### ✅ 2. Run a Pod That Uses It

```yaml
# configmap-simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo
spec:
  containers:
  - name: demo
    image: busybox
    command: ["sh", "-c", "echo $GREETING && sleep 3600"]
    env:
    - name: GREETING
      valueFrom:
        configMapKeyRef:
          name: my-config
          key: GREETING
```

Apply it:

```bash
kubectl apply -f configmap-simple-pod.yaml
```

---

### ✅ 3. Check the Output

```bash
kubectl logs configmap-demo
```

You should see:

```
Hello from ConfigMap
```
---
secret

## What are Secrets?
### A Secret is similar to a ConfigMap but used for sensitive data like passwords, tokens, and SSH keys.
They are stored Base64-encoded and are better than hardcoding secrets into Pod specs.

---

### ✅ 1. Create a Secret my-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  DB_PASSWORD: mydummysecret
```

This creates a Secret named `my-secret` with:

```
DB_PASSWORD = mydummysecret
```

---

### ✅ 2. Create a Pod That Uses the Secret

```yaml
# secret-simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: demo
    image: busybox
    command: ["sh", "-c", "echo $DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: DB_PASSWORD
```

Apply it:

```bash
kubectl apply -f secret-simple-pod.yaml
```

---

### ✅ 3. Check the Output

```bash
kubectl logs secret-demo
```

You should see:

```
mydummysecret
```

---

