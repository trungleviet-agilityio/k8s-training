# Multi-container Pods

## Design Patterns

### Multi-container Pods Design Patterns

**1. Co-located Containers**
- Two containers in `containers` array
- No startup order guarantee (both start together)
- Both run throughout pod lifecycle
- Use when: Services dependent on each other, no startup order needed

```yaml
containers:
- name: container1
  image: app1:latest
- name: container2
  image: app2:latest
```

**2. Init Containers**
- Defined in `initContainers` section
- Run sequentially before main containers
- Stop after completion
- Use when: Initialization steps needed (wait for DB, API checks)

```yaml
initContainers:
- name: wait-for-db
  image: busybox:1.36
  command: ['sh', '-c', 'until nc -z db 5432; do sleep 1; done']
containers:
- name: main-app
  image: app:latest
```

**3. Sidecar Containers**
- Use `initContainers` with `restartPolicy: Always` (via pod spec)
- Start before main app, continue running throughout lifecycle
- Stop after main app ends
- Use when: Log shippers, monitoring agents (capture startup/termination logs)

```yaml
initContainers:
- name: log-shipper
  image: filebeat:latest
  # restartPolicy: Always set at pod level
containers:
- name: main-app
  image: app:latest
```

**Key Differences:**

| Pattern | Startup Order | Lifecycle | Use Case |
|---------|---------------|-----------|----------|
| **Co-located** | No order | Both run throughout | Dependent services, no order needed |
| **Init** | Sequential, before main | Stop after completion | Initialization (wait for DB, API) |
| **Sidecar** | Before main app | Continue running | Log shippers, monitoring |

**Reference:** [Kubernetes Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

### Sidecar Logging Pattern (Issue #18)

**Why sidecar was used:**
- Main application writes logs to a shared volume
- Sidecar container tails and processes logs (ships to stdout or external system)
- Separates application logic from logging infrastructure
- Allows log shipping without modifying main application

**How volumes are shared:**
```yaml
# Volume defined at Pod level (not container level)
volumes:
- name: shared-logs
  emptyDir: {}  # Temporary storage, deleted when Pod terminates

# Each container mounts the same volume
containers:
- name: main-container
  volumeMounts:
  - name: shared-logs
    mountPath: /var/log
- name: sidecar-container
  volumeMounts:
  - name: shared-logs  # Same volume name
    mountPath: /var/log  # Same or different path
```

**Key points:**
- Volume defined once at Pod level in `spec.volumes`
- Each container references it in `volumeMounts`
- All containers see the same underlying storage
- Changes by one container are visible to others
- `emptyDir`: Created when Pod starts, deleted when Pod terminates

**Verification commands:**
```bash
# Check pod status (should show 2/2 containers ready)
kubectl get pod sidecar-pod

# View logs from main container
kubectl logs sidecar-pod -c main-container

# View logs from sidecar container (shows tailed logs)
kubectl logs sidecar-pod -c sidecar-container -f

# Verify shared volume access
kubectl exec sidecar-pod -c main-container -- ls -la /var/log
kubectl exec sidecar-pod -c sidecar-container -- cat /var/log/app.log
```

**Example:** `exercises/multi-container/sidecar-pod.yaml`

## Common Use Cases

### Multi-container Patterns

**Sidecar Pattern:**
- Log aggregation (main app writes, sidecar ships)
- Monitoring (sidecar collects metrics)
- Proxy/network helper

**Key points:**
- Containers share network (localhost) and volumes
- Both start/stop together
- Use when containers need tight coupling

## Essential Commands

### Multi-container Pods

```bash
# Multi-container pods
kubectl logs <pod-name> -c <container-name>
kubectl exec <pod-name> -c <container-name> -- <command>
```

## Common Issues & Solutions

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
