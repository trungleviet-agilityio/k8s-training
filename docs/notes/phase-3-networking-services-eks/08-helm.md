# Helm - Kubernetes Package Manager

## Overview

**Helm** is the package manager for Kubernetes, similar to `apt` or `yum` for Linux. It simplifies deployment and management of Kubernetes applications by packaging them into reusable units called **charts**.

### Why Use Helm?

- **Simplified Deployment:** Install complex applications with a single command
- **Version Management:** Track and manage different versions of deployments
- **Configuration Management:** Customize deployments using values files
- **Rollback Capability:** Easily revert to previous versions
- **Reusability:** Share and reuse charts across teams

We used Helm to install the **AWS Load Balancer Controller** in [Issue #23](./07-alb-controller.md).

---

## Core Concepts

- **Charts:** Pre-configured Kubernetes resource packages (Deployments, Services, ConfigMaps, etc.)
- **Releases:** Instances of charts deployed to a cluster (each has a unique name)
- **Repositories:** Collections of charts (like package repositories)
- **Values:** Configuration parameters that customize charts (via `values.yaml` or `--set` flags)

**Example:**
```bash
helm install my-nginx bitnami/nginx  # Creates release "my-nginx" from bitnami/nginx chart
```

---

## Installation

### Install Helm 3 on Linux

**Method 1: Using Installation Script (Requires sudo)**
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
sudo ./get_helm.sh
```

**Method 2: Install to User Directory (No sudo)**
```bash
mkdir -p ~/.local/bin
cd /tmp
curl -fsSL https://get.helm.sh/helm-v3.19.2-linux-amd64.tar.gz -o helm.tar.gz
tar -zxvf helm.tar.gz
mv linux-amd64/helm ~/.local/bin/helm
rm -rf linux-amd64 helm.tar.gz
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc  # or ~/.bashrc
```

**Verify Installation:**
```bash
helm version
```

**Note:** As of December 2025, Helm 4.0.0 is available, but Helm 3.x is still widely used and recommended for compatibility.

---

## Essential Commands

### Repository Management

```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add eks https://aws.github.io/eks-charts

# List repositories
helm repo list

# Update repository cache
helm repo update

# Remove repository
helm repo remove bitnami
```

### Chart Operations

```bash
# Search for charts
helm search repo nginx

# Install chart
helm install my-nginx bitnami/nginx

# List installed releases
helm list                    # Current namespace
helm list -A                 # All namespaces
helm list -n kube-system     # Specific namespace

# Check release status
helm status my-nginx

# Uninstall release
helm uninstall my-nginx
```

### Upgrade & Rollback

```bash
# Upgrade release
helm upgrade my-nginx bitnami/nginx

# Upgrade with custom values
helm upgrade my-nginx bitnami/nginx --set service.type=NodePort

# View release history
helm history my-nginx

# Rollback to previous version
helm rollback my-nginx 1
```

### Common Flags

- `-n, --namespace`: Specify namespace
- `-f, --values`: Use values file
- `--set`: Override single value
- `--version`: Install specific chart version
- `--dry-run`: Test without installing

---

## Practical Examples

### Example 1: Basic Installation

```bash
# Add repository and install NGINX
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-nginx bitnami/nginx

# Check status
helm status my-nginx
helm list
```

### Example 2: Install with Custom Values

**Using --set:**
```bash
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort \
  --set replicas=3
```

**Using values file:**
Create `nginx-values.yaml`:
```yaml
service:
  type: NodePort
replicaCount: 3
```

```bash
helm install my-nginx bitnami/nginx -f nginx-values.yaml
```

### Example 3: Upgrade and Rollback

```bash
# Upgrade
helm upgrade my-nginx bitnami/nginx --set service.type=LoadBalancer

# View history
helm history my-nginx

# Rollback if needed
helm rollback my-nginx 1
```

---

## Real-World Example: AWS Load Balancer Controller

We used Helm to install the AWS Load Balancer Controller:

```bash
# Add EKS charts repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8s-training-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0

# Verify
helm status aws-load-balancer-controller -n kube-system
helm list -n kube-system
```

**Command breakdown:**
- `aws-load-balancer-controller`: Release name
- `eks/aws-load-balancer-controller`: Chart from eks repository
- `-n kube-system`: Install in kube-system namespace
- `--set`: Override chart values
- `--version`: Install specific chart version

---

## Troubleshooting

### Chart Not Found
```bash
# Check repository is added and updated
helm repo list
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
```

### Installation Fails
```bash
# Check release status and resources
helm status <release-name>
kubectl get pods -l app=<app-name>
kubectl describe pod <pod-name>

# Verify namespace exists
kubectl get namespace <namespace>
```

**Common causes:**
- Invalid values (check `--set` flags)
- Namespace doesn't exist
- Insufficient permissions (check RBAC)
- Resource limits too low

### Upgrade Fails
```bash
# Rollback immediately
helm rollback <release-name>
helm history <release-name>
```

### Release Not Found
```bash
# List all releases
helm list -A
helm list -n <namespace>
```

---

## Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `helm repo add` | Add repository | `helm repo add bitnami https://charts.bitnami.com/bitnami` |
| `helm repo update` | Update cache | `helm repo update` |
| `helm search repo` | Search charts | `helm search repo nginx` |
| `helm install` | Install chart | `helm install my-app bitnami/nginx` |
| `helm list` | List releases | `helm list` or `helm list -A` |
| `helm status` | Check status | `helm status my-app` |
| `helm upgrade` | Upgrade release | `helm upgrade my-app bitnami/nginx` |
| `helm rollback` | Rollback | `helm rollback my-app 1` |
| `helm uninstall` | Remove release | `helm uninstall my-app` |
| `helm history` | View history | `helm history my-app` |

**Popular Repositories:**
- **Bitnami:** `https://charts.bitnami.com/bitnami`
- **EKS Charts (AWS):** `https://aws.github.io/eks-charts`
- **Artifact Hub:** `https://artifacthub.io` (search engine for charts)

---

## References

- **Official Helm Documentation:** https://helm.sh/docs/
- **Helm Quickstart Guide:** https://helm.sh/docs/intro/quickstart/
- **Artifact Hub:** https://artifacthub.io/
- **AWS Load Balancer Controller Guide:** [./07-alb-controller.md](./07-alb-controller.md)
