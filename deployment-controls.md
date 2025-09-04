
# Kubernetes Advanced Deployment Controls: Taints, Affinities, Probes, and More

This guide explains **Taints & Tolerations**, **Affinities**, **Probes**, and other advanced deployment controls in Kubernetes. Each section includes real-life analogies, ready-to-deploy YAML manifests, and clear descriptions for production use.

---

## 1. Affinities (Where Pods Should Run)

### 🧩 Simple Analogy
- **Node Affinity:** “I prefer a computer with SSD storage.”
- **Pod Affinity:** “I want to sit close to my friend.” (Pods stick together).
- **Pod Anti-Affinity:** “I don’t want to sit next to my friend.” (Pods spread out).

### 🛠 Node Affinity Example: Deploy on SSD Nodes Only. This deployment ensures pods only run on nodes labeled with `disktype=ssd`.
```yamlapiVersion: apps/v1
kind: Deployment
metadata:
  name: ssd-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ssd-app
  template:
    metadata:
      labels:
        app: ssd-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disktype
                    operator: In
                    values:
                      - ssd
      tolerations:
        - key: role
          operator: Equal
          value: db
          effect: NoSchedule
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80

```
**Add Label**

kubectl label nodes ip-192-168-60-251.ap-south-1.compute.internal disktype=ssd

---

### 🛠 Pod Anti-Affinity Example: Spread Pods Across Nodes
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web
                topologyKey: "kubernetes.io/hostname"
      tolerations:
        - key: role
          operator: Equal
          value: db
          effect: NoSchedule
      containers:
        - name: web
          image: httpd:2.4
          ports:
            - containerPort: 80
```
**Description:**
Pods with the label `app=web` will be scheduled on different nodes, maximizing availability.

---


## 2. Taints & Tolerations

### 🧩 Simple Analogy
- **Taint (Node):** Like a bus seat marked "Reserved".
- **Toleration (Pod):** The special pass that lets you sit there.
👉 Without the pass, you cannot sit on the reserved seat.

### Real-world Use Cases:
- Dedicated nodes for databases, GPU workloads, or specific applications
- Isolating workloads by team, environment, or resource requirements
- Preventing general workloads from consuming specialized hardware

### 🛠 Example: Taint a Node and Deploy a Tolerating Pod
**Step 1: Taint a node so only special pods can run:**
```bash
kubectl taint nodes <node1-name> role=db:NoSchedule
```

**Step 2: Deploy a Pod that can tolerate the taint: Pod WITH toleration (db-pod): ✅ Running on tainted node**

**Description:**
This pod will only be scheduled on nodes tainted with `role=db:NoSchedule`. The toleration allows it to "sit on the reserved seat".

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  nodeSelector:
    disktype: ssd
  tolerations:
    - key: "role"
      operator: "Equal"
      value: "db"
      effect: "NoSchedule"
  containers:
    - name: mysql
      image: mysql:8.0
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: example
      ports:
        - containerPort: 3306
```
---

**Pod WITHOUT toleration (no-toleration-pod): ❌ Pending (blocked by taint)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
spec:
  nodeSelector:
    disktype: ssd  # This will try to force it to the tainted node
  containers:
    - name: nginx
      image: nginx:1.25
```


---

## 3. Probes (Health Checks)

### 🧩 Simple Analogy
Probes are like a doctor checking a patient’s condition:
- **Liveness Probe:** “Are you alive? If not, restart.”
- **Readiness Probe:** “Are you ready to serve customers? If not, don’t send traffic.”
- **Startup Probe:** “Need extra wake-up time? I’ll wait.”

### 🛠 Example: Deployment with All Probes
```yaml
# KUBERNETES HEALTH PROBES DEMONSTRATION
# 
# THREE TYPES OF PROBES:
# 1. STARTUP PROBE: Checks if container has started successfully
# 2. READINESS PROBE: Checks if container is ready to receive traffic
# 3. LIVENESS PROBE: Checks if container is still running healthy
#
# PROBE EXECUTION ORDER: Startup → Readiness → Liveness (ongoing)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthcheck-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: healthcheck
  template:
    metadata:
      labels:
        app: healthcheck
    spec:
      containers:
        - name: app
          # Using nginx with custom health endpoints via configmap
          image: nginx:1.25
          ports:
            - containerPort: 80
          
          # STARTUP PROBE: Runs FIRST, before other probes
          # Purpose: Protects slow-starting containers from being killed by liveness probe
          # Behavior: Disables liveness/readiness until startup succeeds
          startupProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10    # Wait 10s before first check
            periodSeconds: 5           # Check every 5s
            failureThreshold: 6        # Allow 6 failures (30s total)
            timeoutSeconds: 3          # 3s timeout per check
          
          # READINESS PROBE: Determines if pod should receive traffic
          # Purpose: Controls service endpoint inclusion
          # Behavior: Pod removed from service if fails, added back when passes
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5     # Start checking after 5s
            periodSeconds: 10          # Check every 10s
            failureThreshold: 3        # 3 failures = not ready
            successThreshold: 1        # 1 success = ready
            timeoutSeconds: 5          # 5s timeout per check
          
          # LIVENESS PROBE: Determines if container should be restarted
          # Purpose: Detects deadlocks, infinite loops, unresponsive states
          # Behavior: Kills and restarts container if fails
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15    # Wait 15s before first check
            periodSeconds: 20          # Check every 20s
            failureThreshold: 3        # 3 failures = restart container
            timeoutSeconds: 5          # 5s timeout per check
          
          # Resource limits to prevent resource exhaustion
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"

---
# SERVICE TO DEMONSTRATE READINESS PROBE EFFECT
apiVersion: v1
kind: Service
metadata:
  name: healthcheck-service
spec:
  selector:
    app: healthcheck
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
**Description:**
This deployment uses all three probe types to ensure the app is healthy, ready, and given time to start up.

---


## 4. Deployment Timing Options

- **minReadySeconds:** Delay before a pod is considered available after becoming ready (helps avoid sending traffic too soon).
- **progressDeadlineSeconds:** Max time for a rollout to complete before it's considered failed.
- **terminationGracePeriodSeconds:** Time for a pod to shut down gracefully before being killed.

### 🛠 Example: Deployment with Timing Controls
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timing-app
spec:
  replicas: 2
  minReadySeconds: 10
  progressDeadlineSeconds: 600
  selector:
    matchLabels:
      app: timing
  template:
    metadata:
      labels:
        app: timing
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
```
**Description:**
This deployment ensures pods are only marked available after 10 seconds, rollouts fail after 10 minutes, and pods have 30 seconds to shut down gracefully.

---


## 5. Other Controls

### Pod Disruption Budgets (PDB)
**Purpose:** Ensures only a limited number of pods are down during voluntary disruptions (e.g., node drain, upgrades).

# PDB BENEFITS:
- Prevents complete service outage during maintenance
- Ensures gradual, controlled pod eviction
- Maintains application availability during cluster operations

```yaml
# POD DISRUPTION BUDGET (PDB) DEMONSTRATION
#
# WHAT IS PDB?
# - Ensures minimum number of pods stay running during voluntary disruptions
# - Protects applications from becoming unavailable during maintenance
# - Only applies to VOLUNTARY disruptions (not node failures or crashes)
#
# VOLUNTARY DISRUPTIONS: Node drain, cluster upgrade, scaling down
# INVOLUNTARY DISRUPTIONS: Node crash, hardware failure, kernel panic

---
# STEP 1: Create a deployment with multiple replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 5  # We want 5 pods running
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi

---
# STEP 2: Create Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  # SELECTOR: Which pods this PDB applies to
  selector:
    matchLabels:
      app: web-app  # Must match deployment labels
  
  # OPTION 1: Minimum available pods (choose one option)
  minAvailable: 3  # Always keep at least 3 pods running
  
  # OPTION 2: Maximum unavailable pods (alternative to minAvailable)
  # maxUnavailable: 2  # Allow maximum 2 pods to be down
  # maxUnavailable: "40%"  # Allow 40% of pods to be unavailable

# HOW IT WORKS:
# - With 5 replicas and minAvailable: 3
# - Kubernetes will only allow 2 pods to be disrupted at once
# - If you try to drain a node with 3 pods, only 2 will be evicted
# - The 3rd eviction will be blocked until other pods are rescheduled

---
# STEP 3: Service to demonstrate availability
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP

# TESTING SCENARIOS:
#
# 1. NORMAL OPERATION:
#    kubectl get pods -l app=web-app
#    Result: 5 pods running
#
# 2. VOLUNTARY DISRUPTION (Node Drain):
#    kubectl drain <node-name> --ignore-daemonsets
#    Result: Only 2 pods evicted at once, 3 remain available
#
# 3. CHECK PDB STATUS:
#    kubectl get pdb web-app-pdb
#    Shows: ALLOWED DISRUPTIONS and current status
#
# 4. SIMULATE DISRUPTION:
#    kubectl delete pod <pod-name>  # This bypasses PDB (direct deletion)
#    kubectl drain node             # This respects PDB
```
**Description:**
At most 1 pod with label `app=myapp` can be unavailable at any time during maintenance.

### PriorityClasses
**Purpose:** Assigns importance to pods so critical workloads are scheduled first.
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class is for critical workloads"
```
**Usage Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
```
**Description:**
Pods with `priorityClassName: high-priority` will be scheduled before lower-priority pods.

---


## ✅ Summary
- **Taints & Tolerations:** Reserve nodes for special workloads.
- **Affinities:** Control where pods run and how they are distributed.
- **Probes:** Ensure your apps are healthy and ready for traffic.
- **Deployment Timing:** Add delays, deadlines, and graceful shutdowns for safe rollouts.
- **PDBs & PriorityClasses:** Keep your workloads highly available and critical apps prioritized.

These controls make Kubernetes deployments more **reliable, predictable, and production-ready**. Copy, customize, and deploy these YAMLs for robust production clusters!
