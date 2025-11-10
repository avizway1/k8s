# **What is Vertical Pod Autoscaler (VPA)?**  
Vertical Pod Autoscaler (VPA) is a **Kubernetes component** that **automatically adjusts CPU and memory** requests and limits for running pods. Instead of scaling pods horizontally (adding more replicas), VPA **modifies the resource allocation** of existing pods to optimize performance.

VPA is an open-source project developed by the Kubernetes community and is available on GitHub under the Kubernetes Autoscaler repository.

### **Why is VPA Not Recommended for Production Workloads?**  
Despite its advantages, VPA has limitations that make it **less suitable for production environments**:

1. **Pod Restarts Are Required**  
   - VPA **terminates and restarts** pods to apply new resource requests/limits.  
   - This disrupts running applications and is **not ideal for stateful workloads** or applications requiring **high availability**.

2. **Conflicts with Horizontal Pod Autoscaler (HPA)**  
   - **HPA and VPA do not work well together** because HPA scales out based on CPU/memory usage, while VPA tries to resize existing pods.  
   - **Best practice:** Use **HPA for stateless workloads** and **VPA for batch jobs or long-running background processes**.

3. **Unpredictable Scaling Decisions** 
   - VPA relies on historical usage data, which may **not react quickly** to sudden traffic spikes.  
   - This can cause **latency issues** in dynamic environments.

4. **Not Supported in Managed Kubernetes Services (EKS, AKS, GKE)** 
   - AWS EKS, Azure AKS, and Google GKE **do not natively support VPA**, as they encourage **HPA + Cluster Autoscaler** for scaling.


# Understanding and Testing Vertical Pod Autoscaler (VPA)

## **Step 1: Install Vertical Pod Autoscaler (VPA)**
To install VPA, first download the source code from the Kubernetes autoscaler repository:

```sh
# Clone the autoscaler repository
git clone https://github.com/kubernetes/autoscaler.git

# Navigate to the vertical-pod-autoscaler directory
cd /autoscaler/vertical-pod-autoscaler/
```

Now, install VPA by running the following command:

```sh
./hack/vpa-up.sh
```

## **Step 2: Verify Installation**
Once VPA is installed, verify that the necessary components are running:

```sh
kubectl get pods -n kube-system | grep vpa
```

Expected output:
```
vpa-admission-controller-xxxxx Running
vpa-recommender-xxxxx Running
vpa-updater-xxxxx Running
```

## **Step 3: Deploy Example Application (Hamster)**
Deploy the `hamster.yaml` file located under the `examples` directory:

```sh
kubectl apply -f ./examples/hamster.yaml
```

Verify that the pod is running:

```sh
kubectl get pods
```


## **Step 4: Get VPA Recommendations**
To get the resource recommendations, describe the VPA object:

```sh
kubectl describe vpa hamster-vpa
```

Sample output:
```
Recommendation:
  Container Recommendations:
    Container Name:  hamster
    Lower Bound:
      Cpu:     100m
      Memory:  262144k
    Target:
      Cpu:     548m
      Memory:  262144k
    Uncapped Target:
      Cpu:     548m
      Memory:  262144k
    Upper Bound:
      Cpu:     1
      Memory:  500Mi
```

## **Step 5: Understanding the Recommendation**
### **Lower Bound (Minimum Required Resources)**
- **CPU:** `100m` → The container should have at least **0.1 CPU cores** (100 milliCPUs).
- **Memory:** `262144k` (or **256MiB**) → The minimum memory required to run safely.

### **Target (Optimal Resource Allocation)**
- **CPU:** `548m` → The container is expected to use **0.548 CPU cores** (548 milliCPUs) for optimal performance.
- **Memory:** `262144k` (or **256MiB**) → The optimal memory suggested for smooth operation.

### **Uncapped Target (Theoretical Best Case Without Constraints)**
- **CPU:** `548m` → The same as the **Target CPU** (i.e., no external limitations are affecting this).
- **Memory:** `262144k` (or **256MiB**) → The same as the **Target Memory**.

### **Upper Bound (Maximum Recommended Resources Before Wasting CPU/Memory)**
- **CPU:** `1` → The container should not use more than **1 full CPU core**.
- **Memory:** `500Mi` → The container should not use more than **500MiB of RAM**, beyond which it may be considered over-allocated.

## **Step 6: Apply the Recommendation for Better Performance**
Based on the recommendation, update your deployment resource requests and limits for optimal performance:



============
## Using goldilocks to get more recommendations

---

### **1. Install Helm (if not already installed)**
```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

### **2. Add the Goldilocks Helm Repository and Install**
```sh
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
kubectl create namespace goldilocks
helm install goldilocks --namespace goldilocks fairwinds-stable/goldilocks
```

---

### **3. Enable Goldilocks for Namespace**
```sh
kubectl label ns goldilocks goldilocks.fairwinds.com/enabled=true
kubectl label namespace default goldilocks.fairwinds.com/enabled=true
```

---

### **4. Expose Goldilocks Dashboard Externally**
By default, Goldilocks deploys a **ClusterIP** service, which is accessible only inside the cluster. To access it externally, change the service type to **LoadBalancer**.

#### **Change Service to LoadBalancer**
```sh
kubectl patch svc goldilocks-dashboard -n goldilocks -p '{"spec": {"type": "LoadBalancer"}}'
```
- Get the external IP:
  ```sh
  kubectl get svc goldilocks-dashboard -n goldilocks
  ```
