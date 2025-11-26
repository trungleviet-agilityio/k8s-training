# M2-03 Init Containers Practice

**Goal:** Master the use and behavior of init containers.

---

## What are Init Containers?

Init containers are specialized containers that run **before** the main application containers in a Pod. They are used for setup, verification, and preparation tasks.

**Key characteristics:**
- Run **sequentially** (one after another)
- Must **complete successfully** (exit code 0) before main containers start
- Share volumes with main containers
- Can access the same network namespace
- If any init container fails, the Pod is not started

---

## Ordering Behavior

### Sequential Execution
Init containers run **one at a time**, in the order they are defined in the manifest.

```yaml
initContainers:
- name: init-1  # Runs first
- name: init-2  # Runs after init-1 completes
- name: init-3  # Runs after init-2 completes
containers:
- name: main    # Runs after ALL init containers succeed
```

**Example:**
```yaml
initContainers:
- name: wait-dns
  # ... runs first
- name: prepare-data
  # ... runs second (after wait-dns succeeds)
- name: verify-setup
  # ... runs third (after prepare-data succeeds)
containers:
- name: main-app
  # ... runs last (after all init containers succeed)
```

### Parallel Execution
Init containers **cannot** run in parallel. Each must complete before the next starts.

---

## Exit Codes

### Success (Exit Code 0)
- Init container completes successfully
- Next init container (or main container) can start
- Pod continues to the next phase

### Failure (Non-Zero Exit Code)
- Init container fails
- Pod status shows `Init:Error` or `Init:CrashLoopBackOff`
- Main containers **never start**
- Pod is restarted (if `restartPolicy` allows)

**Common exit codes:**
- `0` - Success
- `1` - General error
- `2` - Misuse of shell command
- `126` - Command cannot execute
- `127` - Command not found
- `128` - Invalid exit argument

---

## Common Patterns

### Pattern 1: Wait for DNS
**Use case:** Ensure DNS is available before starting the app.

```yaml
initContainers:
- name: wait-for-dns
  image: busybox:1.36
  command: ['sh', '-c']
  args:
  - |
    until nslookup kubernetes.default.svc.cluster.local; do
      echo "DNS not ready, waiting..."
      sleep 2
    done
    echo "DNS is ready!"
```

**File:** `dns-wait-pod.yaml`

---

### Pattern 2: Prepare Shared Volume
**Use case:** Initialize configuration files or data that main containers need.

```yaml
initContainers:
- name: prepare-config
  image: busybox:1.36
  command: ['sh', '-c']
  args:
  - |
    echo "APP_MODE=production" > /shared/config.env
    echo "LOG_LEVEL=info" >> /shared/config.env
  volumeMounts:
  - name: shared-data
    mountPath: /shared
containers:
- name: main-app
  image: busybox:1.36
  command: ['sh', '-c', 'cat /shared/config.env']
  volumeMounts:
  - name: shared-data
    mountPath: /shared
volumes:
- name: shared-data
  emptyDir: {}
```

**File:** `shared-volume-pod.yaml`

**Key points:**
- Init container writes to `/shared/config.env`
- Main container reads from `/shared/config.env`
- Both mount the same `emptyDir` volume
- Data persists from init to main container

---

### Pattern 3: Pre-check / Verification
**Use case:** Verify prerequisites before starting the app.

```yaml
initContainers:
- name: verify-config
  image: busybox:1.36
  command: ['sh', '-c']
  args:
  - |
    if [ ! -d /config ]; then
      echo "ERROR: /config directory does not exist"
      exit 1
    fi
    echo "Pre-check passed"
```

**File:** `precheck-pod.yaml`

**Common checks:**
- Directory exists
- File exists
- Network connectivity
- External service availability
- Required environment variables

---

### Pattern 4: Combined (Multiple Init Containers)
**Use case:** Chain multiple setup tasks.

```yaml
initContainers:
# Step 1: Wait for DNS
- name: wait-dns
  # ... waits for DNS
# Step 2: Prepare data
- name: prepare-data
  # ... writes to shared volume
# Step 3: Verify setup
- name: verify-setup
  # ... checks that data exists
containers:
- name: main-app
  # ... uses prepared data
```

**File:** `combined-pod.yaml`

---

## Debugging Init Containers

### 1. Check Pod Status
```bash
kubectl get pods
# Look for: Init:0/3, Init:Error, Init:CrashLoopBackOff
```

**Status meanings:**
- `Init:0/3` - First init container running (0 of 3 completed)
- `Init:1/3` - Second init container running (1 of 3 completed)
- `Init:Error` - An init container failed
- `Init:CrashLoopBackOff` - Init container keeps failing

### 2. Describe Pod
```bash
kubectl describe pod <pod-name>
```

**Look for:**
- **Init Containers:** Status of each init container
- **State:** Running, Terminated, Waiting
- **Exit Code:** 0 (success) or non-zero (failure)
- **Reason:** Completed, Error, CrashLoopBackOff
- **Events:** Error messages, restart reasons

**Example output:**
```
Init Containers:
  wait-dns:
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
  prepare-data:
    State:          Terminated
      Reason:       Error
      Exit Code:    1
```

### 3. View Init Container Logs
```bash
# View logs from a specific init container
kubectl logs <pod-name> -c <init-container-name>

# Example
kubectl logs my-pod -c wait-for-dns
```

**Note:** Init container logs are only available if the container has run. If the Pod is stuck in `Init:0/1`, the first init container may still be running.

### 4. Check Exit Codes
```bash
kubectl describe pod <pod-name> | grep -A 10 "Init Containers:"
# Look for "Exit Code:"
```

---

## Common Errors

### Error 1: Init Container Fails
**Symptoms:**
- Pod status: `Init:Error` or `Init:CrashLoopBackOff`
- Main containers never start
- Pod keeps restarting

**Causes:**
- Wrong command in init container
- Missing file or directory
- Network connectivity issue
- Non-zero exit code

**Fix:**
```bash
# Check init container logs
kubectl logs <pod-name> -c <init-container-name>

# Check exit code
kubectl describe pod <pod-name> | grep "Exit Code:"

# Fix the command or add error handling
```

**Example:**
```yaml
# Broken
command: ['cat', '/nonexistent/file.txt']

# Fixed
command: ['sh', '-c', 'if [ ! -f /file.txt ]; then exit 1; fi']
```

---

### Error 2: Init Container Hangs
**Symptoms:**
- Pod status: `Init:0/1` (stuck)
- Init container never completes
- No error, just waiting

**Causes:**
- Infinite loop without exit condition
- Waiting for resource that never becomes available
- Blocking operation

**Fix:**
```bash
# Check logs to see what it's waiting for
kubectl logs <pod-name> -c <init-container-name>

# Add timeout or exit condition
```

**Example:**
```yaml
# Broken (infinite wait)
args:
- |
  while true; do
    nslookup myservice
    sleep 1
  done

# Fixed (with timeout)
args:
- |
  timeout=60
  elapsed=0
  until nslookup myservice || [ $elapsed -ge $timeout ]; do
    sleep 2
    elapsed=$((elapsed + 2))
  done
  if [ $elapsed -ge $timeout ]; then
    echo "Timeout waiting for DNS"
    exit 1
  fi
```

---

### Error 3: Volume Not Shared
**Symptoms:**
- Init container writes file
- Main container can't find file
- File not in expected location

**Causes:**
- Volume not mounted in both containers
- Different mount paths
- Volume name mismatch

**Fix:**
```yaml
# Ensure same volume name and mount path
initContainers:
- name: prepare
  volumeMounts:
  - name: shared-data      # Same name
    mountPath: /shared     # Same path
containers:
- name: main
  volumeMounts:
  - name: shared-data      # Same name
    mountPath: /shared     # Same path
volumes:
- name: shared-data        # Same name
  emptyDir: {}
```

---

### Error 4: Wrong Execution Order
**Symptoms:**
- Init container 2 runs before init container 1 completes
- Dependencies not met

**Causes:**
- Misunderstanding of sequential execution
- Missing dependency between init containers

**Fix:**
- Init containers always run sequentially
- Order them correctly in the manifest
- Ensure each init container completes before the next starts

---

## Best Practices

1. **Use init containers for setup only** - Not for long-running processes
2. **Make commands idempotent** - Safe to run multiple times
3. **Add error handling** - Check for files, directories, network
4. **Use appropriate images** - Small, fast images (busybox, alpine)
5. **Set timeouts** - Prevent infinite waits
6. **Log progress** - Help with debugging
7. **Test exit codes** - Ensure commands exit correctly
8. **Share volumes correctly** - Same name and mount path

---

## Quick Debugging Commands

```bash
# Check pod status
kubectl get pods

# Describe pod (shows init container status)
kubectl describe pod <pod-name>

# View init container logs
kubectl logs <pod-name> -c <init-container-name>

# Check all init containers
kubectl get pod <pod-name> -o jsonpath='{.status.initContainerStatuses[*].name}'

# Check init container exit codes
kubectl get pod <pod-name> -o jsonpath='{.status.initContainerStatuses[*].state}'

# View events
kubectl get events --sort-by='.lastTimestamp'
```

---

## Practice Scenarios

### Scenario 1: DNS Wait
**File:** `dns-wait-pod.yaml`

**Behavior:**
- Init container waits for DNS resolution
- Main container starts after DNS is ready

**Test:**
```bash
kubectl apply -f dns-wait-pod.yaml
kubectl get pod dns-wait-pod
kubectl logs dns-wait-pod -c wait-for-dns
```

---

### Scenario 2: Shared Volume
**File:** `shared-volume-pod.yaml`

**Behavior:**
- Init container creates config file in shared volume
- Main container reads the config file

**Test:**
```bash
kubectl apply -f shared-volume-pod.yaml
kubectl logs shared-volume-pod -c prepare-config
kubectl logs shared-volume-pod -c main-app
```

---

### Scenario 3: Pre-check
**File:** `precheck-pod.yaml`

**Behavior:**
- Init container verifies `/config` directory exists
- Main container starts only if pre-check passes

**Test:**
```bash
kubectl apply -f precheck-pod.yaml
kubectl describe pod precheck-pod
```

---

### Scenario 4: Combined
**File:** `combined-pod.yaml`

**Behavior:**
- Three init containers run sequentially:
  1. Wait for DNS
  2. Prepare data file
  3. Verify data file exists
- Main container uses prepared data

**Test:**
```bash
kubectl apply -f combined-pod.yaml
kubectl logs combined-init-pod -c wait-dns
kubectl logs combined-init-pod -c prepare-data
kubectl logs combined-init-pod -c verify-setup
kubectl logs combined-init-pod -c main-app
```

---

### Scenario 5: Broken Init Container
**File:** `broken-init-pod.yaml`

**Behavior:**
- Init container fails (exit code 1)
- Pod status: `Init:Error`
- Main container never starts

**Observe:**
```bash
kubectl apply -f broken-init-pod.yaml
kubectl get pod broken-init-pod
# Shows: Init:Error or Init:CrashLoopBackOff

kubectl describe pod broken-init-pod
# Shows: Exit Code: 1, Reason: Error

kubectl logs broken-init-pod -c failing-init
# Shows error message
```

**Fix:**
- Correct the command in the init container
- Add proper error handling
- Ensure exit code 0 on success

---

## Files

- `dns-wait-pod.yaml` - Init container waits for DNS
- `shared-volume-pod.yaml` - Init container prepares shared volume
- `precheck-pod.yaml` - Init container performs pre-check
- `combined-pod.yaml` - Multiple init containers in sequence
- `broken-init-pod.yaml` - Demonstrates init container failure
