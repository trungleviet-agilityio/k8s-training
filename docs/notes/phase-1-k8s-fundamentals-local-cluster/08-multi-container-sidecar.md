# Day 08 — Multi-container Pods: Sidecar

**Goal:** Use a sidecar container sharing a volume with the main container.

---

## What is a Sidecar Pattern?

A **sidecar** is a helper container that runs alongside the main container in the same Pod. Common use cases:
- Log aggregation
- Monitoring/metrics collection
- Proxy/network helper
- File watchers

---

## Sidecar Pod Manifest

**File:** `exercises/multi-container/sidecar-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
  labels:
    app: sidecar-demo
spec:
  containers:
  # Main container - writes logs to shared volume
  - name: main-container
    image: busybox:1.36
    command: ["sh", "-c"]
    args:
    - |
      counter=0
      while true; do
        echo "$(date): Main container log entry $counter" >> /var/log/app.log
        counter=$((counter+1))
        sleep 5
      done
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  # Sidecar container - tails the log file
  - name: sidecar-container
    image: busybox:1.36
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  volumes:
  - name: shared-logs
    emptyDir: {}
```

**Apply:**
```bash
kubectl apply -f exercises/multi-container/sidecar-pod.yaml
```

**Verify:**
```bash
kubectl get pod sidecar-pod
```

**Expected output:**
```
NAME          READY   STATUS    RESTARTS   AGE
sidecar-pod   2/2     Running   0          20s
```

**Key indicator:** `2/2` means both containers are ready.

---

## How Volumes are Shared Between Containers

### Volume Definition

Volumes are defined at the **Pod level** (not container level):

```yaml
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
```

### Volume Mounts

Each container **mounts** the same volume at its desired path:

```yaml
containers:
- name: main-container
  volumeMounts:
  - name: shared-logs      # Reference to Pod-level volume
    mountPath: /var/log    # Where to mount in this container

- name: sidecar-container
  volumeMounts:
  - name: shared-logs      # Same volume reference
    mountPath: /var/log    # Same or different mount path
```

### Key Points

1. **Volume is Pod-level:** Defined once in `spec.volumes`
2. **Each container mounts it:** Each container references the volume in `volumeMounts`
3. **Same underlying storage:** All containers see the same files/data
4. **Independent mount paths:** Containers can mount at different paths if needed

### emptyDir Volume

```yaml
volumes:
- name: shared-logs
  emptyDir: {}
```

**Characteristics:**
- Created when Pod starts
- Deleted when Pod terminates
- Stored on node's disk (or in memory with `emptyDir: { medium: Memory }`)
- Shared between all containers in the Pod
- Perfect for temporary shared storage

---

## Observing Logs from Both Containers

### Main Container Logs

```bash
kubectl logs sidecar-pod -c main-container --tail=5
```

**Output:**
```
(empty - main container writes to file, not stdout)
```

**Note:** Main container writes to file, so logs may be empty.

### Sidecar Container Logs

```bash
kubectl logs sidecar-pod -c sidecar-container --tail=10
```

**Output:**
```
Mon Nov 24 09:20:39 UTC 2025: Main container log entry 0
Mon Nov 24 09:20:44 UTC 2025: Main container log entry 1
Mon Nov 24 09:20:49 UTC 2025: Main container log entry 2
Mon Nov 24 09:20:54 UTC 2025: Main container log entry 3
Mon Nov 24 09:20:59 UTC 2025: Main container log entry 4
```

**Key observation:** Sidecar shows logs written by main container!

### Verify Shared Volume

**From main container:**
```bash
kubectl exec sidecar-pod -c main-container -- ls -la /var/log/
kubectl exec sidecar-pod -c main-container -- cat /var/log/app.log | tail -5
```

**From sidecar container:**
```bash
kubectl exec sidecar-pod -c sidecar-container -- ls -la /var/log/
kubectl exec sidecar-pod -c sidecar-container -- cat /var/log/app.log | tail -5
```

**Result:** Both containers see the same file with same content!

---

## Multi-container Pod Characteristics

### Shared Resources

Containers in the same Pod share:
- **Network:** Same IP, can communicate via `localhost`
- **Storage:** Volumes mounted at Pod level
- **IPC:** Can use inter-process communication

### Independent Resources

Each container has:
- **Filesystem:** Except mounted volumes
- **Processes:** Separate process trees
- **Logs:** Separate log streams

---

## Common Commands

```bash
# Apply sidecar pod
kubectl apply -f exercises/multi-container/sidecar-pod.yaml

# Check pod status (shows container count)
kubectl get pod sidecar-pod

# Describe pod (shows all containers)
kubectl describe pod sidecar-pod

# View logs from specific container
kubectl logs sidecar-pod -c main-container
kubectl logs sidecar-pod -c sidecar-container

# Follow logs from sidecar
kubectl logs sidecar-pod -c sidecar-container -f

# Execute command in specific container
kubectl exec sidecar-pod -c main-container -- ls /var/log
kubectl exec sidecar-pod -c sidecar-container -- cat /var/log/app.log

# View all containers in pod
kubectl get pod sidecar-pod -o jsonpath='{.spec.containers[*].name}'
```

---

## Sidecar Use Cases

### 1. Log Aggregation

**Pattern:** Main app writes logs, sidecar collects and ships them.

```yaml
- name: app
  # Writes logs to /var/log/app.log
- name: log-shipper
  # Reads /var/log/app.log and ships to external system
```

### 2. Monitoring

**Pattern:** Sidecar collects metrics from main container.

```yaml
- name: app
  # Application runs
- name: metrics-collector
  # Collects metrics and exposes them
```

### 3. Proxy/Network Helper

**Pattern:** Sidecar handles network operations.

```yaml
- name: app
  # Application logic
- name: proxy
  # Handles networking, SSL termination, etc.
```

### 4. File Watcher

**Pattern:** Sidecar watches for file changes.

```yaml
- name: app
  # Writes files
- name: file-watcher
  # Watches files and triggers actions
```

---

## Volume Types for Multi-container Pods

### emptyDir

```yaml
volumes:
- name: shared-data
  emptyDir: {}
```

**Use:** Temporary shared storage, deleted with Pod.

### ConfigMap/Secret

```yaml
volumes:
- name: config
  configMap:
    name: app-config
```

**Use:** Share configuration between containers.

### PersistentVolumeClaim

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

**Use:** Persistent shared storage across Pod restarts.

---

## Key Learnings

1. **Volumes are Pod-level:**
   - Defined once in `spec.volumes`
   - Mounted by each container via `volumeMounts`

2. **Same underlying storage:**
   - All containers see the same files/data
   - Changes by one container visible to others

3. **Independent mount paths:**
   - Containers can mount at different paths
   - Same volume, different locations in each container

4. **emptyDir characteristics:**
   - Created with Pod, deleted with Pod
   - Perfect for temporary shared storage
   - Stored on node disk (or memory)

5. **Sidecar pattern:**
   - Helper container alongside main container
   - Shares resources (network, volumes)
   - Common for logging, monitoring, proxies

---

## Deliverables

**sidecar-pod.yaml** - Multi-container Pod with main and sidecar containers  
**Shared emptyDir volume** - Both containers mount same volume  
**Verified** - Logs observed from both containers, volume sharing confirmed  
**Documentation** - How volumes are shared between containers explained

---

## Summary

- Volumes are defined at Pod level
- Each container mounts the volume independently
- All containers see the same underlying storage
- Sidecar pattern enables helper containers for logging, monitoring, etc.
- emptyDir provides temporary shared storage

**Multi-container Pods enable powerful patterns like sidecars for enhanced functionality.**

