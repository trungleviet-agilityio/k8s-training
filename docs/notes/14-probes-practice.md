# M2-02 Probes Practice

**Goal:** Deep practice configuring, fixing, and debugging probes.

---

## When to Use Each Probe

### Readiness Probe
**Purpose:** Determines if a pod is ready to receive traffic.

**When to use:**
- App needs time to initialize (load config, connect to DB)
- App can become temporarily unavailable but will recover
- You want to prevent sending traffic to unready pods

**Failure behavior:**
- Pod marked as `Not Ready` (READY 0/1)
- Pod continues running (not restarted)
- If a Service exists, pod is removed from Service endpoints

### Liveness Probe
**Purpose:** Determines if a pod is alive and should be restarted.

**When to use:**
- App can deadlock or hang
- App needs restart when unhealthy
- Detect when container is running but app is broken

**Failure behavior:**
- Pod is **restarted** by kubelet
- Pod status may show `CrashLoopBackOff` if repeatedly failing
- If a Service exists, pod is removed from Service endpoints during restart

### Startup Probe
**Purpose:** Gives slow-starting containers extra time before other probes start.

**When to use:**
- App takes a long time to start (minutes)
- You want to disable readiness/liveness until startup completes
- Prevents premature restarts during initialization

**Failure behavior:**
- If startup probe fails, readiness/liveness probes are disabled
- After startup succeeds, other probes take over
- If startup never succeeds, pod remains in `Not Ready` state

---

## Probe Configuration

### HTTP Probe
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3
```

### TCP Probe
```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Exec Probe
```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - pgrep -f myapp
  initialDelaySeconds: 30
  periodSeconds: 10
```

### Key Parameters
- **initialDelaySeconds**: Wait time before first probe
- **periodSeconds**: How often to probe
- **timeoutSeconds**: Probe timeout
- **successThreshold**: Consecutive successes needed (default: 1)
- **failureThreshold**: Consecutive failures before marking unhealthy (default: 3)

---

## Debugging Probe Failures

### 1. Check Pod Status
```bash
kubectl get pods
# Look for: Not Ready, CrashLoopBackOff, Restarting
```

### 2. Describe Pod
```bash
kubectl describe pod <pod-name>
```

**Look for:**
- **Events**: Probe failures, restarts
- **Conditions**: Ready status, reason
- **Readiness/Liveness**: Probe configuration and last result

**Example output:**
```
Readiness:      http-get http://:80/ready delay=5s timeout=1s period=10s #success=1 #failure=3
Conditions:
  Type              Status
  Ready             False
  ContainersReady   False
Events:
  Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
```

### 3. Check Logs
```bash
kubectl logs <pod-name>
# Check if app is actually running
# Look for errors during startup
```

### 4. Exec into Pod
```bash
kubectl exec -it <pod-name> -- curl http://localhost:80/ready
# Test the probe endpoint manually
```

### 5. Check Service Endpoints (if Service exists)
```bash
kubectl get endpoints <service-name>
# If pods are Not Ready, they won't appear in endpoints
# Note: This practice doesn't create Services, only Deployments
```

---

## Common Probe Issues

### Issue: Pod Running but Not Ready
**Symptoms:**
- Pod status: `Running` but `READY 0/1`
- Readiness probe failing
- If a Service exists, pod won't appear in Service endpoints

**Causes:**
- Wrong probe path (404)
- Wrong port
- App not fully initialized
- Probe timeout too short

**Fix:**
```bash
kubectl describe pod <pod-name> | grep -A 5 "Readiness:"
# Check the probe path and port
# Verify app responds to that endpoint
```

### Issue: CrashLoopBackOff
**Symptoms:**
- Pod status: `CrashLoopBackOff`
- High restart count
- Liveness probe failing

**Causes:**
- Wrong liveness probe path
- App crashes on startup
- Probe too aggressive (fails before app starts)

**Fix:**
```bash
kubectl describe pod <pod-name> | grep -A 5 "Liveness:"
kubectl logs <pod-name> --previous  # Check previous container logs
# Increase initialDelaySeconds
# Fix probe path/port
```

### Issue: Pod Never Becomes Ready
**Symptoms:**
- Pod stays in `ContainerCreating` or `Not Ready`
- Startup probe failing

**Causes:**
- Startup probe path wrong
- App takes longer than `failureThreshold * periodSeconds`
- Startup probe too strict

**Fix:**
```bash
kubectl describe pod <pod-name> | grep -A 5 "Startup:"
# Increase failureThreshold
# Fix probe path
# Check app startup time
```

---

## Practice Scenarios

### Scenario 1: Broken Readiness Probe
**File:** `broken-readiness.yaml`

**Issue:**
- Readiness probe checks `/ready` (doesn't exist)
- Pods are `Running` but `Not Ready` (READY 0/1)

**Observe:**
```bash
kubectl get pods -l app=broken-readiness
# Shows: READY 0/1, STATUS Running

kubectl describe pod -l app=broken-readiness
# Shows: Readiness probe failed: HTTP probe failed with statuscode: 404
# Shows: Ready: False, ContainersReady: False
```

**Fix:**
- Change readiness probe path to `/` (or create `/ready` endpoint)

### Scenario 2: Broken Liveness Probe
**File:** `broken-liveness.yaml`

**Issue:**
- Liveness probe checks `/healthz` (doesn't exist)
- Probe fails after `initialDelaySeconds` (10s)
- Container gets restarted by kubelet

**Observe:**
```bash
kubectl get pods -l app=broken-liveness
# Shows: STATUS Running, READY 0/1, RESTARTS 1 (or higher)

kubectl describe pod -l app=broken-liveness
# Shows:
# - Liveness: http-get http://:80/healthz delay=10s ...
# - Events: "Liveness probe failed: HTTP probe failed with statuscode: 404"
# - Events: "Container web-server failed liveness probe, will be restarted"
# - Restart Count: 1 (increases with each restart)
```

**Note:** Pod status shows `Running` (not `CrashLoopBackOff`) because the container restarts quickly. If restarts happen too frequently, you may see `CrashLoopBackOff`.

**Fix:**
- Change liveness probe path to `/` (or create `/healthz` endpoint)
- Increase `initialDelaySeconds` if app needs more startup time

### Scenario 3: Working Deployment
**File:** `working-deployment.yaml`

**Configuration:**
- All probes check `/` (exists)
- Proper timing configured
- Pods become `Ready` and stay healthy

**Observe:**
```bash
kubectl get pods -l app=probed-app
# Shows: READY 1/1, STATUS Running

kubectl describe pod -l app=probed-app
# Shows: Ready: True, ContainersReady: True
# Shows: All probes passing
```

---

## Best Practices

1. **Start with readiness probe** - Most apps need this
2. **Add liveness probe carefully** - Only if app can deadlock
3. **Use startup probe for slow apps** - Prevents premature restarts
4. **Set appropriate delays** - `initialDelaySeconds` should account for startup time
5. **Use different paths** - `/ready` for readiness, `/healthz` for liveness
6. **Monitor probe metrics** - Track probe success/failure rates
7. **Test probes manually** - `curl` the endpoints before deploying

---

## Quick Debugging Commands

```bash
# Check pod status
kubectl get pods -o wide

# Describe pod (shows probe details)
kubectl describe pod <pod-name>

# Check probe endpoint manually
kubectl exec -it <pod-name> -- curl http://localhost:<port>/<path>

# View events
kubectl get events --sort-by='.lastTimestamp'

# Check service endpoints (if Service exists)
kubectl get endpoints <service-name>

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container
```

---

## Files

- `working-deployment.yaml` - Deployment with all three probes configured correctly
- `broken-readiness.yaml` - Demonstrates readiness probe failure
- `broken-liveness.yaml` - Demonstrates liveness probe failure
- `broken-startup.yaml` - Demonstrates startup probe failure
