
### **Why is Ingress Widely Used? (Advantages of Ingress)**  
Ingress is widely used in Kubernetes because it provides a centralized way to manage external access to services. Instead of creating multiple LoadBalancers or NodePort services, Ingress offers a scalable and efficient way to expose applications.

#### **Key Advantages of Ingress:**  
 **Single Entry Point** – Manages access to multiple services from a single external URL.  
 **Path-Based Routing** – Allows different applications to be hosted under different paths (`/app1`, `/app2`).  
 **Host-Based Routing** – Supports different domains/subdomains (`app1.example.com`, `app2.example.com`).  
 **TLS/SSL Termination** – Secures applications with HTTPS, offloading SSL/TLS termination at the Ingress.  
 **Load Balancing** – Distributes incoming traffic across multiple backend pods.  
 **Reduces Costs** – Eliminates the need to create separate AWS ELBs for each service.  

---
![flow-new](https://github.com/user-attachments/assets/88a31984-6f95-4667-bde2-40e09e0b9a5c)

---

### **Components of Ingress in Kubernetes**  

An **Ingress setup** consists of several components:

####  **1. Ingress Resource** (Kubernetes Object)  
- Defines routing rules, specifying how external traffic reaches services.  
- Example: Path-based, host-based routing, SSL termination.  

####  **2. Ingress Controller**  
- The actual implementation that processes Ingress rules and provisions the underlying AWS ALB or NLB.  
- Example: **AWS Load Balancer Controller**, NGINX Ingress Controller, Traefik.  

####  **3. Load Balancer**  
- The external AWS **Application Load Balancer (ALB)** or **Network Load Balancer (NLB)** that routes traffic.  
- The ALB Controller automatically creates and manages this.  

####  **4. Target Groups**  
- AWS resources that ALB forwards traffic to.  
- In **Instance Mode**, the target is EC2 instances.  
- In **IP Mode**, the target is pod IPs.  

---
### ** What are the 2 Ingress Controller Modes in AWS EKS?**  
In AWS EKS, the AWS Load Balancer Controller supports two modes for managing traffic:  

1. **Instance Mode**  
2. **IP Mode**  

---

Underdtand Path Based vs IP Based Routing

![ip-path-png](https://github.com/user-attachments/assets/a0f61d0b-0b8b-40c9-963f-8ff477bcef58)

---
![Modes](https://github.com/user-attachments/assets/26853ca3-f76d-4eaf-b52e-8b75a44c8968)

---

### ** Instance Mode vs. IP Mode in AWS EKS Ingress Controller**  

| Feature           | **Instance Mode** | **IP Mode** |
|------------------|-----------------|----------------|
| **Target Type**  | Routes traffic to EC2 instances | Routes traffic directly to pod IPs |
| **Networking**   | Uses AWS VPC routing, requires Worker Nodes to be in a Public Subnet | Uses AWS VPC CNI, direct pod communication |
| **Target Group** | Uses **Instance Target Group** | Uses **IP Target Group** |
| **Traffic Flow** | ALB → NodePort → Pods | ALB → Pods (bypassing NodePort) |
| **Use Case**     | When using EC2 worker nodes in public subnets | For efficient, low-latency networking with EKS Fargate or private nodes |
| **Security Group** | Worker nodes need to allow inbound traffic from ALB | Pods must allow inbound traffic from ALB |

 **When to use Instance Mode?**  
- If your cluster runs EC2 instances and you need a simpler setup.  
- When using traditional Kubernetes `NodePort` services.  

 **When to use IP Mode?**  
- If you are running on **EKS Fargate** or using private subnets.  
- If you want **better performance and lower latency** (direct traffic to pods).  
- If you want to avoid additional hops through `NodePort`.  

---


# Deploying the 2048 Game on EKS Using AWS Load Balancer Controller

This guide details the steps required to deploy a 2048 game application on an Amazon EKS cluster. You will use the AWS Load Balancer Controller (ALB) to expose your application via an Application Load Balancer (ALB).

---

## Prerequisites

Before you begin, ensure you have the following installed and configured:
- **eksctl**
- **kubectl**
- **AWS CLI tools**

Also, you must have already created an EKS cluster.

---

## Step 1: Connect kubectl to Your EKS Cluster

Create or update your kubeconfig file so that `kubectl` can connect to your EKS cluster.

```bash
aws eks update-kubeconfig --region ap-south-1 --name ekswithavinash
```

This command fetches the cluster details and updates your local kubeconfig.

---

## Step 2: Create an IAM OIDC Provider for Your Cluster

Set your cluster name in an environment variable:

```bash
cluster_name=ekswithavinash
```

Extract the OIDC ID from your cluster:

```bash
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Check if an IAM OIDC provider already exists for your cluster:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

- **If output is returned:** You already have an IAM OIDC provider and can skip the next step.
- **If no output is returned:** Create an IAM OIDC provider for your cluster:

```bash
eksctl utils associate-iam-oidc-provider --cluster ekswithavinash --approve
```

---

## Step 3: Install AWS Load Balancer Controller with Helm

### 3.1 – Download and Create the IAM Policy

Download the IAM policy required for the AWS Load Balancer Controller:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 3.2 – Create the IAM Service Account

Create the IAM service account for the AWS Load Balancer Controller using `eksctl`. This command attaches the policy to the service account and (if it exists) overrides the existing service account:

```bash
eksctl create iamserviceaccount \
    --cluster=ekswithavinash \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::501170964283:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-south-1 \
    --approve
```

### 3.3 – Install the AWS Load Balancer Controller

Add the Helm repository and update it:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Install the AWS Load Balancer Controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ekswithavinash \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 3.4 – Verify the Controller Deployment

Confirm that the controller is installed and running:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Step 4: Deploy the 2048 Game Application

Below are the four YAML files you provided. These files create a dedicated namespace, deploy the 2048 game, create a service, and set up an ingress to expose the application via the ALB.

### 4.1 – Namespace (namespace.yaml)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: 2048-game
```

Apply the namespace:

```bash
kubectl apply -f namespace.yaml
```

### 4.2 – Deployment (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-2048-deployment
  namespace: 2048-game
  labels:
    app: game-2048
spec:
  replicas: 2
  selector:
    matchLabels:
      app: game-2048
  template:
    metadata:
      labels:
        app: game-2048
    spec:
      containers:
      - name: game-2048
        image: thipparthiavinash/2048-game
        ports:
        - containerPort: 80
```

Deploy the application:

```bash
kubectl apply -f deployment.yaml
```

### 4.3 – Service (service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-2048-service
  namespace: 2048-game
  labels:
    app: game-2048
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: game-2048
```

Create the service:

```bash
kubectl apply -f service.yaml
```

## If you want to deploy using "NodePort" then use below command.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-2048-service
  namespace: 2048-game
  labels:
    app: game-2048
spec:
  type: NodePort
  selector:
    app: game-2048
  ports:
    - protocol: TCP
      port: 80            # Service Port
      targetPort: 80      # Container Port
      nodePort: 30080     # NodePort (Optional: You can specify a custom port or let Kubernetes assign one)
```


### 4.4 – Ingress (ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-2048-ingress
  namespace: 2048-game
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
  - http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: game-2048-service
            port:
              number: 80
```

Apply the ingress:

```bash
kubectl apply -f ingress.yaml
```

---

## Step 5: Verify the Deployment

Check the status of the ingress to ensure that an ALB is provisioned:

```bash
kubectl describe ingress game-2048-ingress -n 2048-game
```

When the ALB is ready, you should see its DNS name in the **Address** field. You can then access the 2048 game in your browser using that DNS name.

---
# If you have a Route53 hostedZone and an ACM Certificate to map with alb DNS name, Please deploy below ingress.yml file
---

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-2048-ingress
  namespace: 2048-game
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:1234567890:certificate/Your-Cert-ARN
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
spec:
  rules:
  - host: game.learnaws.today
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: game-2048-service
            port:
              number: 80


```

Then map domain name "game.learnaws.today" at route53 and test the output.
