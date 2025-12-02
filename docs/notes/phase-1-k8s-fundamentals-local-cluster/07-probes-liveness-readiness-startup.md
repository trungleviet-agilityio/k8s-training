# Day 07 — Probes: Liveness, Readiness, Startup

**Goal:** Understand when and how to use liveness, readiness, and startup probes.

---

## What are Probes?

Probes are health checks that Kubernetes performs on containers:
- **Readiness Probe:** Is the container ready to accept traffic?
- **Liveness Probe:** Is the container still alive and working?
- **Startup Probe:** Is the container finished starting up?

---

## Deployment with Probes

**File:** `exercises/probes/probed-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probed-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probed-app
  template:
    metadata:
      labels:
        app: probed-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        startupProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 30
```

**Apply:**
```bash
kubectl apply -f exercises/probes/probed-deployment.yaml
```

---

## Difference: Readiness vs Liveness vs Startup

### Readiness Probe

**Purpose:** Determines if container is ready to accept traffic.

**When it runs:**
- After `initialDelaySeconds`
- Every `periodSeconds`
- Continues throughout container lifetime

**What happens when it fails:**
- Pod marked as **Not Ready** (0/1 Ready)
- **Removed from Service endpoints**
- Traffic **not routed** to this pod
- Container **continues running** (not restarted)

**Use case:** Container needs time to initialize (load config, connect to DB, etc.)

### Liveness Probe

**Purpose:** Determines if container is still alive and working.

**When it runs:**
- After `initialDelaySeconds`
- Every `periodSeconds`
- Continues throughout container lifetime

**What happens when it fails:**
- After `failureThreshold` failures
- Container is **killed and restarted**
- Pod shows **RESTARTS** count increases
- New container instance created

**Use case:** Detect deadlocks, infinite loops, or hung processes

### Startup Probe

**Purpose:** Gives slow-starting containers time to initialize.

**When it runs:**
- Starts immediately (or after `initialDelaySeconds`)
- Every `periodSeconds`
- Stops once successful
- **Disables liveness/readiness** until startup succeeds

**What happens when it fails:**
- Container is **killed and restarted** (after `failureThreshold`)
- Liveness/readiness probes **don't run** until startup succeeds

**Use case:** Slow-starting applications (Java apps, databases, etc.)

---

## Probe Configuration Parameters

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5     # Wait 5s before first probe
  periodSeconds: 10          # Probe every 10s
  timeoutSeconds: 3          # Probe times out after 3s
  successThreshold: 1        # 1 success = probe passes
  failureThreshold: 3        # 3 failures = probe fails
```

**Parameters explained:**
- **initialDelaySeconds:** Wait time before first probe
- **periodSeconds:** How often to probe
- **timeoutSeconds:** Probe timeout
- **successThreshold:** Consecutive successes needed
- **failureThreshold:** Consecutive failures before action

---

## What Happens When a Probe Fails?

### Readiness Probe Failure

**Observed behavior:**
```bash
kubectl get pods
```

**Output:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
probed-deployment-xxx                0/1     Running   0          30s
```

**Key indicators:**
- `READY: 0/1` - Pod not ready
- `STATUS: Running` - Container still running
- `RESTARTS: 0` - No restart

**Events:**
```bash
kubectl describe pod <pod-name>
```

**Shows:**
```
Events:
  Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
```

**Impact:**
- Pod removed from Service endpoints
- No traffic routed to this pod
- Container continues running

### Liveness Probe Failure

**Observed behavior:**
```bash
kubectl get pods
```

**Output:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
probed-deployment-xxx                0/1     Running   2          1m
```

**Key indicators:**
- `RESTARTS: 2` - Container restarted multiple times
- `STATUS: Running` - New container instance running

**Events:**
```bash
kubectl describe pod <pod-name>
```

**Shows:**
```
Events:
  Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    Container nginx failed liveness probe, will be restarted
  Normal   Created    Created container nginx
  Normal   Started    Started container nginx
```

**Impact:**
- Container killed and restarted
- Pod may be in CrashLoopBackOff if restarts keep failing
- New container instance created

### Startup Probe Failure

**Observed behavior:**
- Similar to liveness (container restarted)
- Liveness/readiness probes don't run until startup succeeds
- Useful for slow-starting apps

---

## Testing Probe Failures

### Create Deployment with Wrong Paths

**File:** `exercises/probes/probed-deployment-fail.yaml`

```yaml
readinessProbe:
  httpGet:
    path: /ready  # Wrong path - doesn't exist
    port: 80
livenessProbe:
  httpGet:
    path: /healthz  # Wrong path - doesn't exist
    port: 80
```

**Apply and observe:**
```bash
kubectl apply -f exercises/probes/probed-deployment-fail.yaml
kubectl get pods -w
```

**Observed results:**
- Readiness fails → Pod shows `0/1 Ready`
- Liveness fails → Container restarts (RESTARTS increases)
- Events show probe failures

---

## Probe Types

### HTTP GET Probe

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: value
```

### TCP Socket Probe

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
```

### Exec Command Probe

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/ready
```

---

## Common Commands

```bash
# Apply deployment with probes
kubectl apply -f exercises/probes/probed-deployment.yaml

# Check pod status
kubectl get pods -l app=probed-app

# Watch pods (see probe effects)
kubectl get pods -l app=probed-app -w

# Describe pod (see probe configuration and events)
kubectl describe pod <pod-name>

# Check probe status in describe output
kubectl describe pod <pod-name> | grep -A 5 "Readiness\|Liveness\|Startup"

# View events (probe failures)
kubectl describe pod <pod-name> | grep Events -A 20

# Check if pod is ready
kubectl get pods -l app=probed-app -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}'
```

---

## Best Practices

1. **Always use readiness probes** for services that receive traffic
2. **Use liveness probes** for applications that can get stuck
3. **Use startup probes** for slow-starting applications
4. **Set appropriate delays:**
   - Readiness: Short delay (5-10s)
   - Liveness: Longer delay (30s+) to avoid premature restarts
   - Startup: Based on actual startup time
5. **Choose correct paths:** Ensure probe endpoints exist
6. **Set reasonable thresholds:** Balance between responsiveness and stability

---

## Key Learnings

1. **Readiness Probe:**
   - Removes pod from Service when failing
   - Container keeps running
   - Prevents traffic to unhealthy pods

2. **Liveness Probe:**
   - Restarts container when failing
   - Kills and recreates container
   - Recovers from deadlocks/hangs

3. **Startup Probe:**
   - Gives slow apps time to start
   - Disables other probes until successful
   - Prevents premature restarts

4. **Probe Failures:**
   - Readiness: Pod not ready, no traffic
   - Liveness: Container restart
   - Both: Pod unhealthy, may restart

---

## Deliverables

**probed-deployment.yaml** - Deployment with readiness, liveness, and startup probes  
**Verified** - Probes working correctly, failures tested and observed  
**Documentation** - Differences between probe types and failure behavior explained

---

## Summary

- **Readiness:** Controls traffic routing (removes from Service)
- **Liveness:** Controls container lifecycle (restarts on failure)
- **Startup:** Gives slow apps time to initialize
- **Failures:** Readiness = no traffic, Liveness = restart, Startup = restart before other probes run

**Probes are essential for reliable Kubernetes applications.**
