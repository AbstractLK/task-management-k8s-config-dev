Moving from ECS to Amazon EKS (Elastic Kubernetes Service) is a massive leap into advanced container orchestration. It is a fantastic environment for running a microservices stack, but the initial setup involves a few more moving parts than ECS.

A critical note before we begin: **You must revert your Nginx configuration.** In ECS Fargate, your containers were in the same task and shared `localhost`. In Kubernetes, they will run as separate Pods across different nodes. You need to change your Nginx proxy passes back to how you had them in Docker Compose (e.g., `proxy_pass http://auth-service:5001;`). Kubernetes uses its own internal DNS, so the service names will resolve automatically.

Here is your end-to-end guide for deploying to EKS manually via the AWS Console, complete with the Kubernetes manifests.

### **Phase 1: IAM Roles (The Security Foundation)**

EKS requires two specific IAM roles: one for the control plane (the cluster itself) and one for the worker nodes (the EC2 instances running your pods).

**1. Create the EKS Cluster Role**

1. Navigate to the **IAM Dashboard** > **Roles** > **Create role**.
2. Select **AWS service**. From the "Use case" dropdown, select **EKS** and then select **EKS - Cluster**.
3. Click **Next** (the `AmazonEKSClusterPolicy` will be attached automatically).
4. Name it `microservices-eks-cluster-role` and click **Create role**.

**2. Create the Node Group Role**

1. Click **Create role** again.
2. Select **AWS service**. For "Use case", select **EC2**. Click **Next**.
3. Search for and attach these three policies:
    - `AmazonEKSWorkerNodePolicy`
    - `AmazonEC2ContainerRegistryReadOnly`
    - `AmazonEKS_CNI_Policy`
4. Name it `microservices-eks-node-role` and click **Create role**.

### **Phase 2: The Network (Creating the VPC)**

Kubernetes requires a highly specific network topology. The safest way to do this in the console without Terraform is using the VPC Wizard.

1. Navigate to the **VPC Dashboard** and click **Create VPC**.
2. Select **VPC and more**.
3. **Name tag auto-generation:** `eks-microservices`
4. **Number of Availability Zones (AZs):** `2`
5. **Number of public subnets:** `2`
6. **Number of private subnets:** `2` (EKS best practice: Nodes in private subnets, Load Balancers in public).
7. **NAT gateways ($):** `1 per AZ` (Required so your private nodes can pull images from Docker Hub).
8. Click **Create VPC** and wait for it to finish.

### **Phase 3: Provision the EKS Cluster & Compute**

**1. Create the Cluster**

1. Navigate to the **Amazon EKS Dashboard** and click **Add cluster** > **Create**.
2. **Name:** `microservices-cluster`
3. **Kubernetes version:** Select the latest default.
4. **Cluster service role:** Select your `microservices-eks-cluster-role`. Click **Next**.
5. **VPC:** Select your `eks-microservices-vpc`. Ensure all 4 subnets are selected.
6. **Cluster endpoint access:** Select **Public and private**. Click **Next** through the remaining screens and click **Create**.
*Note: Cluster creation takes about 10-15 minutes.*

**2. Add Compute (Node Group)***Wait until your cluster status is `Active` before doing this.*

1. Click on your `microservices-cluster`. Go to the **Compute** tab and click **Add node group**.
2. **Name:** `microservices-nodes`
3. **Node IAM role:** Select your `microservices-eks-node-role`. Click **Next**.
4. **Instance type:** Select `t3.medium`. *(Do not use t2.micro or t3.micro; EKS system pods require more baseline memory, and micro instances will crash).*
5. **Node Group scaling:** Set Minimum, Maximum, and Desired size to `2`. Click **Next**.
6. **Subnets:** Select ONLY the two **Private** subnets. Click **Next** and **Create**.

### **Phase 4: Connecting and Deploying**

Once your nodes are active, open your local terminal. Ensure you have the AWS CLI and `kubectl` installed.

**1. Connect to your Cluster**

Bash

```
aws eks update-kubeconfig --region us-east-1 --name microservices-cluster
```

*(Update `us-east-1` to your actual region).*

**2. Create the Database Secret**
Instead of AWS Parameter Store, we will use a native Kubernetes Secret to store your MongoDB connection string securely.

Bash

```
kubectl create secret generic db-secrets --from-literal=MONGO_URI="mongodb+srv://<db-user>:<db-password>@yourcluster.mongodb.net/database"
```
or

**secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: task-management-secrets
type: Opaque
stringData:
  MONGODB_URI: "mongodb+srv://<db-user>:<db-password>@yourcluster.mongodb.net/database"
  JWT_SECRET: "replace-with-a-long-random-secret"
```

**3. The Kubernetes Manifest (`microservices.yaml`)**
Save the following code to a file named `microservices.yaml`. This file defines your Deployments (the pods running your Docker Hub images) and Services (internal networking and load balancing).

YAML

```
---
# ==========================================
# AUTH SERVICE
# ==========================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        image: abstraxlk/task-management-auth-service:5.1
        ports:
        - containerPort: 4001
        env:
        - name: PORT
          value: "4001"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: task-management-secrets
              key: MONGODB_URI
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: task-management-secrets
              key: JWT_SECRET
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service # This is the internal DNS name Nginx will look for
spec:
  selector:
    app: auth-service
  ports:
    - protocol: TCP
      port: 4001
      targetPort: 4001

---
# ==========================================
# TASK SERVICE
# ==========================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: task-service
  template:
    metadata:
      labels:
        app: task-service
    spec:
      containers:
      - name: task-service
        image: abstraxlk/task-management-task-service:5.1
        ports:
        - containerPort: 4002
        env:
        - name: PORT
          value: "4002"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: task-management-secrets
              key: MONGODB_URI
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: task-management-secrets
              key: JWT_SECRET
---
apiVersion: v1
kind: Service
metadata:
  name: task-service
spec:
  selector:
    app: task-service
  ports:
    - protocol: TCP
      port: 4002
      targetPort: 4002

---
# ==========================================
# FRONTEND & LOAD BALANCER
# ==========================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: abstraxlk/task-management-frontend:3.1
        ports:
        - containerPort: 80 # Or 3000, depending on your container setup
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  # type: LoadBalancer # This tells AWS to provision an external Load Balancer automatically
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80 # Or 3000

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      # Direct routing to Auth Microservice
      # 1. AUTH API (Must come before the frontend catch-all)
      - path: /api/auth/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: auth-service
            port:
              number: 4001

      # Direct routing to Task Microservice
      # 2. TASK API (Must come before the frontend catch-all)
      - path: /api/task/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: task-service
            port:
              number: 4002

      # Static Frontend files routing
      # 3. FRONTEND (The catch-all regex captures file paths like 'assets/main.js' and passes them through intact)
      - path: /?(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**4. Apply the Manifest**
In your terminal, run:

Bash

```
kubectl apply -f secrets/secret.yml
kubectl apply -f microservices.yaml

```

**Install the Ingress Controller**

Run this command in your terminal. It pulls the official installation manifest directly from the Kubernetes community repository and sets up the controller along with a dedicated AWS Load Balancer:  

Bash
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml

```
Note: You will see a long list of resources being created, including namespaces, service accounts, config maps, and the controller deployment itself.

**Get the Public DNS Name of the Ingress Controller**
```
kubectl get service ingress-nginx-controller -n ingress-nginx -w
```

**Verify Your Routing Rules**
```
kubectl get ingress
```

**5. Get Your Live URL**
Because you defined the Frontend Service as `type: LoadBalancer`, AWS will automatically spin up a Classic Load Balancer for you. To get the DNS name (URL) for your application, run:

Bash

```
kubectl get services
```

Look at the `frontend-service` line under the `EXTERNAL-IP` column. It will look something like `a1b2c3d4e5f6...elb.us-east-1.amazonaws.com`. Once that Load Balancer finishes provisioning (usually 2-3 minutes), you can paste that URL into your browser to see your app running on EKS.