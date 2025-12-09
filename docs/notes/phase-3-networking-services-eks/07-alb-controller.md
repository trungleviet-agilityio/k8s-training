# AWS Load Balancer Controller

## Overview

The AWS Load Balancer Controller is a controller that manages AWS Elastic Load Balancers for Kubernetes clusters. When you create a Kubernetes Ingress resource, the controller automatically provisions an AWS Application Load Balancer (ALB) that routes traffic to your application pods.

### Key Features

- **Automatic ALB Provisioning:** Creates and configures AWS Application Load Balancers based on Kubernetes Ingress resources
- **IP Target Mode:** Routes traffic directly to pod IPs using VPC CNI (recommended for EKS)
- **Path-based & Host-based Routing:** Supports HTTP/HTTPS routing rules
- **SSL/TLS Termination:** Handles certificate management with AWS Certificate Manager (ACM)
- **Health Checks:** Automatic pod health monitoring through ALB target groups
- **IRSA Integration:** Uses IAM Roles for Service Accounts for secure AWS API access

---

## Prerequisites

Before installing the AWS Load Balancer Controller, ensure you have:

1. **EKS Cluster:** An operational Amazon EKS cluster
   ```bash
   eksctl get cluster --region ap-southeast-1 --name k8s-training-cluster
   ```

2. **IAM OIDC Provider:** Enabled for the cluster (required for IRSA)
   ```bash
   aws eks describe-cluster --region ap-southeast-1 --name k8s-training-cluster --query 'cluster.identity.oidc.issuer'
   ```

3. **kubectl:** Configured to communicate with your EKS cluster
   ```bash
   kubectl get nodes
   ```

4. **Helm:** Package manager for Kubernetes (v3.x)
   ```bash
   helm version
   ```

5. **AWS CLI:** Configured with appropriate credentials
   ```bash
   aws sts get-caller-identity
   ```

---

## Installation Steps

### Step 1: Download IAM Policy

Download the IAM policy JSON that grants the controller permissions to manage AWS resources.

```bash
cd exercises/eks
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

**Verify download:**
```bash
ls -lh iam_policy.json
```

### Step 2: Create IAM Policy

Create the IAM policy in your AWS account.

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

**Expected output:**
```json
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "Arn": "arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy",
        ...
    }
}
```

**Note:** If the policy already exists, you'll see an error. This is fine - you can reuse the existing policy across multiple clusters.

### Step 3: Create IAM Service Account with IRSA

Create a Kubernetes ServiceAccount linked to an IAM role using IAM Roles for Service Accounts (IRSA).

```bash
eksctl create iamserviceaccount \
    --cluster=k8s-training-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-southeast-1 \
    --approve
```

**Expected output:**
```
[ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
```

**What this does:**
- Creates an IAM role with the Load Balancer Controller policy attached
- Creates a Kubernetes ServiceAccount annotated with the IAM role ARN
- Sets up the trust relationship between the IAM role and the cluster's OIDC provider

### Step 4: Install Helm (if not installed)

If Helm is not already installed, install it to your user directory:

```bash
mkdir -p ~/.local/bin
cd /tmp
curl -fsSL https://get.helm.sh/helm-v3.19.2-linux-amd64.tar.gz -o helm.tar.gz
tar -zxvf helm.tar.gz
mv linux-amd64/helm ~/.local/bin/helm
rm -rf linux-amd64 helm.tar.gz
export PATH="$HOME/.local/bin:$PATH"
helm version
```

**Add to shell profile (optional):**
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
# or for bash: echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```

### Step 5: Add Helm Repository

Add the EKS charts Helm repository maintained by AWS.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

**Expected output:**
```
"eks" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "eks" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Step 6: Install AWS Load Balancer Controller

Install the controller using Helm.

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8s-training-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

**Parameters explained:**
- `clusterName`: Name of your EKS cluster
- `serviceAccount.create=false`: Don't create a new ServiceAccount (we created it with `eksctl`)
- `serviceAccount.name`: Use the ServiceAccount we created in Step 3
- `version 1.14.0`: Specific chart version (corresponds to controller v2.14.x)

**Expected output:**
```
NAME: aws-load-balancer-controller
LAST DEPLOYED: ...
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
...
AWS Load Balancer controller installed!
```

---

## Verification

### Check Deployment Status

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**Expected output:**
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           2m
```

### Check Pods

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Expected output:**
```
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-xxxxxxxxx-xxxxx   1/1     Running   0          2m
aws-load-balancer-controller-xxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### Check Logs

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

**Look for:**
- `"Starting Controller"` messages
- `"Starting workers"` messages
- No error messages

### Verify ServiceAccount

```bash
kubectl get serviceaccount -n kube-system aws-load-balancer-controller -o yaml
```

**Check for annotation:**
```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/eksctl-k8s-training-cluster-addon-iamserviceacco-Role1-...
```

---

## Testing with Sample Application

### Create Test Resources

**1. Deploy Nginx Application:**

```bash
kubectl apply -f exercises/eks/test-nginx-deployment.yaml
```

**2. Create ClusterIP Service:**

```bash
kubectl apply -f exercises/eks/test-nginx-service.yaml
```

**3. Create Ingress with ALB:**

```bash
kubectl apply -f exercises/eks/test-nginx-ingress.yaml
```

### Monitor ALB Provisioning

**Check Ingress status:**
```bash
kubectl get ingress nginx-test-ingress
```

**Expected output (initially):**
```
NAME                 CLASS   HOSTS   ADDRESS   PORTS   AGE
nginx-test-ingress   alb     *                 80      10s
```

**After 1-2 minutes:**
```
NAME                 CLASS   HOSTS   ADDRESS                                                                 PORTS   AGE
nginx-test-ingress   alb     *       k8s-default-nginxtes-xxxxxxxx-xxxxxxxxxx.ap-southeast-1.elb.amazonaws.com   80      2m
```

### Verify Ingress Details

```bash
kubectl describe ingress nginx-test-ingress
```

**Look for:**
- **Address:** ALB DNS name
- **Annotations:** Check `alb.ingress.kubernetes.io/*` annotations
- **Rules:** Path routing configuration
- **Events:** `Successfully reconciled` message

### Check AWS Resources

**List ALBs:**
```bash
aws elbv2 describe-load-balancers --region ap-southeast-1 \
  --query 'LoadBalancers[?contains(DNSName, `k8s-default-nginxtes`)].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}' \
  --output table
```

**Check Target Group Health:**
```bash
TG_ARN=$(aws elbv2 describe-target-groups --region ap-southeast-1 \
  --query 'TargetGroups[?contains(LoadBalancerArns[0], `k8s-default-nginxtes`)].TargetGroupArn' \
  --output text)

aws elbv2 describe-target-health --region ap-southeast-1 \
  --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State}' \
  --output table
```

**Expected:** Targets should be `healthy`

### Test HTTP Access

```bash
# Get ALB DNS name
ALB_DNS=$(kubectl get ingress nginx-test-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test endpoint
curl -I http://$ALB_DNS
```

**Expected output:**
```
HTTP/1.1 200 OK
Server: nginx/1.x.x
...
```

**Test in browser:**
Open `http://<ALB_DNS>` in your browser - you should see the Nginx welcome page.

---

## Key Concepts

### IRSA (IAM Roles for Service Accounts)

**How it works:**
1. The cluster has an OIDC provider (configured with `withOIDC: true` in `eksctl-cluster.yaml`)
2. The ServiceAccount is annotated with an IAM role ARN
3. When pods use this ServiceAccount, they automatically assume the IAM role
4. The controller can then make AWS API calls to create/manage load balancers

**Benefits:**
- No need for instance profiles or hardcoded credentials
- Fine-grained permissions per application
- Follows AWS security best practices

### Target Type: IP Mode vs Instance Mode

**Instance Mode (Legacy):**
- ALB → NodePort → kube-proxy → Pod
- Extra network hop through `kube-proxy`
- Can cause uneven load distribution

**IP Mode (Recommended for EKS):**
- ALB → Pod IP (directly)
- Requires VPC CNI (native to EKS)
- Lower latency, better load distribution
- Annotation: `alb.ingress.kubernetes.io/target-type: ip`

**Why IP mode works in EKS:**
EKS uses the VPC CNI plugin, which assigns VPC IP addresses directly to pods. This makes pods first-class citizens in the VPC, allowing the ALB to route directly to pod IPs.

### Common Annotations

**Scheme:**
```yaml
alb.ingress.kubernetes.io/scheme: internet-facing  # or 'internal'
```

**Target Type:**
```yaml
alb.ingress.kubernetes.io/target-type: ip  # or 'instance'
```

**Health Check:**
```yaml
alb.ingress.kubernetes.io/healthcheck-path: /health
alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
```

**SSL/TLS:**
```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/xxx
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

**Subnet Selection:**
```yaml
alb.ingress.kubernetes.io/subnets: subnet-xxx, subnet-yyy  # Public subnets for internet-facing
```

**For more annotations, see:** [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)

---

## Troubleshooting

### Issue: Controller pods not starting

**Symptoms:** Pods stuck in `Pending` or `CrashLoopBackOff`

**Check:**
```bash
kubectl describe pod -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Common causes:**
- **IAM permissions:** Check ServiceAccount annotation and IAM role
- **OIDC provider:** Ensure cluster has OIDC provider enabled
- **Resource limits:** Check if node has sufficient resources

**Fix:**
```bash
# Verify OIDC provider
aws eks describe-cluster --region ap-southeast-1 --name k8s-training-cluster \
  --query 'cluster.identity.oidc.issuer'

# Verify ServiceAccount
kubectl get sa -n kube-system aws-load-balancer-controller -o yaml
```

### Issue: Ingress not getting ALB address

**Symptoms:** `ADDRESS` field remains empty after several minutes

**Check:**
```bash
kubectl describe ingress nginx-test-ingress
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller | grep -i error
```

**Common causes:**
- **IngressClass mismatch:** Ensure `ingressClassName: alb` is set
- **No public subnets:** ALB needs public subnets (for internet-facing)
- **IAM permissions:** Controller can't create load balancers
- **Controller not running:** Check controller pods are running

**Fix:**
```bash
# Check IngressClass exists
kubectl get ingressclass alb

# Verify ingress events
kubectl describe ingress nginx-test-ingress
```

### Issue: ALB created but targets unhealthy

**Symptoms:** ALB exists but HTTP requests fail or timeout

**Check target health:**
```bash
TG_ARN=$(aws elbv2 describe-target-groups --region ap-southeast-1 \
  --query 'TargetGroups[?contains(LoadBalancerArns[0], `k8s-default-nginxtes`)].TargetGroupArn' \
  --output text)

aws elbv2 describe-target-health --region ap-southeast-1 --target-group-arn $TG_ARN
```

**Common causes:**
- **Wrong target port:** Service port doesn't match container port
- **Security groups:** Pods not reachable from ALB security group
- **Health check path:** ALB health check failing (check annotation)
- **No endpoints:** Service has no backend pods

**Fix:**
```bash
# Check service endpoints
kubectl get endpoints nginx-test

# Check pod status
kubectl get pods -l app=nginx-test

# Verify service selector matches pod labels
kubectl get svc nginx-test -o yaml
kubectl get pods -l app=nginx-test --show-labels
```

### Issue: Connection timeout when accessing ALB

**Symptoms:** `curl` times out, browser can't connect

**Check security groups:**
```bash
# Get ALB security group
ALB_ARN=$(aws elbv2 describe-load-balancers --region ap-southeast-1 \
  --query 'LoadBalancers[?contains(DNSName, `k8s-default-nginxtes`)].LoadBalancerArn' --output text)

SG_IDS=$(aws elbv2 describe-load-balancers --region ap-southeast-1 \
  --load-balancer-arns $ALB_ARN --query 'LoadBalancers[0].SecurityGroups' --output text)

# Check inbound rules
aws ec2 describe-security-groups --region ap-southeast-1 --group-ids $SG_IDS
```

**Common causes:**
- **Security group rules:** No inbound rule for port 80/443 from `0.0.0.0/0`
- **ALB still provisioning:** Wait 2-3 minutes after creation
- **Wrong scheme:** Using `internal` instead of `internet-facing`

**Fix:**
The controller should automatically create security group rules. If not, check controller logs for errors.

### Issue: `helm install` fails with "already exists"

**Symptoms:** Controller already installed

**Check existing installation:**
```bash
helm list -n kube-system
```

**Options:**
1. **Upgrade instead:** `helm upgrade aws-load-balancer-controller ...`
2. **Uninstall and reinstall:** `helm uninstall aws-load-balancer-controller -n kube-system`

---

## Cleanup

### Remove Test Application

```bash
kubectl delete -f exercises/eks/test-nginx-ingress.yaml
kubectl delete -f exercises/eks/test-nginx-service.yaml
kubectl delete -f exercises/eks/test-nginx-deployment.yaml
```

**Wait for ALB deletion:**
The controller will automatically delete the ALB. This takes 1-2 minutes.

```bash
# Verify ALB is deleted
aws elbv2 describe-load-balancers --region ap-southeast-1 \
  --query 'LoadBalancers[?contains(DNSName, `k8s-default-nginxtes`)]'
```

### Uninstall Controller (Optional)

**If you want to completely remove the controller:**

```bash
# Uninstall Helm chart
helm uninstall aws-load-balancer-controller -n kube-system

# Delete service account
eksctl delete iamserviceaccount \
  --cluster=k8s-training-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --region ap-southeast-1

# Delete IAM policy (optional)
aws iam delete-policy \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

---

## References

- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [AWS Official EKS Guide - Install with Helm](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)
- [AWS Load Balancer Controller GitHub](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
- [Ingress Annotations Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)
- [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [EKS Networking Concepts](./05-eks-networking-concepts.md)
- [EKS Setup Guide](./06-eks-setup-guide.md)
