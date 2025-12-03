# Pod and Deployment Design

## Quick Reference

### Pod vs Deployment vs ReplicaSet

| Feature | Pod | ReplicaSet | Deployment |
|---------|-----|------------|------------|
| **Self-healing** | No | Yes | Yes |
| **Scaling** | No | Yes | Yes |
| **Rolling updates** | No | No | Yes |
| **Rollback** | No | No | Yes |
| **Use case** | One-off, debugging | Pod replication (rare) | Production workloads |
| **When to use** | Testing, debugging | Almost never directly | 99% of production cases |

**Decision Tree:**
```
Need pod replication?
├─ No → Use Pod
└─ Yes → Use Deployment (99% of cases)
```

## Common Use Cases

### When to Use Pod vs Deployment

**Use Pod when:**
- One-off tasks, debugging, testing
- Jobs/CronJobs (they create Pods)
- Init containers (part of Pod spec)

**Use Deployment when:**
- Production workloads (99% of cases)
- Need rolling updates/rollback
- Need self-healing and scaling

## Essential Commands

### Pod Operations

```bash
# List pods
kubectl get pods
kubectl get pods -l app=my-app
kubectl get pods -o wide  # Show node and IP

# Describe pod (see events, conditions, volumes)
kubectl describe pod <pod-name>
kubectl describe pod -l app=my-app

# Execute commands
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- ps aux
kubectl exec <pod-name> -- curl http://localhost:8080/health
```

### Deployment Operations

```bash
# Create/apply deployment
kubectl apply -f deployment.yaml

# Scale deployment
kubectl scale deployment <name> --replicas=3

# Update image
kubectl set image deployment/<name> <container>=<image>:<tag>

# Rollout management
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2

# View deployment
kubectl get deployment <name>
kubectl describe deployment <name>
```

### Label Selectors

```bash
# Filter by label
kubectl get pods -l app=my-app
kubectl get pods -l app=my-app,env=prod
kubectl get pods -l 'app in (app1,app2)'

# View resources with label
kubectl get all -l app=my-app
```

## Resource Limits

### Resource Limits (API Deployment Example)

**CPU:**
- Requests: 100m (0.1 CPU) - Guaranteed minimum
- Limits: 500m (0.5 CPU) - Maximum allowed
- Reasoning: nginx is lightweight, 100m baseline, 500m for traffic spikes

**Memory:**
- Requests: 128Mi - Guaranteed minimum
- Limits: 256Mi - Maximum allowed
- Reasoning: nginx uses 50-100Mi normally, 128Mi ensures availability, 256Mi for headroom

**QoS Class:** Burstable (requests < limits)

**Best Practices:**
- Always set both requests and limits
- Set requests based on typical usage
- Set limits based on maximum acceptable usage
- Monitor with `kubectl top pods`

## Common Issues & Solutions

### Pod Not Starting

**Symptoms:** `Pending`, `ErrImagePull`, `ImagePullBackOff`

**Commands:**
```bash
kubectl describe pod <pod-name>  # Check events
kubectl get events --field-selector involvedObject.name=<pod-name>
```

**Common causes:**
- Invalid image tag → Fix image name/tag
- Insufficient resources → Check node capacity: `kubectl describe node`
- Image pull errors → Check registry access, image pull secrets
- Node selector mismatch → Check pod spec and node labels

### Pod Crashing

**Symptoms:** `CrashLoopBackOff`, `RESTARTS` increasing

**Commands:**
```bash
kubectl logs <pod-name>  # Current container
kubectl logs <pod-name> --previous  # Previous container
kubectl describe pod <pod-name>  # See events
```

**Common causes:**
- Application errors → Check logs for error messages
- Missing configuration → Verify ConfigMap/Secret exists and keys match
- Wrong command/args → Check container spec
- Missing dependencies → Check init containers, dependencies

### Resource Issues

**Symptoms:** Pod `Pending`, `OOMKilled`, CPU throttling

**Commands:**
```bash
# Check resource usage
kubectl top pods
kubectl top nodes

# Check pod resource requests/limits
kubectl describe pod <pod-name> | grep -A 5 "Limits\|Requests"

# Check node capacity
kubectl describe node <node-name>
```

**Common causes:**
- Insufficient resources → Increase requests/limits or add nodes
- Memory limit too low → Increase memory limit
- CPU throttling → Increase CPU limit
- Node capacity exceeded → Scale cluster or reduce resource requests

**Fix:**
```yaml
resources:
  requests:
    cpu: 200m      # Increase if pod can't start
    memory: 256Mi  # Increase if OOMKilled
  limits:
    cpu: 1000m     # Increase if throttled
    memory: 512Mi  # Increase if OOMKilled
```
