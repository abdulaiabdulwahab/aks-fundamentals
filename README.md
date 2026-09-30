# aks-fundamentals

# AKS Foundations Project

## Overview

This project is a hands-on introduction to **Azure Kubernetes Service (AKS)**.

The goal is to learn how to create an AKS cluster, connect to it using `kubectl`, deploy a containerized application, expose the application using a Kubernetes Service, scale the workload, perform rolling updates, test self-healing, and troubleshoot common Kubernetes issues.

The application used in this project is a simple **NGINX web server**.

---

## Architecture

```text
Internet
   |
   v
Azure Load Balancer
   |
   v
Kubernetes Service
   |
   v
Deployment
   |
   +-- Pod 1 -> NGINX
   +-- Pod 2 -> NGINX
```

---

## Technologies Used

- Microsoft Azure
- Azure Kubernetes Service (AKS)
- Azure CLI
- Kubernetes
- kubectl
- NGINX
- YAML

---

## Project Objectives

By completing this project, I learned how to:

- Create an AKS cluster
- Connect to AKS using `kubectl`
- Create Kubernetes namespaces
- Deploy applications using Kubernetes Deployments
- Expose applications using a `LoadBalancer` Service
- Scale Kubernetes workloads
- Perform rolling updates
- Roll back failed deployments
- Test Kubernetes self-healing
- View pod logs and events
- Troubleshoot common AKS and Kubernetes issues

---

# Project Structure

```text
01-aks-foundations/
│
├── deployment.yaml
├── service.yaml
└── README.md
```

---

# 1. Create Azure Resources

Set environment variables:

```bash
RG="rg-aks-foundations"
AKS="aks-foundations"
LOCATION="canadacentral"
```

Create the resource group:

```bash
az group create \
  --name "$RG" \
  --location "$LOCATION"
```

Create the AKS cluster:

```bash
az aks create \
  --resource-group "$RG" \
  --name "$AKS" \
  --node-count 2 \
  --generate-ssh-keys
```

---

# 2. Connect to the AKS Cluster

Download the AKS credentials:

```bash
az aks get-credentials \
  --resource-group "$RG" \
  --name "$AKS"
```

Verify connectivity:

```bash
kubectl get nodes
```

Example:

```text
NAME                                STATUS   ROLES
aks-nodepool1-xxxxx-vmss000000      Ready    agent
aks-nodepool1-xxxxx-vmss000001      Ready    agent
```

---

# 3. Create a Namespace

Create a namespace for the application:

```bash
kubectl create namespace foundations
```

Verify:

```bash
kubectl get namespaces
```

---

# 4. Deploy NGINX

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-web
  namespace: foundations

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx-web

  template:
    metadata:
      labels:
        app: nginx-web

    spec:
      containers:
        - name: nginx
          image: nginx:1.28-alpine

          ports:
            - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f deployment.yaml
```

Check the deployment:

```bash
kubectl get deployments -n foundations
```

Check the pods:

```bash
kubectl get pods -n foundations
```

View which nodes are running the pods:

```bash
kubectl get pods -n foundations -o wide
```

---

# 5. Expose the Application

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service
  namespace: foundations

spec:
  type: LoadBalancer

  selector:
    app: nginx-web

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the service:

```bash
kubectl apply -f service.yaml
```

Check the service:

```bash
kubectl get services -n foundations
```

Wait until an external IP appears.

Example:

```text
NAME            TYPE           EXTERNAL-IP
nginx-service   LoadBalancer   4.x.x.x
```

Open the application using:

```text
http://<EXTERNAL-IP>
```

The NGINX welcome page should appear.

---

# 6. Scale the Application

Scale the deployment from two pods to five:

```bash
kubectl scale deployment nginx-web \
  --replicas=5 \
  --namespace foundations
```

Verify:

```bash
kubectl get pods -n foundations
```

---

# 7. Test Kubernetes Self-Healing

List the pods:

```bash
kubectl get pods -n foundations
```

Delete one pod:

```bash
kubectl delete pod <POD-NAME> \
  --namespace foundations
```

Check again:

```bash
kubectl get pods -n foundations
```

Kubernetes automatically creates a replacement pod because the Deployment defines the desired replica count.

---

# 8. Perform a Rolling Update

Update the NGINX image:

```bash
kubectl set image deployment/nginx-web \
  nginx=nginx:1.29-alpine \
  --namespace foundations
```

Monitor the update:

```bash
kubectl rollout status deployment/nginx-web \
  --namespace foundations
```

Check rollout history:

```bash
kubectl rollout history deployment/nginx-web \
  --namespace foundations
```

---

# 9. Roll Back a Deployment

If an update causes problems:

```bash
kubectl rollout undo deployment/nginx-web \
  --namespace foundations
```

Check the rollout:

```bash
kubectl rollout status deployment/nginx-web \
  --namespace foundations
```

---

# Troubleshooting

## Pod is not running

Check pod status:

```bash
kubectl get pods -n foundations
```

Inspect the pod:

```bash
kubectl describe pod <POD-NAME> \
  -n foundations
```

Check logs:

```bash
kubectl logs <POD-NAME> \
  -n foundations
```

Common statuses include:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
```

---

## LoadBalancer IP times out

Check the service:

```bash
kubectl get svc -n foundations
```

Describe the service:

```bash
kubectl describe svc nginx-service \
  -n foundations
```

Check endpoints:

```bash
kubectl get endpointslice \
  -n foundations \
  -l kubernetes.io/service-name=nginx-service
```

Verify the application locally:

```bash
kubectl port-forward \
  deployment/nginx-web \
  8080:80 \
  -n foundations
```

Then open:

```text
http://localhost:8080
```

---

## Service has no endpoints

Check pod labels:

```bash
kubectl get pods \
  -n foundations \
  --show-labels
```

The Deployment label:

```yaml
app: nginx-web
```

must match the Service selector:

```yaml
selector:
  app: nginx-web
```

---

## kubectl cannot connect

Check the current Kubernetes context:

```bash
kubectl config current-context
```

Refresh AKS credentials:

```bash
az aks get-credentials \
  --resource-group "$RG" \
  --name "$AKS" \
  --overwrite-existing
```

---

# Useful Commands

```bash
# View cluster nodes
kubectl get nodes

# View pods
kubectl get pods -n foundations

# View deployments
kubectl get deployments -n foundations

# View services
kubectl get svc -n foundations

# View pod details
kubectl describe pod <POD-NAME> -n foundations

# View logs
kubectl logs <POD-NAME> -n foundations

# View cluster events
kubectl get events -n foundations

# Scale deployment
kubectl scale deployment nginx-web --replicas=5 -n foundations

# Check rollout
kubectl rollout status deployment/nginx-web -n foundations

# Roll back deployment
kubectl rollout undo deployment/nginx-web -n foundations
```

---

# Key Concepts Learned

### Pod

The smallest deployable unit in Kubernetes. A Pod normally contains one or more containers.

### Deployment

Manages application replicas and maintains the desired state of the application.

### Service

Provides stable network access to a group of Pods.

### LoadBalancer

Creates an Azure Load Balancer and exposes the application externally.

### Namespace

Provides logical separation between Kubernetes resources.

### Replica

An individual instance of an application Pod.

### Self-Healing

Kubernetes automatically recreates failed or deleted Pods to maintain the desired state.

### Rolling Update

Updates application Pods gradually instead of stopping the entire application at once.

---

# Cleanup

Delete the Azure resource group when the project is complete:

```bash
az group delete \
  --name "$RG" \
  --yes \
  --no-wait
```

This removes the AKS cluster and associated Azure resources.

---

# Project Summary

This project provided practical experience deploying and managing an application on **Azure Kubernetes Service**.

The main workflow was:

```text
Azure
   |
   v
AKS Cluster
   |
   v
Kubernetes Deployment
   |
   v
NGINX Pods
   |
   v
LoadBalancer Service
   |
   v
Internet
```

Through this project, I gained foundational experience with AKS cluster management, Kubernetes workloads, scaling, application exposure, rolling deployments, self-healing, and troubleshooting.