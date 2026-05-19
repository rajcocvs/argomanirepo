# AKS + Argo CD + GitOps Deployment Guide

## Overview

This document covers a complete end-to-end implementation of:

- AKS Cluster Setup
- Argo CD Installation and Configuration
- Application Deployment using Argo CD (GitOps)

This implementation demonstrates a real-time enterprise Kubernetes and GitOps deployment workflow using:

- Azure Kubernetes Service (AKS)
- Azure Container Registry (ACR)
- Argo CD
- GitHub
- Kubernetes YAML manifests

---

# Architecture

```text
Developer
    ↓
GitHub Repository
    ↓
Argo CD
    ↓
AKS Cluster
    ↓
Kubernetes Deployment + Service
```

---

# PART 1 — AKS Cluster Setup

## Step 1 — Login to Azure

```bash
az login
```

Verify subscription:

```bash
az account show
```

---

## Step 2 — Create Resource Group

```bash
az group create \
--name devops-aks-rg \
--location centralindia
```

Verify:

```bash
az group list --output table
```

---

## Step 3 — Create Azure Container Registry (ACR)

```bash
az acr create \
--resource-group devops-aks-rg \
--name devopsacrraj \
--sku Basic
```

Verify:

```bash
az acr list --output table
```

---

## Step 4 — Create AKS Cluster

Due to quota limitations, use a small VM size.

```bash
az aks create \
--resource-group devops-aks-rg \
--name devops-aks \
--node-count 1 \
--node-vm-size Standard_B2s \
--enable-addons monitoring \
--generate-ssh-keys \
--attach-acr devopsacrraj
```

### What Happens Internally During AKS Creation?

AKS automatically creates:

- Kubernetes control plane
- VM Scale Set
- Virtual Network
- Load Balancer
- Managed Identity
- Managed Disks
- Node Pool

---

## Step 5 — Connect to AKS Cluster

```bash
az aks get-credentials \
--resource-group devops-aks-rg \
--name devops-aks
```

---

## Step 6 — Verify AKS Cluster

```bash
kubectl get nodes
```

Expected:

```text
STATUS = Ready
```

---

## Step 7 — Verify Kubernetes Context

Current context:

```bash
kubectl config current-context
```

All contexts:

```bash
kubectl config get-contexts
```

---

# Important AKS Concepts

## AKS Architecture

### Control Plane (Managed by Azure)

- API Server
- Scheduler
- Controller Manager
- ETCD

### Worker Nodes

- kubelet
- kube-proxy
- Container Runtime
- Pods

---

## Kubernetes Communication Flow

```text
kubectl
   ↓
API Server
   ↓
Scheduler
   ↓
Worker Node
   ↓
Container Runtime
```

---

# PART 2 — Install and Configure Argo CD

## What is Argo CD?

Argo CD is a GitOps continuous delivery tool for Kubernetes.

Argo CD continuously compares:

```text
Git Repository State
VS
Kubernetes Cluster State
```

If differences exist:

- Argo CD synchronizes changes
- Restores deleted resources
- Detects configuration drift
- Performs automatic deployments

---

## Step 1 — Create Argo CD Namespace

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get ns
```

---

## Step 2 — Install Argo CD

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs:

- argocd-server
- argocd-repo-server
- argocd-application-controller
- argocd-dex-server
- argocd-redis

---

## Step 3 — Verify Argo CD Pods

```bash
kubectl get pods -n argocd
```

Wait until all pods become:

```text
Running
```

---

## Step 4 — Verify Services

```bash
kubectl get svc -n argocd
```

Initially the service type will be:

```text
ClusterIP
```

---

## Step 5 — Expose Argo CD UI

Convert Argo CD service into LoadBalancer.

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec": {"type": "LoadBalancer"}}'
```

---

## Step 6 — Get External IP

```bash
kubectl get svc -n argocd
```

Wait until:

```text
EXTERNAL-IP = 20.x.x.x
```

---

## Step 7 — Open Argo CD UI

Open browser:

```text
https://EXTERNAL-IP
```

Important:

- Use HTTPS
- Accept self-signed certificate warning

---

## Step 8 — Get Argo CD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

Example output:

```text
abcd1234
```

That output itself is the password.

---

## Step 9 — Login to Argo CD

| Field | Value |
|---|---|
| Username | admin |
| Password | output from previous command |

---

# Important Argo CD Concepts

## GitOps Workflow

```text
Git Repository
      ↓
Argo CD
      ↓
AKS Cluster
```

---

## Desired State Management

Argo CD continuously compares:

```text
Desired State (Git)
VS
Actual State (Cluster)
```

---

## Self-Healing

If Kubernetes resources are deleted manually:

```text
Argo CD recreates them automatically
```

---

# PART 3 — Create Application in Argo CD

## Prerequisites

Before this step:

- Kubernetes YAML files should already exist
- YAML files should already be pushed to GitHub repository

---

## Example Repository Structure

```text
argomanirepo/
 ├── mydeploy.yml
 └── myservice.yml
```

---

## Example Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: ncpl-deployment
  namespace: java-app

spec:
  replicas: 3

  selector:
    matchLabels:
      app: ncpl

  template:
    metadata:
      labels:
        app: ncpl

    spec:
      containers:
      - name: ncpl

        image: devopsacrraj.azurecr.io/java-app:v1

        ports:
        - containerPort: 8080
```

---

## Example Service YAML

```yaml
apiVersion: v1
kind: Service

metadata:
  name: ncpl-service
  namespace: java-app

spec:
  selector:
    app: ncpl

  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080

  type: LoadBalancer
```

---

## Step 1 — Create Namespace for Application

```bash
kubectl create namespace java-app
```

Verify:

```bash
kubectl get ns
```

---

## Step 2 — Open Argo CD UI

Open:

```text
https://EXTERNAL-IP
```

Login using:

| Field | Value |
|---|---|
| Username | admin |
| Password | Argo CD password |

---

## Step 3 — Create New Application

Click:

```text
NEW APP
```

---

## Step 4 — Fill Application Details

### General Section

| Field | Value |
|---|---|
| Application Name | java-app |
| Project | default |

Important:

Use:

```text
default
```

Do NOT create custom project unless required.

---

## Sync Policy

Select:

- Automatic

Enable:

- Auto Sync
- Self Heal
- Prune Resources

---

## Source Section

| Field | Value |
|---|---|
| Repository URL | Your GitHub Repo URL |
| Revision | HEAD |
| Path | . |

Because YAML files are present in root repository.

---

## Destination Section

| Field | Value |
|---|---|
| Cluster URL | https://kubernetes.default.svc |
| Namespace | java-app |

---

## Step 5 — Create Application

Click:

```text
CREATE
```

Argo CD now:

- pulls YAML files from GitHub
- compares desired state
- deploys resources into AKS

---

## Step 6 — Verify Application

Application status should become:

```text
Healthy
Synced
```

---

## Step 7 — Verify Deployment in AKS

```bash
kubectl get all -n java-app
```

You should see:

- Pods
- Deployment
- ReplicaSet
- Service

---

## Step 8 — Get External IP

```bash
kubectl get svc -n java-app
```

Wait until:

```text
EXTERNAL-IP
```

appears.

---

## Step 9 — Open Application

Open:

```text
http://EXTERNAL-IP
```

Application should load successfully.

---

# Troubleshooting — External IP Pending

If EXTERNAL-IP remains pending:

Run:

```bash
kubectl describe svc ncpl-service
```

Common issue:

```text
PublicIPCountLimitReached
```

Meaning:

```text
Azure subscription reached public IP quota.
```

---

# Fix Public IP Limit Issue

## Option 1 — Delete Unused Public IPs

Go to:

```text
Azure Portal
→ Load Balancer
→ Frontend IP Configuration
```

Delete unused Public IP.

---

## Option 2 — Recreate Service

```bash
kubectl delete svc ncpl-service
```

Argo CD automatically recreates it.

This demonstrates GitOps self-healing.

---

# Important Enterprise Concepts

## Kubernetes Service Flow

```text
Kubernetes Service
       ↓
Cloud Controller Manager
       ↓
Azure API
       ↓
Azure Load Balancer
       ↓
Public IP
       ↓
Traffic reaches Pods
```

---

# Final Architecture

```text
Developer
    ↓
GitHub Repository
    ↓
Argo CD
    ↓
AKS Cluster
    ↓
Deployment + Service
    ↓
Application Access
```

---

# Final Outcome

Successfully implemented:

- AKS Cluster
- ACR
- Argo CD
- GitOps Workflow
- Kubernetes Deployment
- Kubernetes Service
- Automatic Synchronization
- Self-Healing Deployment
