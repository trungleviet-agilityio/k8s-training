# Phase 3 Conclusion & Resume Guide

## Phase 3 Completion Summary

Congratulations! You have successfully completed **Phase 3: Networking, Services & AWS EKS Basics**.

**Key Achievements:**
1.  **Local Networking:** Mastered Services (ClusterIP, NodePort), Ingress, and NetworkPolicies on Minikube.
2.  **EKS Architecture:** Understood AWS EKS networking (VPC CNI, IP target mode, ALBs).
3.  **Cluster Operations:** Provisioned a cost-effective EKS cluster using `eksctl`.
4.  **Traffic Management:** Installed the **AWS Load Balancer Controller** using Helm and IRSA.
5.  **Real-World Deployment:** Deployed the "2048 Game" application and exposed it to the internet via an Application Load Balancer.

---

## Resume Guide: How to Restart Your EKS Environment

Because EKS clusters incur costs, you likely deleted your resources after finishing the exercises. Use this guide to quickly rebuild your environment when you're ready to continue to Phase 4.

### 1. Recreate the Cluster
Use the configuration file we created to provision the cluster again.

```bash
# Navigate to project root
cd k8s-training

# Create cluster (takes ~15-20 mins)
eksctl create cluster -f exercises/eks/eksctl-cluster.yaml
```

**Verify:**
```bash
kubectl get nodes
```

### 2. Reinstall AWS Load Balancer Controller
The controller is required for Ingress resources to work.

**A. Create IAM Policy (if you deleted it)**
```bash
# Check if policy exists
aws iam get-policy --policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/AWSLoadBalancerControllerIAMPolicy

# If not, create it
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://exercises/eks/iam_policy.json
```

**B. Create Service Account (IRSA)**
```bash
eksctl create iamserviceaccount \
  --cluster=k8s-training-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region ap-southeast-1 \
  --approve
```

**C. Install Controller with Helm**
```bash
# Add repo if needed
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8s-training-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

**Verify:**
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### 3. Redeploy Sample App
To confirm everything is working:

```bash
kubectl apply -f exercises/eks-sample-app/
```

Wait for the Ingress to get an address:
```bash
kubectl get ingress -n game-2048 -w
```

---

## Cleanup Reference (Tear Down)

When you are done working for the day, **always clean up** to avoid AWS charges.

### 1. Delete Applications
This deletes the Load Balancers (ALBs) created by Ingress. **Crucial:** Delete Ingress resources *before* deleting the cluster to ensure ALBs are removed.

```bash
# Delete Sample App
kubectl delete -f exercises/eks-sample-app/

# Delete Test App (if deployed)
kubectl delete -f exercises/eks/test-nginx-ingress.yaml
kubectl delete -f exercises/eks/test-nginx-service.yaml
kubectl delete -f exercises/eks/test-nginx-deployment.yaml
```

**Verify:** Check AWS Console (EC2 -> Load Balancers) to ensure ALBs are gone.

### 2. Uninstall Controller (Optional but Recommended)
```bash
helm uninstall aws-load-balancer-controller -n kube-system
```

### 3. Delete Cluster
This removes the nodes (EC2), Control Plane, and VPC (if created by eksctl).

```bash
eksctl delete cluster -f exercises/eks/eksctl-cluster.yaml
```

*Note: If deletion fails, check CloudFormation in AWS Console.*
