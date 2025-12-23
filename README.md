# EKS Sample Project – AWS Load Balancer Controller (ALB Ingress)

This project demonstrates how to integrate an Amazon EKS cluster with the **AWS Load Balancer Controller** using a simple sample application.  
An **Application Load Balancer (ALB)** is automatically provisioned via Kubernetes **Ingress**, without using NGINX, databases, or persistent storage.

The setup is intentionally minimal, secure, and aligned with real-world production best practices.

---

## 1. Objective

The goal of this project is to:

- Deploy a sample application on Amazon EKS
- Expose the application using AWS Load Balancer Controller
- Automatically provision an Application Load Balancer (ALB)
- Route external traffic using Kubernetes Ingress
- Avoid using NGINX Ingress Controller

---

## 2. Prerequisites

The following components must already be available:

- Amazon EKS cluster (`eks-ingress`)
- Worker nodes in **Ready** state
- Tools installed:
  - `kubectl`
  - `aws`
  - `eksctl`
  - `helm`
- AWS Load Balancer Controller installed in the `kube-system` namespace
- OIDC provider associated with the EKS cluster
- IRSA (IAM Role for ServiceAccount) configured for the controller

---

## 3. Project Structure
```
simple-eks-demo/
├── namespace.yml
├── app1-deployment.yml
├── app1-service.yml
├── alb-ingress.yml
```

Each file has a specific responsibility and is explained below.

---

## 4. Namespace Creation

**File:** `namespace.yml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```
**Explanation**
- Kubernetes namespaces provide logical isolation

- A dedicated namespace demo is created for this project

- This follows real-world best practices

**Command**
```kubectl apply -f namespace.yml
```

## 5. Application Deployment

File: app1-deployment.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx
        ports:
        - containerPort: 80
```

**Explanation**
- Deploys a simple nginx container

- Runs a single replica for demonstration

- Exposes port 80 inside the pod

- Labels allow the Service to target this pod

## 6. Application Service
 File: app1-service.yml
 ```apiVersion: v1
kind: Service
metadata:
  name: app1
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
```
**Explanation**
- Creates a ClusterIP service (internal-only)

- Exposes the nginx pod on port 80

- Acts as the backend for the Ingress

- No NodePort or LoadBalancer service is required

## 7. Why ClusterIP and NOT NodePort?

In this setup, **ClusterIP** is the correct and recommended service type because traffic is handled by the **AWS Load Balancer Controller (ALB Ingress)**, not directly by Kubernetes nodes.

### Service Type Comparison

| Service Type   | Behavior                                               | Recommended |
|---------------|--------------------------------------------------------|-------------|
| **ClusterIP** | Internal-only service, accessed via Ingress/ALB        | ✅ Yes      |
| **NodePort**  | Exposes service on each node’s IP and a static port    | ❌ No       |
| **LoadBalancer** | Creates a separate AWS ELB for each service        | ❌ No       |

### Why ClusterIP?

- The **ALB Ingress** routes external traffic to services **inside the cluster**
- Services only need to be reachable **internally**
- Keeps the architecture **secure and clean**
- Avoids unnecessary public exposure of nodes

### Why NOT NodePort?

- Exposes applications directly on **node IPs and ports**
- Bypasses ALB and Ingress routing logic
- Increases **security risk**
- Not aligned with AWS ALB Ingress best practices

### Why NOT LoadBalancer?

- Creates **one AWS ELB per service**
- Higher **cost**
- Breaks the **single ALB + multiple Ingress rules** pattern

### Final Decision

**ClusterIP** is used because:
- ALB handles all external traffic
- Kubernetes services remain internal
- Architecture is production-aligned and cost-efficient

**Explanation**

- The ALB (created via Ingress) handles external traffic

- The Service only provides an internal backend

- NodePort increases attack surface

- LoadBalancer is redundant and costly

## 8. AWS ALB Ingress Configuration
File: alb-ingress.yml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
```
**Explanation**

- ingress.class: alb tells Kubernetes to use AWS Load Balancer Controller

- Creates an internet-facing ALB

- Uses IP mode to route traffic directly to pod IPs

- Listens on HTTP port 80

- Routes / traffic to the app1 service

- No NGINX Ingress Controller is involved

## 9. OIDC + IRSA (IAM Role for ServiceAccount)

### Key Concepts

- **ServiceAccount**  
  Represents the identity of a Pod within Kubernetes.

- **IRSA (IAM Roles for Service Accounts)**  
  Allows Kubernetes Pods to securely assume AWS IAM roles without using node-level credentials.

- **OIDC Provider**  
  Acts as the trust bridge between Amazon EKS and AWS IAM, enabling Pods to authenticate using ServiceAccounts.

---

### Why This Is Required

The **AWS Load Balancer Controller** requires AWS permissions to:

- Create and manage **Application Load Balancers (ALBs)**
- Create and manage **Target Groups**
- Manage **Security Groups** and related networking resources

Using **OIDC + IRSA** ensures:

- ✅ Permissions are granted **only to the controller Pod**
- ✅ **Worker nodes are not over-privileged**
- ✅ No hardcoded AWS credentials inside Pods
- ✅ Secure, short-lived credentials via IAM

---

### Security Responsibility Split

| Layer | Responsibility |
|------|----------------|
| **RBAC** | Controls access to the Kubernetes API |
| **IRSA** | Controls access to AWS APIs |

---

### Benefits

- Follows **least-privilege principle**
- Improves **security posture**
- Required for **production-grade EKS setups**
- Recommended by AWS for all controllers and workloads needing AWS access

---

## 10. Deployment Command
```
kubectl apply -f . -n demo
```
**This command:**

- Creates the namespace

- Deploys the application

- Creates the service

- Creates the ingress

  
## 11. Verification Steps

**Check Pods**
```
kubectl get pods -n demo
```
**Check Service**
```
kubectl get svc -n demo
```
**Check Ingress**
```
kubectl get ingress -n demo
```
## 12. Browser Access

**Open the ALB DNS name in a browser:**
```
http://<ALB-DNS-NAME>/
```
> **Summary:**  
> OIDC + IRSA allows the AWS Load Balancer Controller to securely interact with AWS resources while keeping both Kubernetes and AWS permissions tightly scoped and production-ready.

## conclusion 
A sample application was deployed on Amazon EKS and exposed using AWS Load Balancer Controller.
An Application Load Balancer was automatically provisioned through Kubernetes Ingress and routes external traffic to internal ClusterIP services.
IRSA ensures secure AWS access for the controller, while RBAC enforces Kubernetes API permissions.
