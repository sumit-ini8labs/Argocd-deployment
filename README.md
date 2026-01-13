# ArgoCD & Infrastructure Deployment Guide

This document summarizes the installation of ArgoCD and the deployment of infrastructure components (MinIO, Redis, Rook-Ceph, Local Path Provisioner) using GitOps.

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

## 3. GitOps Repository Structure
The repository at `https://github.com/sumit-ini8labs/Argocd-deployment` is structured to manage applications and their configurations separately.

```text
.
├── app-of-apps.yaml      # Parent application
├── apps/                 # Application manifests
│   ├── minio.yaml
│   ├── redis.yaml
│   ├── rook-ceph.yaml
│   └── local-path-provisioner.yaml
└── values/               # Vendored Helm values
    ├── minio.yaml
    ├── redis.yaml
    ├── rook-ceph-operator.yaml
    ├── rook-ceph-cluster.yaml
    └── local-path-provisioner.yaml
```

## 4. Advanced Configuration: Vendoring & Multiple Sources
To have full control over the configuration and maintain GitOps best practices, we use **Multiple Sources**.

### Vendoring Values
The full default `values.yaml` were fetched from Helm repos:
```bash
helm repo add minio https://charts.min.io/
helm show values minio/minio --version 5.0.14 > values/minio.yaml
```

### Multiple Sources in Application
Applications reference both the remote chart and the local values file:
```yaml
spec:
  sources:
    - repoURL: https://charts.min.io/
      chart: minio
      targetRevision: 5.0.14
      helm:
        valueFiles:
          - $values/values/minio.yaml
    - repoURL: https://github.com/sumit-ini8labs/Argocd-deployment.git
      targetRevision: main
      ref: values
```

## 5. Git Command History
Commands used to synchronize the local manifests with the GitHub repository:

```bash
# Identity & Credential Helper
git config --global user.name "sumit-ini8labs"
git config --global user.email "sumit@ini8labs.tech"
git config --global credential.helper store

# Sync Process
git add .
git commit -m "Vendor full values and update to Multiple Sources"
git push origin main
```

## 6. Deployment via ArgoCD
The "App of Apps" pattern manages all components.

```bash
# Apply Parent App
kubectl apply -f app-of-apps.yaml -n argocd

# Check Status
kubectl get applications -n argocd
```

## 7. Current Status & Observations
- **ArgoCD**: Fully functional.
- **Storage**: Added `local-path-provisioner` as a fallback for Rook-Ceph.
- **GitOps**: All components are synced using vendored `values/` in Git.
