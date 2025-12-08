# EKS Cluster Setup Guide

This guide provides step-by-step instructions to provision the EKS cluster for Phase 3 networking exercises using `eksctl`.

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Create the Cluster](#2-create-the-cluster)
3. [Monitor Cluster Creation](#3-monitor-cluster-creation)
4. [Verify the Cluster](#4-verify-the-cluster)
5. [Issue #22 Completion Checklist](#41-issue-22-completion-checklist)
6. [Troubleshooting](#5-troubleshooting)
7. [Common eksctl Commands](#6-common-eksctl-commands)
8. [Cleanup](#7-cleanup-important)
9. [References](#8-references)

---

## 1. Prerequisites

Ensure you have the following installed and configured:

*   **AWS CLI:** `aws --version` (Should be v2.x)
*   **eksctl:** `eksctl version`
*   **kubectl:** `kubectl version --client`

### Install eksctl (if not installed)

**Linux/macOS:**
```bash
# Install to user directory (no sudo required)
mkdir -p ~/.local/bin
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl ~/.local/bin/
export PATH="$HOME/.local/bin:$PATH"  # Add to ~/.zshrc or ~/.bashrc for persistence
eksctl version
```

### Configure AWS Credentials

Verify AWS credentials are configured:
```bash
aws sts get-caller-identity
```

If not configured:
```bash
aws configure
# Enter your Access Key, Secret Key, Region (ap-southeast-1), and Output format (json)
```

---

## 2. Create the Cluster

We will use the configuration file created in `exercises/eks/eksctl-cluster.yaml`.

**Important:** The configuration uses `t3.small` instance type, which is Free Tier eligible. If you're not using Free Tier, you can change it to `t3.medium` or larger for better performance.

**Command:**
```bash
# Navigate to project root (adjust path as needed)
cd <project-root>

# Create cluster
eksctl create cluster -f exercises/eks/eksctl-cluster.yaml
```

**Timeline:**
*   **Total time:** 15-30 minutes
    *   Control plane: ~10 minutes
    *   Node group: ~5-15 minutes (can take up to 30 minutes on first creation)
*   **What happens:**
    1.  Creates a VPC with public and private subnets
    2.  Creates the EKS Control Plane (takes ~10 minutes)
    3.  Creates IAM Roles (Service Role, Node Role)
    4.  Installs addons (VPC CNI, kube-proxy, CoreDNS)
    5.  Creates **1 Managed Node Group** (Public, 1 node) - simplified for cost efficiency
    6.  Updates your `~/.kube/config` automatically

**Important Notes:**
*   `eksctl` has a default timeout (~30-40 minutes) and may exit with an error, **but cluster creation continues in AWS**.
*   Node group creation can take **30-45 minutes** total, especially on first creation.
*   **Do not delete the cluster** if `eksctl` times out - continue monitoring using the commands below.

---

## 3. Monitor Cluster Creation

If `eksctl` times out or you want to monitor progress, use these commands:

**Quick Status Check (Script):**
```bash
./exercises/eks/check-cluster-status.sh
```

**Manual Commands:**

### Check Cluster Status
```bash
eksctl get cluster --region ap-southeast-1 --name k8s-training-cluster
```

### Check Node Group Status
```bash
aws eks describe-nodegroup --region ap-southeast-1 --cluster-name k8s-training-cluster --nodegroup-name public-ng --query 'nodegroup.status' --output text
```

**Expected statuses:**
*   `CREATING` - Node groups are being created (normal, takes 10-15 minutes)
*   `ACTIVE` - Node groups are ready

### Check CloudFormation Stacks
```bash
aws cloudformation describe-stacks --region ap-southeast-1 --stack-name eksctl-k8s-training-cluster-nodegroup-public-ng --query 'Stacks[0].StackStatus' --output text
```

**Expected statuses:**
*   `CREATE_IN_PROGRESS` - Still creating
*   `CREATE_COMPLETE` - Ready

### Update kubeconfig (if needed)
```bash
aws eks update-kubeconfig --region ap-southeast-1 --name k8s-training-cluster
```

---

## 4. Verify the Cluster

Once node groups are `ACTIVE`, verify the cluster is operational.

### Check Nodes
```bash
kubectl get nodes -o wide
```
*   **Expected:** **1 node** (Public)
*   **Status:** `Ready`
*   **IPs:** Should be from VPC CIDR (10.0.0.0/16)

### Check Pods
```bash
kubectl get pods -A -o wide
```
*   **Expected:** System pods (`aws-node`, `coredns`, `kube-proxy`) running
*   **Note:** CoreDNS pods may be `Pending` until nodes are ready

### Verify Networking (VPC CNI)
Check if the pods have IP addresses from the VPC CIDR (10.0.0.0/16).
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide
```

**Expected:** Pod IPs should be in the VPC subnet ranges (e.g., 10.0.x.x).

---

## 4.1. Issue #22 Completion Checklist

Before considering Issue #22 complete, verify all items below:

### Cluster Control Plane
- [ ] Cluster status is `ACTIVE`
  ```bash
  eksctl get cluster --region ap-southeast-1 --name k8s-training-cluster
  ```

### Node Group
- [ ] Node group status is `ACTIVE` (not CREATING)
  ```bash
  aws eks describe-nodegroup --region ap-southeast-1 --cluster-name k8s-training-cluster --nodegroup-name public-ng --query 'nodegroup.status' --output text
  ```

### Worker Nodes
- [ ] At least 1 node is in `Ready` state
  ```bash
  kubectl get nodes -o wide
  ```
  **Expected:** 1 node with status `Ready`, IP from VPC CIDR (10.0.x.x)

### System Pods
- [ ] CoreDNS pods are running (2 pods)
  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  ```
- [ ] VPC CNI (aws-node) pods are running (1 pod per node)
  ```bash
  kubectl get pods -n kube-system -l k8s-app=aws-node
  ```
- [ ] kube-proxy pods are running (1 pod per node)
  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-proxy
  ```

### Networking Verification
- [ ] Pod IPs are from VPC CIDR (10.0.0.0/16)
  ```bash
  kubectl get pods -n kube-system -o wide | grep -E "NAME|aws-node|coredns" | head -5
  ```
  **Expected:** All pod IPs start with `10.0.`

### Quick Verification
```bash
./exercises/eks/check-cluster-status.sh
```

Once all items above are checked, **Issue #22 is complete** and you can proceed to **Issue #23** (AWS Load Balancer Controller installation).

---

## 5. Troubleshooting

### Issue: `eksctl` command not found
*   Install eksctl (see [Prerequisites](#1-prerequisites) section)
*   Ensure `~/.local/bin` is in your PATH

### Issue: Instance type not eligible for Free Tier
*   **Error:** `InvalidParameterCombination - The specified instance type is not eligible for Free Tier`
*   **Cause:** Your AWS account may be in Free Tier mode, which restricts instance types
*   **Free Tier eligible instance types for EKS:**
    | Instance Type | vCPU | RAM | Free Tier | Recommended |
    |--------------|------|-----|-----------|-------------|
    | `t3.micro` | 1 | 1GB | ✅ Yes | ❌ Too small (not recommended) |
    | `t3.small` | 2 | 2GB | ✅ Yes | ✅ **Recommended minimum** |
    | `t3.medium` | 2 | 4GB | ❌ No | ✅ Good if not using Free Tier |
*   **Solution:** Update `exercises/eks/eksctl-cluster.yaml` to use `t3.small`:
    ```yaml
    instanceType: t3.small  # Free Tier eligible
    ```
*   **Note:** If your account is not in Free Tier, you can use `t3.medium` or larger

### Issue: `eksctl` timeout with "exceeded max wait time"
*   **This is expected!** `eksctl` has a default timeout (~30-40 minutes), but EKS node group creation can take 30-45 minutes
*   **The cluster creation continues in AWS even after eksctl exits**
*   **Action:** Continue monitoring using [Section 3](#3-monitor-cluster-creation) commands. Do NOT delete the cluster
*   **Check actual status:**
    ```bash
    # Node group status (should eventually become ACTIVE)
    aws eks describe-nodegroup --region ap-southeast-1 --cluster-name k8s-training-cluster --nodegroup-name public-ng --query 'nodegroup.status' --output text

    # CloudFormation stack status
    aws cloudformation describe-stacks --region ap-southeast-1 --stack-name eksctl-k8s-training-cluster-nodegroup-public-ng --query 'Stacks[0].StackStatus' --output text

    # Check if EC2 instances are launching
    aws ec2 describe-instances --region ap-southeast-1 --filters "Name=tag:eks:nodegroup-name,Values=public-ng" "Name=tag:eks:cluster-name,Values=k8s-training-cluster" --query 'Reservations[*].Instances[*].[InstanceId,State.Name,LaunchTime]' --output table
    ```
*   **Wait up to 45 minutes total** before considering it stuck. Check every 5-10 minutes
*   If status shows `CREATE_FAILED`, check CloudFormation console for specific errors

### Issue: Kubernetes version error
*   **Error:** `invalid version, 1.28 is no longer supported`
*   **Fix:** Update `exercises/eks/eksctl-cluster.yaml` to use a supported version (e.g., `1.29`)
*   **Supported versions:** 1.29, 1.30, 1.31, 1.32, 1.33, 1.34

### Issue: `kubectl` connection refused or no nodes
*   Update your kubeconfig:
    ```bash
    aws eks update-kubeconfig --region ap-southeast-1 --name k8s-training-cluster
    ```
*   If nodes are not showing, wait for node groups to become `ACTIVE` (check with [monitoring commands](#3-monitor-cluster-creation))

### Issue: Node groups stuck in CREATING (taking >45 minutes with no progress)
*   **Check for actual errors:**
    ```bash
    # Check CloudFormation stack events for failures
    aws cloudformation describe-stack-events --region ap-southeast-1 --stack-name eksctl-k8s-training-cluster-nodegroup-public-ng --max-items 30 --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[Timestamp,ResourceType,ResourceStatusReason]' --output table
    ```
*   **Verify prerequisites:**
    1.  **VPC DNS:** Ensure DNS hostnames and DNS resolution are enabled:
        ```bash
        VPC_ID=$(aws eks describe-cluster --region ap-southeast-1 --name k8s-training-cluster --query 'cluster.resourcesVpcConfig.vpcId' --output text)
        aws ec2 describe-vpc-attribute --region ap-southeast-1 --vpc-id $VPC_ID --attribute enableDnsHostnames --query 'EnableDnsHostnames.Value'
        aws ec2 describe-vpc-attribute --region ap-southeast-1 --vpc-id $VPC_ID --attribute enableDnsSupport --query 'EnableDnsSupport.Value'
        ```
    2.  **Subnet capacity:** Check available IPs:
        ```bash
        aws ec2 describe-subnets --region ap-southeast-1 --subnet-ids $(aws eks describe-cluster --region ap-southeast-1 --name k8s-training-cluster --query 'cluster.resourcesVpcConfig.subnetIds' --output text) --query 'Subnets[*].[SubnetId,AvailableIpAddressCount]' --output table
        ```
    3.  **EC2 service limits:**
        ```bash
        aws service-quotas get-service-quota --region ap-southeast-1 --service-code ec2 --quota-code L-0263D0A3
        ```
*   **If truly stuck (>45 min, no instances, no errors):**
    1.  Delete the stuck node group:
        ```bash
        eksctl delete nodegroup --cluster k8s-training-cluster --region ap-southeast-1 --name public-ng
        ```
    2.  Wait for deletion to complete (~5 minutes)
    3.  Recreate with the same config:
        ```bash
        eksctl create nodegroup --config-file exercises/eks/eksctl-cluster.yaml
        ```
*   **Common causes:** IAM permissions, EC2 instance limits, subnet capacity, VPC DNS issues, EKS service delays

---

## 6. Common `eksctl` Commands

### Cluster Management
```bash
# List clusters
eksctl get cluster --region ap-southeast-1

# Get cluster details
eksctl get cluster --name k8s-training-cluster --region ap-southeast-1 -o yaml

# Delete cluster
eksctl delete cluster --name k8s-training-cluster --region ap-southeast-1
```

### Node Group Management
```bash
# List node groups
eksctl get nodegroup --cluster k8s-training-cluster --region ap-southeast-1

# Scale node group
eksctl scale nodegroup --cluster k8s-training-cluster --name public-ng --nodes 3 --region ap-southeast-1

# Delete node group
eksctl delete nodegroup --cluster k8s-training-cluster --name public-ng --region ap-southeast-1
```

### IAM & Auth
```bash
# View IAM identity mapping
eksctl get iamidentitymapping --cluster k8s-training-cluster --region ap-southeast-1
```

---

## 7. Cleanup (Important!)

To avoid AWS charges, delete the cluster when finished.

```bash
eksctl delete cluster -f exercises/eks/eksctl-cluster.yaml
```

**Alternative (if eksctl delete fails):**
```bash
# Delete node group first
eksctl delete nodegroup --cluster k8s-training-cluster --region ap-southeast-1 --name public-ng

# Then delete cluster
eksctl delete cluster --region ap-southeast-1 --name k8s-training-cluster
```

*   **Time:** ~10-15 minutes
*   **Note:** Always clean up to avoid AWS charges!

---

## 8. References

*   [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
*   [eksctl Documentation](https://eksctl.io/)
*   [Amazon VPC CNI Plugin](https://github.com/aws/amazon-vpc-cni-k8s)
*   [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
*   [AWS Free Tier EC2 Instance Types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)
