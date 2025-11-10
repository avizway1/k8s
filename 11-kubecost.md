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

### 3. Create an IAM Service Account with `eksctl`
```bash
eksctl create iamserviceaccount   \
   --name ebs-csi-controller-sa   \
   --namespace kube-system   \
   --cluster ekswithavinash  \
   --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy  \
   --approve \
   --role-only \
   --role-name AmazonEKS_EBS_CSI_DriverRole
```
 **Description:**  
- Creates an IAM service account (`ebs-csi-controller-sa`) in the `kube-system` namespace for the EKS cluster named `ekswithavinash`.
- The service account is attached to the `AmazonEBSCSIDriverPolicy` to provide permissions required by the Amazon EBS CSI driver.
- The `--role-only` option creates the IAM role (`AmazonEKS_EBS_CSI_DriverRole`) without associating it with the service account immediately.
- `--approve` automatically applies the configuration.

---

### 4. Set IAM Role ARN as Environment Variable
```bash
export SERVICE_ACCOUNT_ROLE_ARN=arn:aws:iam::501170964283:role/AmazonEKS_EBS_CSI_DriverRole
```
 **Description:**  
- Stores the ARN of the created IAM role (`AmazonEKS_EBS_CSI_DriverRole`) in the `SERVICE_ACCOUNT_ROLE_ARN` environment variable.
- This variable is used later to link the role with the EBS CSI driver add-on.

---

### 5. Create the EBS CSI Driver Add-on
```bash
eksctl create addon --name aws-ebs-csi-driver --cluster ekswithavinash \
   --service-account-role-arn $SERVICE_ACCOUNT_ROLE_ARN --force
```
 **Description:**  
- Creates the `aws-ebs-csi-driver` add-on in the `ekswithavinash` cluster.
- Associates the add-on with the IAM role stored in the `SERVICE_ACCOUNT_ROLE_ARN` variable.
- `--force` forces the add-on creation even if it already exists or is partially installed.

---

### 6. Authenticate and Login to ECR Public with Helm
```bash
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
```
 **Description:**  
- Retrieves the authentication token for Amazon ECR Public in the `ap-south-1` region.
- Passes the token to `helm registry login` to authenticate and log in to the public ECR registry (`public.ecr.aws`).

---

### 7. Add Kubecost Helm Repository
```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
```
 **Description:**  
- Adds the Kubecost Helm repository to the Helm configuration.
- The URL points to the official Kubecost Helm chart repository.

---

### 8. Update Helm Repositories
```bash
helm repo update
```
 **Description:**  
- Updates the list of Helm charts from all configured repositories, ensuring that the latest versions are available.

---

### 9. Install/Upgrade Kubecost Using Helm
```bash
helm upgrade -i kubecost \
oci://public.ecr.aws/kubecost/cost-analyzer --version 2.6.5 \
--namespace kubecost --create-namespace \
-f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/develop/cost-analyzer/values-eks-cost-monitoring.yaml
```
 **Description:**  
- Installs or upgrades the `kubecost` Helm chart from the public ECR registry (`public.ecr.aws/kubecost/cost-analyzer`), version `2.6.5`.
- Deploys it in the `kubecost` namespace, creating the namespace if it doesn't exist (`--create-namespace`).
- Uses custom values from the provided `values-eks-cost-monitoring.yaml` file for configuration.

---

### 10. Patch gp2 Storage Class as Default
```bash
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```
 **Description:**  
- Modifies the `gp2` storage class to mark it as the default storage class in the cluster.
- Ensures that new Persistent Volume Claims (PVCs) use the `gp2` storage class by default if no other class is specified.

---

### 11. Port Forward Kubecost Service to Access UI
```bash
kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
```
 **Description:**  
- Forwards traffic from port `9090` on the local machine to the `kubecost-cost-analyzer` deployment running in the `kubecost` namespace.
- Allows accessing the Kubecost UI on `http://localhost:9090`.

--- 
