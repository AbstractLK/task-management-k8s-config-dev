# **Task Management - Kubernetes GitOps Configuration**

This repository contains the declarative Kubernetes infrastructure for the Task Management microservices application. The deployment is fully automated using an **ArgoCD "App of Apps"** pattern and utilizes the **External Secrets Operator (ESO)** to securely fetch parameters from AWS Systems Manager (SSM) Parameter Store.

## **Architecture Highlights**

- **GitOps Engine:** ArgoCD
- **Secrets Management:** External Secrets Operator (ESO)
- **Ingress Controller:** NGINX Ingress Controller (AWS Network Load Balancer)
- **Deployment Pattern:** App of Apps with automated Sync Waves

## **Prerequisites**

- An active AWS EKS Cluster (provisioned via Terraform or `eksctl`).
- `aws-cli`, `kubectl`, and `helm` installed locally.
- AWS credentials configured with administrative access to the EKS cluster.
- **Provision the AWS Infrastructure:**

Run from your Terraform directory.

First, build your EKS cluster and VPC.

```
terraform init
terraform apply
```

*Type `yes` when prompted to confirm the infrastructure creation. This will take roughly 10-15 minutes as AWS provisions the cluster.*

---

## **Deployment Instructions**

### **1. Connect to the EKS Cluster**

Once your cluster is provisioned, configure your local `kubectl` to communicate with it.

bash

```
aws eks update-kubeconfig --region <your-aws-region> --name <your-eks-cluster-name>
```

Verify the connection by running:

bash

```
kubectl get nodes
```

### **2. Bootstrap ArgoCD (GitOps Engine)**

Install ArgoCD directly into the cluster using the official Helm chart.

bash

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD into its own namespace
helm install argocd argo/argo-cd --namespace argocd --create-namespace

# Wait for all ArgoCD components to become healthy
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### **3. Deploy the Application (The Handoff)**

Apply the master application file. From this point forward, ArgoCD takes over and manages the cluster state based on the contents of this repository.

bash

```
kubectl apply -f root-application.yaml
```

### **4. Monitor the Deployment**

ArgoCD orchestrates the deployment in specific **Sync Waves** to ensure dependencies are met before pods start:

- **Wave 0:** NGINX Ingress Controller is installed (requests an AWS Load Balancer).
- **Wave 1:** AWS IAM ServiceAccount is registered.
- **Wave 2:** SecretStore connects to AWS.
- **Wave 3:** ExternalSecret fetches data from AWS and generates the native Kubernetes Secret.
- **Wave 4:** Microservices (Auth, Task, Frontend) and Ingress routing rules are deployed.

To watch the deployment visually, port-forward the ArgoCD dashboard:

bash

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access the dashboard at `https://localhost:8080`.

- **Username:** `admin`
- **Password:** Run the following command to retrieve it:

bash

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## **Safe Teardown & Deletion**

> ⚠️ **CRITICAL:** Do not use `helm uninstall` or `terraform destroy` immediately. If you delete ArgoCD or the EKS cluster before gracefully deleting your applications, you will leave orphaned AWS Load Balancers running, which will incur unexpected AWS charges.
> 

Follow these steps in order for a safe and clean teardown.

### **Step 1: Cascade Delete the GitOps Applications**

Trigger the finalizers to safely remove all application resources, including the NGINX Load Balancer and ESO resources.

bash

```
kubectl delete -f root-application.yaml
```

Wait **2–3 minutes** for this command to complete. ArgoCD actively deletes the NGINX Ingress, which triggers AWS to destroy the physical Load Balancer.

### **Step 2: Uninstall ArgoCD via Helm**

Once the applications and load balancers are confirmed deleted, safely remove the ArgoCD engine.

bash

```
helm uninstall argocd -n argocd
```

### **Step 3: Clean up Namespaces and CRDs**

Remove any lingering Custom Resource Definitions (CRDs) and namespaces.

bash

```
kubectl delete namespace argocd
kubectl delete namespace ingress-nginx
kubectl delete namespace external-secrets
kubectl delete namespace monitoring
```

### **Step 4: Destroy Infrastructure (Optional)**

If you are completely tearing down the environment, you can now safely run your Terraform destroy commands.

bash

```
terraform destroy
```