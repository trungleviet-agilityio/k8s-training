# Phase 2 – Workloads & Configuration (Application-level K8s)

**Quick reference for Kubernetes workloads, configuration, and debugging commands.**

---

## 1. Quick Reference Tables

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

### ConfigMap vs Secret

| Criteria | ConfigMap | Secret |
|----------|-----------|--------|
| **Data type** | Non-sensitive | Sensitive |
| **Encoding** | Plain text | Base64 encoded |
| **Examples** | App config, env vars | Passwords, tokens, keys |
| **Git commit** | OK | Never |

### Probe Types

| Probe Type | Purpose | Failure Action | When to Use |
|------------|---------|----------------|-------------|
| **Readiness** | Is container ready? | Remove from Service | Container needs initialization |
| **Liveness** | Is container alive? | Restart container | App can get stuck |
| **Startup** | Is container started? | Restart container | Slow-starting apps |

### Injection Methods

| Method | Use Case | Example |
|--------|----------|---------|
| **env (individual)** | Specific keys, rename | `valueFrom.configMapKeyRef` |
| **envFrom (all keys)** | All keys needed | `envFrom.configMapRef` |
| **Volume mounts** | Files, hot reload | Mount at `/etc/config` |

---

## 2. Essential Commands

### Pod Operations

```bash
# List pods
kubectl get pods
kubectl get pods -l app=my-app
kubectl get pods -o wide  # Show node and IP

# Describe pod (see events, conditions, volumes)
kubectl describe pod <pod-name>
kubectl describe pod -l app=my-app

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -f  # Follow
kubectl logs <pod-name> --tail=50
kubectl logs <pod-name> --previous  # Previous container instance

# Execute commands
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- ps aux
kubectl exec <pod-name> -- curl http://localhost:8080/health

# Multi-container pods
kubectl logs <pod-name> -c <container-name>
kubectl exec <pod-name> -c <container-name> -- <command>
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

### ConfigMap & Secret Operations

```bash
# Create ConfigMap
kubectl create configmap <name> --from-literal=KEY=value
kubectl create configmap <name> --from-file=config.properties
kubectl apply -f configmap.yaml

# Create Secret
kubectl create secret generic <name> --from-literal=KEY=value
kubectl create secret generic <name> --from-file=username.txt
echo -n "value" | base64  # Encode for YAML

# View ConfigMap/Secret
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml
kubectl describe configmap <name>
kubectl describe secret <name>

# Decode Secret value
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 -d
```

### Debugging Workflow

```bash
# 1. Quick status
kubectl get pods -l app=my-app

# 2. Detailed info (see events)
kubectl describe pod <pod-name>

# 3. Application logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # If crashed

# 4. Interactive debugging
kubectl exec -it <pod-name> -- /bin/sh
```

### Resource Monitoring

```bash
# View resource usage
kubectl top pods
kubectl top pods -l app=my-app
kubectl top nodes

# View events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod-name>

# View all resources
kubectl get all
kubectl get all -l app=my-app
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

---

## 3. Common Use Cases

### When to Use Pod vs Deployment

**Use Pod when:**
- One-off tasks, debugging, testing
- Jobs/CronJobs (they create Pods)
- Init containers (part of Pod spec)

**Use Deployment when:**
- Production workloads (99% of cases)
- Need rolling updates/rollback
- Need self-healing and scaling

### When to Use ConfigMap vs Secret

**Use ConfigMap for:**
- Non-sensitive config (APP_MODE, LOG_LEVEL, API_ENDPOINT)
- Configuration files
- Environment variables

**Use Secret for:**
- Passwords, tokens, API keys
- Database credentials
- TLS certificates

**Note:** Secrets are base64 encoded, NOT encrypted. Use external secret management for production.

### When to Use Each Probe Type

**Readiness Probe:**
- Container needs initialization time
- Database connections must be established
- Configuration must be loaded
- **Failure:** Pod removed from Service, container keeps running

**Liveness Probe:**
- Application can get stuck (deadlocks, infinite loops)
- Need automatic recovery
- **Failure:** Container restarted

**Startup Probe:**
- Slow-starting applications (Java apps, databases)
- Prevents premature restarts
- **Failure:** Container restarted, other probes disabled until startup succeeds

### Multi-container Patterns

**Sidecar Pattern:**
- Log aggregation (main app writes, sidecar ships)
- Monitoring (sidecar collects metrics)
- Proxy/network helper

**Key points:**
- Containers share network (localhost) and volumes
- Both start/stop together
- Use when containers need tight coupling

---

## 4. Common Issues & Solutions

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

### Probe Failures

**Symptoms:** `READY: 0/1` (readiness), `RESTARTS` increasing (liveness)

**Commands:**
```bash
kubectl describe pod <pod-name> | grep -A 10 "Readiness\|Liveness"
kubectl exec <pod-name> -- curl http://localhost:8080/health  # Test endpoint
```

**Common causes:**
- Wrong probe path → Verify endpoint exists
- Wrong probe port → Check container port
- Application not responding → Check app logs
- Probe timing too aggressive → Increase `initialDelaySeconds` or `failureThreshold`

**Fix:**
```yaml
readinessProbe:
  httpGet:
    path: /ready  # Verify path exists
    port: 8080    # Verify port matches
  initialDelaySeconds: 10  # Increase if app starts slowly
  failureThreshold: 3       # Allow transient failures
```

### Configuration Not Working

**Symptoms:** Missing env vars, wrong values, files not mounted

**Commands:**
```bash
# Check env vars
kubectl exec <pod-name> -- env | grep KEY

# Check ConfigMap/Secret
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml

# Check mounted volumes
kubectl exec <pod-name> -- ls -la /etc/config
kubectl exec <pod-name> -- cat /etc/config/<file>

# Check pod spec
kubectl get pod <pod-name> -o yaml | grep -A 20 "env\|volumeMounts"
```

**Common causes:**
- ConfigMap/Secret not created → `kubectl get configmap <name>`
- Wrong key names → Verify keys match in ConfigMap/Secret and pod spec
- Volume mount path incorrect → Check `mountPath` in pod spec
- Env var not set → Verify `env` or `envFrom` in container spec

**Fix:**
```yaml
# Verify ConfigMap exists
kubectl get configmap app-config

# Verify keys match
kubectl get configmap app-config -o jsonpath='{.data}'

# In pod spec, ensure key names match
env:
- name: APP_MODE
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_MODE  # Must match key in ConfigMap
```

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

### Multi-container Issues

**Symptoms:** Sidecar not working, volumes not shared, containers can't communicate

**Commands:**
```bash
# Check all containers
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'

# Check container logs
kubectl logs <pod-name> -c <container-name>

# Check volume mounts
kubectl describe pod <pod-name> | grep -A 10 "Volumes\|Mounts"

# Test localhost communication
kubectl exec <pod-name> -c main -- curl http://localhost:8080
```

**Common causes:**
- Volume not defined at pod level → Define in `spec.volumes`
- Volume not mounted in container → Add to `volumeMounts`
- Wrong mount path → Verify paths match
- Containers can't communicate → Use `localhost` (same network namespace)

**Fix:**
```yaml
# Volume must be defined at pod level
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  containers:
  - name: main
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  - name: sidecar
    volumeMounts:
    - name: shared-logs  # Same volume name
      mountPath: /var/log
```

---

## 5. Resource Limits & Probes

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

### Probe Configuration (API Deployment Example)

**Readiness Probe:**
```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5   # Short delay, route traffic quickly
  periodSeconds: 10
  failureThreshold: 3
```

**Liveness Probe:**
```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 30  # Longer delay, avoid premature restarts
  periodSeconds: 10
  failureThreshold: 3
```

**Key Differences:**
- Readiness: Shorter delay (5s) to route traffic quickly
- Liveness: Longer delay (30s) to avoid unnecessary restarts
- Both use same endpoint for simplicity

**Common Pitfalls:**
- Same `initialDelaySeconds` for both → Causes premature restarts
- Too low `initialDelaySeconds` → Causes restart loops
- Too low `failureThreshold` → Causes unnecessary restarts
- Heavy probe endpoints → Impacts application performance

---

## 6. Quick Reference: Problem → Command

| Problem | Command |
|---------|---------|
| Pod not starting | `kubectl describe pod <pod-name>` |
| Pod crashing | `kubectl logs <pod-name> --previous` |
| Container errors | `kubectl logs <pod-name> -c <container-name>` |
| Image pull issues | `kubectl describe pod <pod-name>` |
| Networking issues | `kubectl exec <pod-name> -- curl <url>` |
| Configuration problems | `kubectl get pod <pod-name> -o yaml` |
| Probe failures | `kubectl describe pod <pod-name>` |
| Check env vars | `kubectl exec <pod-name> -- env` |
| View events | `kubectl get events --sort-by='.lastTimestamp'` |
| Resource usage | `kubectl top pods` |
| Check ConfigMap | `kubectl get configmap <name> -o yaml` |
| Check Secret | `kubectl get secret <name> -o yaml` |

---

## Related Exercises

- **Pods:** `exercises/pods/`
- **Deployments:** `exercises/deployments/`
- **ConfigMaps/Secrets:** `exercises/configmaps-secrets/`
- **Probes:** `exercises/probes/`
- **Multi-container:** `exercises/multi-container/`
- **Jobs/CronJobs:** `exercises/jobs-cronjobs/`
- **API Deployment:** `exercises/api-deployment/`
- **Debugging:** `exercises/debugging/`
