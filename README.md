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
│   ├── prometheus.yaml
│   ├── grafana.yaml
│   ├── victoria-metrics-cluster.yaml
│   └── victoria-logs-cluster.yaml
└── charts/               # Vendored Helm charts with embedded values
    ├── minio/
    │   └── values.yaml
    ├── redis/
    │   └── values.yaml
    ├── rook-ceph/
    │   └── values.yaml
    ├── rook-ceph-cluster/
    │   └── values.yaml
    ├── prometheus/
    │   └── values.yaml
    ├── grafana/
    │   └── values.yaml
    ├── victoria-metrics-cluster/
    │   └── values.yaml
    └── victoria-logs-cluster/
        └── values.yaml
```

## 4. Advanced Configuration: Chart Vendoring & Simplified Values
To have full control over the configuration and maintain GitOps best practices, we use **vendored charts with embedded values**.

### Chart Vendoring Process
The complete Helm charts are downloaded and stored in the repository with their default `values.yaml`:

```bash
# Add official repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add redis https://helm.redis.io/
helm repo add minio https://charts.min.io/
helm repo add vm https://victoriametrics.github.io/helm-charts/

# Download and extract charts with values
helm pull prometheus-community/kube-prometheus-stack --destination charts/prometheus --untar
helm pull grafana/grafana --destination charts/grafana --untar
helm pull redis/redis-enterprise-operator --version 7.22.0-16 --destination charts/redis --untar
helm pull minio/minio --destination charts/minio --untar
helm pull vm/victoria-metrics-cluster --destination charts/victoria-metrics-cluster --untar
helm pull vm/victoria-logs-cluster --destination charts/victoria-logs-cluster --untar
```

### Simplified Application Configuration
Applications now reference the vendored charts directly with embedded values:

```yaml
spec:
  sources:
    - repoURL: https://github.com/sumit-ini8labs/Argocd-deployment.git
      targetRevision: main
      path: infrastructure-apps/charts/prometheus
      helm:
        valueFiles:
          - values.yaml
    - repoURL: https://github.com/sumit-ini8labs/Argocd-deployment.git
      targetRevision: main
      ref: values
```

### Benefits of This Approach
- **Single Source of Truth**: Charts and values are versioned together
- **Simplified Structure**: No separate values directory needed
- **Offline Capability**: Charts are available in the repository
- **Easy Customization**: Values can be modified directly in chart directories

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
- **Storage**: Rook-Ceph deployed with distributed storage.
- **Monitoring Stack**: Complete monitoring setup with Prometheus, Grafana, VictoriaMetrics, and VictoriaLogs.
- **Data Services**: Redis Enterprise and MinIO for object storage.
- **GitOps**: All components use vendored charts with embedded values for simplified management.

## 8. Git Troubleshooting & Commands Used

This section documents all git commands used during the monitoring stack setup and troubleshooting process.

### 8.1 Initial Setup Commands

```bash
# Navigate to infrastructure-apps directory
cd /root/infrastructure-apps

# Check git status
git status

# Add all changes
git add .

# Commit changes with descriptive message
git commit -m "Add monitoring stack and update charts to use official repositories

- Add Prometheus, Grafana, VictoriaMetrics cluster, and VictoriaLogs cluster charts
- Update Redis to use official helm.redis.io repository with redis-enterprise-operator
- Update Minio to use official charts.min.io repository
- Remove values folders and update all ArgoCD apps to use chart-internal values.yaml
- Clean up local-path-provisioner application"
```

### 8.2 Push Troubleshooting Commands

```bash
# Initial push attempt (rejected due to remote changes)
git push origin main

# Pull remote changes (encountered divergent branches)
git pull origin main

# Configure pull strategy (rebase)
git pull origin main --rebase

# Check status for merge conflicts
git status

# Resolve merge conflicts by adding our files
git add apps/grafana.yaml apps/prometheus.yaml apps/victoria-logs-cluster.yaml apps/victoria-metrics-cluster.yaml

# Commit the merge resolution
git commit -m "Add monitoring stack and update charts to use official repositories..."

# Second push attempt (still rejected - non-fast-forward)
git push origin main

# Pull again to ensure up to date
git pull origin main

# Check commit history to understand the situation
git log --oneline -5

# Force push with lease (safe force push)
git push --force-with-lease origin main
```

### 8.3 Repository Management Commands

```bash
# Add helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add redis https://helm.redis.io/
helm repo add minio https://charts.min.io/
helm repo add vm https://victoriametrics.github.io/helm-charts/

# Update all repositories
helm repo update

# Download and extract charts
helm pull prometheus-community/kube-prometheus-stack --destination charts/prometheus --untar
helm pull grafana/grafana --destination charts/grafana --untar
helm pull redis/redis-enterprise-operator --version 7.22.0-16 --destination charts/redis --untar
helm pull minio/minio --destination charts/minio --untar
helm pull vm/victoria-metrics-cluster --destination charts/victoria-metrics-cluster --untar
helm pull vm/victoria-logs-cluster --destination charts/victoria-logs-cluster --untar

# Generate values files
helm show values prometheus-community/kube-prometheus-stack > values/prometheus/values.yaml
helm show values grafana/grafana > values/grafana/values.yaml
helm show values vm/victoria-metrics-cluster > values/victoria-metrics-cluster/values.yaml
helm show values vm/victoria-logs-cluster > values/victoria-logs-cluster/values.yaml
```

### 8.4 Directory Structure Commands

```bash
# Create directory structure
mkdir -p charts/prometheus charts/grafana charts/redis charts/minio
mkdir -p charts/victoria-metrics-cluster charts/victoria-logs-cluster
mkdir -p values/prometheus values/grafana values/redis values/minio
mkdir -p values/victoria-metrics-cluster values/victoria-logs-cluster

# Remove old values folder structure
rm -rf values/

# Remove old charts
rm -rf charts/redis charts/minio
```

### 8.5 Git Workflow Best Practices Used

1. **Always check status before operations**: `git status`
2. **Use descriptive commit messages**: Include what changed and why
3. **Handle merge conflicts carefully**: Add specific files that had conflicts
4. **Use safe force push**: `--force-with-lease` instead of `--force`
5. **Verify commit history**: `git log --oneline` to understand the state
6. **Pull before pushing**: Always integrate remote changes first

### 8.6 Common Issues & Solutions

**Issue**: Push rejected due to non-fast-forward
```bash
# Solution: Pull remote changes first
git pull origin main
# Then push
git push origin main
```

**Issue**: Divergent branches error
```bash
# Solution: Use rebase to maintain linear history
git pull origin main --rebase
```

**Issue**: Merge conflicts
```bash
# Solution: Check status, add conflicted files, commit
git status
git add <conflicted-files>
git commit -m "Resolve merge conflicts"
```

**Issue**: Still rejected after merge
```bash
# Solution: Force push with lease (safer than regular force)
git push --force-with-lease origin main
```

## 9. Key ArgoCD Features Used
*   **Git as Source of Truth**: Every configuration is defined in Git, ensuring disaster recovery readiness.
*   **Self-Healing**: ArgoCD automatically reverts manual cluster changes to match the Git state.
*   **Multiple Sources**: Combines official Helm charts with private Git-based configuration values.
*   **App of Apps**: A single parent application manages the entire infrastructure lifecycle.
*   **Visual Health Monitoring**: Real-time insights into resource status via the ArgoCD Web UI.
*   **Sync Waves**: Orchestrates deployment order (e.g., ensuring Storage is ready before Apps).
