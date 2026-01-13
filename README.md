# ArgoCD & Infrastructure Deployment Guide

This document summarizes the installation of ArgoCD and the deployment of infrastructure components (MinIO, Redis, Rook-Ceph) using GitOps.

## 1. ArgoCD Installation & Configuration
ArgoCD was installed on the Kubernetes cluster and exposed via NodePort.

```bash
# Create Namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose ArgoCD Server via NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get Admin Password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## 2. GitHub Repository Authentication
To allow ArgoCD to read from the private repository, a secret was created and labeled.

```bash
# Create GitHub Credentials Secret
kubectl create secret generic github-repo-creds \
  --from-literal=type=git \
  --from-literal=url=https://github.com/sumit-ini8labs/Argocd-deployment \
  --from-literal=password=<YOUR_PAT_TOKEN> \
  --from-literal=username=sumit-ini8labs \
  -n argocd

# Label the secret so ArgoCD recognizes it as a Repository
kubectl label secret github-repo-creds -n argocd argocd.argoproj.io/secret-type=repository
```

## 3. Manifest Preparation
A local Git repository was created at `/root/infrastructure-apps` with the following structure:
- `apps/minio.yaml`: MinIO Object Storage (Helm-based)
- `apps/redis.yaml`: Redis Cache (Helm-based)
- `apps/rook-ceph.yaml`: Rook-Ceph Storage Cluster
- `app-of-apps.yaml`: Parent application managing all child apps.

## 4. Git Command History
These are the commands used to synchronize the local manifests with the GitHub repository:

```bash
# Identity Setup
git config --global user.name "sumit-ini8labs"
git config --global user.email "sumit@ini8labs.tech"
git config --global credential.helper store

# Local Repo Initialization
git init
git add .
git commit -m "Initialize infrastructure manifests"

# Synchronization with GitHub
git remote add origin https://github.com/sumit-ini8labs/Argocd-deployment.git
git pull origin main --rebase
git push -u origin main
```

## 5. Deployment via ArgoCD
The "App of Apps" pattern was used to deploy all components at once.

```bash
# Apply the Parent Application
kubectl apply -f app-of-apps.yaml -n argocd

# Monitor Applications
kubectl get applications -n argocd
```


## 5. Current Status & Observations
- **ArgoCD**: Fully functional.
- **MinIO/Redis**: Deployed but `Pending` status. They require a functional StorageClass.
- **Rook-Ceph**: Deployed but encountering kernel module issues (`rbd: Key was rejected by service`).
- **Next Steps**: Consider using a `local-path-provisioner` for storage if Rook-Ceph is not compatible with the current node's kernel.
