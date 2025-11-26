# Debugging: describe, logs, exec

**Goal:** Practice standard debugging patterns using kubectl commands.

---

## Scenario

A Deployment was created but pods are not becoming Ready. We need to debug and fix the issues.

---

## Step 1: Observe Symptoms

**Check pod status:**
```bash
kubectl get pods -l app=broken-app
```

**Output:**
```
NAME                                 READY   STATUS         RESTARTS   AGE
broken-deployment-5f66bc4fcd-hfgkg   0/1     ErrImagePull   0          12s
broken-deployment-5f66bc4fcd-nppdg   0/1     ErrImagePull   0          12s
```

**Symptoms:**
- Pods in `ErrImagePull` status
- `READY` shows `0/1` (container not ready)

---

## Step 2: Debug with kubectl describe

**Describe pod to see detailed information:**
```bash
kubectl describe pod -l app=broken-app
```

**Key findings from Events section:**
```
Events:
  Warning  Failed      kubelet    Failed to pull image "nginx:invalid-tag": 
                         manifest for nginx:invalid-tag not found
  Warning  Failed      kubelet    Error: ErrImagePull
```

**Container information:**
```
Containers:
  web-server:
    Image:          nginx:invalid-tag
    Port:           8080/TCP
    State:          Waiting
      Reason:       ErrImagePull
```

**Root causes identified:**
1. Invalid image tag: `nginx:invalid-tag` doesn't exist
2. Wrong port: `8080` but nginx uses `80`
3. Hardcoded `DATABASE_URL` (security issue)

---

## Step 3: Debug with kubectl logs

**Try to get logs:**
```bash
kubectl logs -l app=broken-app --tail=10
```

**Output:**
```
Error: container "web-server" is waiting to start: trying and failing to pull image
No logs available (pod not running)
```

**Finding:** Logs unavailable because container never started. Use `describe` for image pull errors.

---

## Step 4: Debug with kubectl exec

**Try to exec into pod:**
```bash
POD_NAME=$(kubectl get pod -l app=broken-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- echo "test"
```

**Output:**
```
Error: container web-server is not running
```

**Finding:** Cannot exec because container is not running. Use `describe` first for non-running pods.

---

## Step 5: Fix the Deployment

**Fixed manifest:** `exercises/debugging/fixed-deployment.yaml`

**Changes:**
- Image: `nginx:invalid-tag` → `nginx:1.25`
- Port: `8080` → `80`
- Removed hardcoded `DATABASE_URL`

**Apply fix:**
```bash
kubectl apply -f exercises/debugging/fixed-deployment.yaml
```

---

## Step 6: Verify the Fix

**Check pod status:**
```bash
kubectl get pods -l app=broken-app
```

**Output:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
broken-deployment-644d777cb9-4wvg2   1/1     Running   0          19s
broken-deployment-644d777cb9-qsjrp   1/1     Running   0          18s
```

**Verify with logs:**
```bash
POD_NAME=$(kubectl get pod -l app=broken-app -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME --tail=5
```

**Verify with exec:**
```bash
kubectl exec $POD_NAME -- nginx -v
kubectl exec $POD_NAME -- curl -s localhost:80 | head -3
```

**Result:** Pods are now Running and Ready ✅

---

## kubectl Commands Reference

### kubectl describe

**Purpose:** Get detailed information about resources and events.

**Common commands:**
```bash
# Describe pod
kubectl describe pod <pod-name>

# Describe pods by label
kubectl describe pod -l app=my-app

# Focus on Events section
kubectl describe pod <pod-name> | grep -A 20 "Events:"

# Focus on Containers section
kubectl describe pod <pod-name> | grep -A 10 "Containers:"
```

**Use when:**
- Pod not starting
- Image pull errors
- Configuration issues
- Need to see events and state transitions

**Key sections:**
- **Events:** Shows what happened (scheduling, pulling, errors)
- **Containers:** Shows container state, image, ports, env vars
- **Conditions:** Shows pod conditions (Ready, PodScheduled)

---

### kubectl logs

**Purpose:** View container logs from running containers.

**Common commands:**
```bash
# Get logs from pod
kubectl logs <pod-name>

# Get logs from specific container
kubectl logs <pod-name> -c <container-name>

# Follow logs (like tail -f)
kubectl logs -f <pod-name>

# Get logs from pods by label
kubectl logs -l app=my-app

# Get last N lines
kubectl logs <pod-name> --tail=50

# Get logs from previous container instance (if crashed)
kubectl logs <pod-name> --previous
```

**Use when:**
- Container is running but having issues
- Application errors
- Runtime problems
- Need to see application output

**Limitations:**
- Only works for running containers
- Cannot get logs if container never started

---

### kubectl exec

**Purpose:** Execute commands inside running containers.

**Common commands:**
```bash
# Execute command in pod
kubectl exec <pod-name> -- <command>

# Interactive shell
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash

# Execute in specific container
kubectl exec <pod-name> -c <container-name> -- <command>

# Check environment variables
kubectl exec <pod-name> -- env

# Test connectivity
kubectl exec <pod-name> -- curl http://localhost:80
kubectl exec <pod-name> -- wget -q -O- http://localhost:80

# Check processes
kubectl exec <pod-name> -- ps aux
```

**Use when:**
- Need to inspect running container
- Test connectivity
- Check file system
- Inspect environment variables

**Limitations:**
- Only works for running containers
- Container must have the command available

---

## Debugging Workflow

### Step-by-step process:

1. **Check pod status**
   ```bash
   kubectl get pods -o wide
   ```

2. **Examine pod events**
   ```bash
   kubectl describe pod <pod-name>
   ```

3. **Review container logs** (if running)
   ```bash
   kubectl logs <pod-name>
   kubectl logs <pod-name> --previous  # If crashed
   ```

4. **Inspect running container** (if running)
   ```bash
   kubectl exec -it <pod-name> -- /bin/sh
   ```

---

## Common Pod Issues

### ErrImagePull / ImagePullBackOff

**Symptoms:** Pod can't pull the image

**Debug:**
```bash
kubectl describe pod <pod-name> | grep -A 10 "Events:"
```

**Common causes:**
- Invalid image tag
- Registry authentication issues
- Network connectivity problems

**Solution:** Fix image tag or check registry access

---

### CrashLoopBackOff

**Symptoms:** Container keeps restarting

**Debug:**
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

**Common causes:**
- Application errors
- Missing configuration
- Wrong command/args

**Solution:** Check logs and fix application/config

---

### Pending

**Symptoms:** Pod not scheduled

**Debug:**
```bash
kubectl describe pod <pod-name>
kubectl get nodes
```

**Common causes:**
- Insufficient resources
- Node selector mismatch
- Taints/tolerations

**Solution:** Check resources and scheduling constraints

---

### Not Ready

**Symptoms:** Running but `READY 0/1`

**Debug:**
```bash
kubectl describe pod <pod-name>
kubectl exec <pod-name> -- curl http://localhost:8080/health
```

**Common causes:**
- Readiness probe failing
- Wrong port in probe
- Application not responding

**Solution:** Fix readiness probe or application

---

## Pod Status Phases

- **Pending:** Pod accepted but containers not running
- **Running:** Pod bound to node, all containers running
- **Succeeded:** All containers terminated successfully
- **Failed:** At least one container terminated with failure
- **Unknown:** Pod state cannot be obtained

**Filter by phase:**
```bash
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=status.phase=Pending
kubectl get pods --field-selector=status.phase=Failed
```

---

## Quick Reference

| Problem | Command |
|---------|---------|
| Pod not starting | `kubectl describe pod <pod-name>` |
| Pod crashing | `kubectl logs <pod-name> --previous` |
| Container errors | `kubectl logs <pod-name> -c <container-name>` |
| Image pull issues | `kubectl describe pod <pod-name>` |
| Networking issues | `kubectl exec <pod-name> -- curl <url>` |
| Configuration problems | `kubectl get pod <pod-name> -o yaml` |

---

## Summary

**Symptoms:**
- Pods in `ErrImagePull` status
- Containers not starting

**Root cause:**
- Invalid image tag `nginx:invalid-tag`
- Wrong container port `8080` (should be `80`)
- Hardcoded sensitive data

**Commands used:**
- `kubectl get pods` - Quick status check
- `kubectl describe pod` - Detailed information and events
- `kubectl logs` - Container logs (not available for non-running pods)
- `kubectl exec` - Interactive debugging (not available for non-running pods)

**Final fix:**
- Changed image to `nginx:1.25`
- Changed port to `80`
- Removed hardcoded `DATABASE_URL`

**Result:** Pods are now Running and Ready ✅

---

## Deliverables

**Broken manifest:** `exercises/debugging/broken-deployment.yaml`
- Invalid image tag: `nginx:invalid-tag`
- Wrong port: `8080` instead of `80`
- Hardcoded `DATABASE_URL`

**Fixed manifest:** `exercises/debugging/fixed-deployment.yaml`
- Valid image: `nginx:1.25`
- Correct port: `80`
- Removed hardcoded `DATABASE_URL`

**Documentation:** This file
- Symptoms documented
- Root cause identified
- Commands used explained
- Final fix applied and verified
