# Local Kubernetes Setup

**Goal:** Have a working local Kubernetes environment for all exercises.

---

## System Information

- **OS:** Ubuntu 22.04 (Linux 6.8.0-87-generic)
- **Architecture:** x86_64

---

## Installed Tools

### kubectl
- **Version:** v1.33.5-dispatcher
- **Kustomize Version:** v5.6.0
- **Location:** `/usr/bin/kubectl`
- **Status:** ✅ Installed

### minikube
- **Version:** v1.31.2
- **Commit:** fd7ecd9c4599bef9f04c0986c4a0187f98a4396e
- **Location:** `/usr/local/bin/minikube`
- **Status:** ✅ Installed

---

## Setup Steps

### Step 1: Verify kubectl Installation

```bash
kubectl version --client
```

**Output:**
```
Client Version: v1.33.5-dispatcher
Kustomize Version: v5.6.0
```

### Step 2: Start minikube Cluster

```bash
minikube start
```

**Output:**
```
* minikube v1.31.2 on Ubuntu 22.04
* Using the kvm2 driver based on user configuration
* Starting control plane node minikube in cluster minikube
* Creating kvm2 VM (CPUs=6, Memory=11264MB, Disk=60000MB) ...
* Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
* Verifying Kubernetes components...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Enabled addons: storage-provisioner, default-storageclass
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

**Cluster Details:**
- **Kubernetes Version:** v1.27.4
- **Driver:** kvm2
- **VM Resources:** 6 CPUs, 11264MB Memory, 60000MB Disk
- **Container Runtime:** Docker 24.0.4

---

## Verification

### Verify kubectl Version

```bash
kubectl version
```

**Output:**
```
Client Version: v1.33.5-dispatcher
Kustomize Version: v5.6.0
Server Version: v1.27.4
WARNING: version difference between client (1.33) and server (1.27) exceeds the supported minor version skew of +/-1
```

> **Note:** There's a version skew warning between kubectl client (1.33) and server (1.27). This is acceptable for local development, but for production, versions should be within +/-1 minor version.

### Verify Cluster Nodes

```bash
kubectl get nodes
```

**Output:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   52s   v1.27.4
```

### Verify minikube Status

```bash
minikube status
```

**Output:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

---

## Deliverables

**Local cluster running** - minikube cluster is up and running  
**kubectl configured** - kubectl is configured to use the minikube cluster  
**Commands and outputs captured** - All verification commands documented above

---

## Next Steps

The local Kubernetes environment is ready for exercises. You can now:
- Deploy applications to the cluster
- Practice with Kubernetes resources (Pods, Deployments, Services, etc.)
- Run exercises from the `exercises/` directory

---

## Useful Commands

- **Stop cluster:** `minikube stop`
- **Delete cluster:** `minikube delete`
- **View cluster info:** `minikube status`
- **Access minikube dashboard:** `minikube dashboard`
- **Get cluster IP:** `minikube ip`
