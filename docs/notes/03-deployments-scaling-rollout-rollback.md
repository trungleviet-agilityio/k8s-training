# Deployments – Scaling, Rollout and Rollback

**Goal:** Be comfortable creating and managing Deployments.

---

## Deployment Manifest

Created `exercises/deployments/nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Key Components:

- **apiVersion:** `apps/v1` (required for Deployments)
- **replicas:** Desired number of pod replicas (3 initially)
- **selector:** Labels used to identify pods managed by this Deployment
- **template:** Pod template that defines what pods should look like
- **matchLabels:** Must match the labels in the pod template

---

## Initial Deployment

### Apply the Deployment

```bash
kubectl apply -f exercises/deployments/nginx-deployment.yaml
```

**Output:**
```
deployment.apps/nginx-deployment created
```

### Verify Deployment, ReplicaSets, and Pods

```bash
kubectl get deploy,rs,pods -l app=nginx
```

**Output:**
```
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           14s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-667c588598   3         3         3       14s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-667c588598-flg7n   1/1     Running   0          14s
pod/nginx-deployment-667c588598-jlc9x   1/1     Running   0          14s
pod/nginx-deployment-667c588598-spw9m   1/1     Running   0          14s
```

### Understanding the Output:

**Deployment:**
- **READY:** 3/3 (current ready / desired replicas)
- **UP-TO-DATE:** Number of replicas updated to match desired state
- **AVAILABLE:** Number of replicas available for use

**ReplicaSet:**
- **DESIRED:** Target number of replicas
- **CURRENT:** Current number of replicas
- **READY:** Number of ready replicas

**Pods:**
- All pods are Running and Ready (1/1)

---

## Scaling Operations

### Scale Up to 5 Replicas

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

**Output:**
```
deployment.apps/nginx-deployment scaled
```

**Verify:**
```bash
kubectl get deploy,rs,pods -l app=nginx
```

**Output:**
```
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   5/5     5            5           96s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-667c588598   5         5         5       96s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-667c588598-4dq8w   1/1     Running   0          38s
pod/nginx-deployment-667c588598-flg7n   1/1     Running   0          96s
pod/nginx-deployment-667c588598-jlc9x   1/1     Running   0          96s
pod/nginx-deployment-667c588598-nswgh   1/1     Running   0          38s
pod/nginx-deployment-667c588598-spw9m   1/1     Running   0          96s
```

**Observations:**
- Deployment scaled from 3 to 5 replicas
- ReplicaSet automatically adjusted to 5 replicas
- 2 new pods were created (4dq8w, nswgh)

### Scale Down to 2 Replicas

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

**Output:**
```
deployment.apps/nginx-deployment scaled
```

**Verify:**
```bash
kubectl get deploy,rs,pods -l app=nginx
```

**Output:**
```
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           113s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-667c588598   2         2         2       113s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-667c588598-flg7n   1/1     Running   0          113s
pod/nginx-deployment-667c588598-spw9m   1/1     Running   0          113s
```

**Observations:**
- Deployment scaled from 5 to 2 replicas
- 3 pods were terminated
- Only 2 pods remain running

---

## Rolling Update

### Update Image to nginx:1.23

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.23
```

**Output:**
```
deployment.apps/nginx-deployment image updated
```

### Watch the Rolling Update

```bash
kubectl rollout status deployment/nginx-deployment
```

**Output:**
```
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

**Observations:**
- Rolling update strategy: new pods are created before old ones are terminated
- Ensures zero-downtime deployment
- One pod at a time is updated

### Verify After Update

```bash
kubectl get deploy,rs,pods -l app=nginx
```

**Output:**
```
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           2m11s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-58d66cddbd   2         2         2       15s
replicaset.apps/nginx-deployment-667c588598   0         0         0       2m11s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-58d66cddbd-lmp69   1/1     Running   0          15s
pod/nginx-deployment-58d66cddbd-vtxmk   1/1     Running   0          4s
```

**Key Observations:**
- **New ReplicaSet created:** `nginx-deployment-58d66cddbd` (with nginx:1.23)
- **Old ReplicaSet scaled to 0:** `nginx-deployment-667c588598` (nginx:1.21)
- **New pods:** All pods have new names with the new ReplicaSet hash
- **Zero downtime:** Service remained available during the update

### How Rolling Updates Work:

1. **New ReplicaSet created** with updated image
2. **New pods created** gradually (based on `maxSurge` and `maxUnavailable`)
3. **Old pods terminated** as new ones become ready
4. **Old ReplicaSet retained** (for rollback capability) but scaled to 0

---

## Rollback

### View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

**Output (before rollback):**
```
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

**Understanding Revisions:**
- **Revision 1:** Initial deployment (nginx:1.21)
- **Revision 2:** After image update (nginx:1.23)

> **Note:** `CHANGE-CAUSE` is `<none>` because we didn't use `--record` flag. To record change causes, use:
> ```bash
> kubectl set image deployment/nginx-deployment nginx=nginx:1.23 --record
> ```

### Perform Rollback

```bash
kubectl rollout undo deployment/nginx-deployment
```

**Output:**
```
deployment.apps/nginx-deployment rolled back
```

### Verify Rollback Status

```bash
kubectl rollout status deployment/nginx-deployment
```

**Output:**
```
deployment "nginx-deployment" successfully rolled out
```

### Verify After Rollback

```bash
kubectl get deploy,rs,pods -l app=nginx
```

**Output:**
```
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           2m31s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-58d66cddbd   0         0         0       35s
replicaset.apps/nginx-deployment-667c588598   2         2         2       2m31s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-667c588598-6rsd9   1/1     Running   0          5s
pod/nginx-deployment-667c588598-ljslb   1/1     Running   0          4s
```

**Observations:**
- Old ReplicaSet (`667c588598` with nginx:1.21) is back with 2 replicas
- New ReplicaSet (`58d66cddbd` with nginx:1.23) scaled to 0
- New pods created with the original image

### View History After Rollback

```bash
kubectl rollout history deployment/nginx-deployment
```

**Output:**
```
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

**Understanding:**
- **Revision 2:** nginx:1.23 (previous state)
- **Revision 3:** nginx:1.21 (current state after rollback)

> **Note:** Rollback creates a new revision, so the history grows.

### Rollback to Specific Revision

You can also rollback to a specific revision:

```bash
# View detailed history
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

---

## Key Commands Summary

### Deployment Management

```bash
# Create/Apply deployment
kubectl apply -f deployment.yaml

# View deployments, replicasets, and pods
kubectl get deploy,rs,pods -l app=nginx

# Scale deployment
kubectl scale deployment <name> --replicas=<number>

# Update image
kubectl set image deployment/<name> <container>=<image>:<tag>

# Watch rollout status
kubectl rollout status deployment/<name>
```

### Rollout History and Rollback

```bash
# View rollout history
kubectl rollout history deployment/<name>

# View specific revision details
kubectl rollout history deployment/<name> --revision=<number>

# Rollback to previous revision
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=<number>

# Pause rollout (useful for canary deployments)
kubectl rollout pause deployment/<name>

# Resume rollout
kubectl rollout resume deployment/<name>
```

---

## Key Learnings

### 1. **Deployment vs Pod**
- **Pod:** Single instance, not managed for high availability
- **Deployment:** Manages ReplicaSets, which manage Pods
- Deployments provide self-healing, scaling, and rolling updates

### 2. **ReplicaSet Relationship**
- Deployment creates and manages ReplicaSets
- Each update creates a new ReplicaSet
- Old ReplicaSets are retained for rollback capability

### 3. **Rolling Update Strategy**
- **Default behavior:** Rolling update (zero downtime)
- New pods are created before old ones are terminated
- Controlled by `maxSurge` and `maxUnavailable` settings

### 4. **Scaling**
- Can scale up or down instantly
- Scaling doesn't create new ReplicaSet (same image)
- Image updates create new ReplicaSet

### 5. **Rollback Mechanism**
- Kubernetes keeps history of ReplicaSets
- Rollback is instant and safe
- Can rollback to any previous revision
- Rollback creates a new revision (history grows)

### 6. **Labels and Selectors**
- **Critical:** Labels in `spec.selector.matchLabels` must match labels in `spec.template.metadata.labels`
- Mismatch will cause deployment to fail
- Labels are used to identify which pods belong to the deployment

### 7. **ReplicaSet Naming**
- ReplicaSets are named: `<deployment-name>-<hash>`
- Hash is based on pod template
- Same template = same hash (even across updates)

---

## Deliverables

**Deployment manifest** created in `exercises/deployments/nginx-deployment.yaml`

**Documentation** covering:
- `kubectl get deploy,rs,pods` output and interpretation
- Rollout history commands (`kubectl rollout history`)
- Rollback commands (`kubectl rollout undo`)
- Scaling operations
- Rolling update process

---

## Best Practices

1. **Always use Deployments** instead of Pods for production workloads
2. **Set appropriate replica counts** for high availability
3. **Use labels consistently** for proper pod selection
4. **Record change causes** with `--record` flag for better history
5. **Test rollbacks** in staging before production
6. **Monitor rollout status** during updates
7. **Use resource limits** (covered in later exercises)

---

## Next Steps

- Learn about Deployment strategies (RollingUpdate vs Recreate)
- Explore Deployment configuration (maxSurge, maxUnavailable)
- Practice with multi-container deployments
- Learn about resource requests and limits
