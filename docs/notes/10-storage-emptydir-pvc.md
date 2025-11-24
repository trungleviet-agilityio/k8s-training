# Day 10 — Storage: emptyDir and PVC

**Goal:** Understand ephemeral and simple persistent storage.

---

## What are emptyDir and PVC?

- **emptyDir:** Ephemeral storage created when Pod starts, deleted when Pod terminates
- **PVC (PersistentVolumeClaim):** Request for persistent storage that survives Pod restarts

---

## emptyDir Example

**File:** `exercises/storage/emptydir-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - name: writer-container
    image: busybox:1.36
    command: ["sh", "-c"]
    args:
    - |
      counter=0
      while true; do
        echo "Data entry $counter written at $(date)" >> /shared/data.txt
        counter=$((counter+1))
        sleep 10
      done
    volumeMounts:
    - name: shared-storage
      mountPath: /shared
  - name: reader-container
    image: busybox:1.36
    command: ["sh", "-c", "tail -f /shared/data.txt"]
    volumeMounts:
    - name: shared-storage
      mountPath: /shared
  volumes:
  - name: shared-storage
    emptyDir: {}
```

**Apply:**
```bash
kubectl apply -f exercises/storage/emptydir-pod.yaml
```

**Verify:**
```bash
kubectl get pod emptydir-pod
kubectl exec emptydir-pod -c writer-container -- ls -la /shared/
```

---

## PersistentVolumeClaim (PVC)

**File:** `exercises/storage/pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: standard
```

**Apply:**
```bash
kubectl apply -f exercises/storage/pvc.yaml
```

**Check status:**
```bash
kubectl get pvc
```

**Expected output:**
```
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pvc-b18b2940-a3db-413d-99fb-f5f54ad0195a   100Mi      RWO            standard       4s
```

**Key indicators:**
- `STATUS: Bound` - PVC is bound to a PersistentVolume
- `VOLUME` - Shows the PV name
- `CAPACITY` - Storage size allocated

---

## Pod Using PVC

**File:** `exercises/storage/pvc-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: pvc-container
    image: busybox:1.36
    command: ["sh", "-c"]
    args:
    - |
      echo "Writing data to persistent volume..." > /data/persistent-data.txt
      echo "Data written at $(date)" >> /data/persistent-data.txt
      cat /data/persistent-data.txt
      sleep 3600
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

**Apply:**
```bash
kubectl apply -f exercises/storage/pvc-pod.yaml
```

**Verify data:**
```bash
kubectl exec pvc-pod -- cat /data/persistent-data.txt
kubectl exec pvc-pod -- ls -la /data/
```

---

## Test Data Persistence

### Delete and Recreate Pod

```bash
# Delete pod
kubectl delete pod pvc-pod

# Recreate pod
kubectl apply -f exercises/storage/pvc-pod.yaml

# Wait for pod to be ready
kubectl wait --for=condition=ready pod pvc-pod --timeout=30s

# Verify data still exists
kubectl exec pvc-pod -- cat /data/persistent-data.txt
```

**Result:** Data persists across pod restarts!

---

## Differences: emptyDir vs PVC

### emptyDir

**Characteristics:**
- **Lifecycle:** Created when Pod starts, deleted when Pod terminates
- **Persistence:** Data lost when Pod is deleted
- **Storage:** Stored on node's disk (or in memory)
- **Use case:** Temporary files, cache, shared data between containers in same Pod
- **Performance:** Fast (local storage)
- **Cost:** No additional cost

**When to use:**
- Temporary files
- Cache data
- Sharing files between containers in same Pod
- Data that doesn't need to survive Pod restarts

**Example:**
```yaml
volumes:
- name: temp-storage
  emptyDir: {}
```

### PVC (PersistentVolumeClaim)

**Characteristics:**
- **Lifecycle:** Created independently, survives Pod restarts
- **Persistence:** Data persists across Pod deletions
- **Storage:** Backed by PersistentVolume (network storage, cloud storage, etc.)
- **Use case:** Databases, application data, logs that need persistence
- **Performance:** Depends on storage backend
- **Cost:** May incur storage costs

**When to use:**
- Database files
- Application data that must survive restarts
- Logs that need to be preserved
- User uploads/files

**Example:**
```yaml
volumes:
- name: persistent-storage
  persistentVolumeClaim:
    claimName: my-pvc
```

### Comparison Table

| Feature | emptyDir | PVC |
|---------|----------|-----|
| **Persistence** | No (ephemeral) | Yes (persistent) |
| **Lifecycle** | Tied to Pod | Independent |
| **Data survives Pod deletion** | No | Yes |
| **Storage location** | Node disk/memory | Network/cloud storage |
| **Use case** | Temporary, cache | Databases, persistent data |
| **Performance** | Fast (local) | Varies (network) |
| **Cost** | Free | May cost |

---

## Access Modes

### ReadWriteOnce (RWO)

```yaml
accessModes:
  - ReadWriteOnce
```

**Meaning:** Volume can be mounted as read-write by a single node.

**Use case:** Single Pod access, most common.

### ReadOnlyMany (ROX)

```yaml
accessModes:
  - ReadOnlyMany
```

**Meaning:** Volume can be mounted read-only by many nodes.

**Use case:** Shared read-only data (configs, static files).

### ReadWriteMany (RWX)

```yaml
accessModes:
  - ReadWriteMany
```

**Meaning:** Volume can be mounted as read-write by many nodes.

**Use case:** Shared writable storage (rare, requires NFS or similar).

---

## Storage Classes

**Check available storage classes:**
```bash
kubectl get storageclass
```

**Output:**
```
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate          5d2h
```

**Key points:**
- **PROVISIONER:** How storage is provisioned
- **RECLAIMPOLICY:** What happens when PVC is deleted (Delete/Retain)
- **VOLUMEBINDINGMODE:** When binding happens (Immediate/WaitForFirstConsumer)

---

## Common Commands

```bash
# Create PVC
kubectl apply -f pvc.yaml

# List PVCs
kubectl get pvc

# Describe PVC
kubectl describe pvc my-pvc

# List PersistentVolumes
kubectl get pv

# Create Pod with PVC
kubectl apply -f pvc-pod.yaml

# Verify data in Pod
kubectl exec pvc-pod -- ls -la /data/
kubectl exec pvc-pod -- cat /data/persistent-data.txt

# Delete Pod (data persists)
kubectl delete pod pvc-pod

# Delete PVC (releases storage)
kubectl delete pvc my-pvc

# Check storage classes
kubectl get storageclass
```

---

## Key Learnings

1. **emptyDir:**
   - Ephemeral storage, deleted with Pod
   - Perfect for temporary files and cache
   - Fast, local storage

2. **PVC:**
   - Persistent storage, survives Pod restarts
   - Perfect for databases and persistent data
   - Backed by PersistentVolume

3. **Access Modes:**
   - RWO: Single node read-write
   - ROX: Multiple nodes read-only
   - RWX: Multiple nodes read-write

4. **Storage Classes:**
   - Define storage provisioning behavior
   - Default class used if not specified

5. **Data Persistence:**
   - emptyDir: Lost when Pod deleted
   - PVC: Survives Pod restarts and deletions

---

## Deliverables

**emptydir-pod.yaml** - Pod using emptyDir for shared storage between containers  
**pvc.yaml** - PersistentVolumeClaim manifest  
**pvc-pod.yaml** - Pod using PVC for persistent storage  
**Verified** - Data persistence tested across pod restarts  
**Documentation** - Differences between emptyDir and PVC explained

---

## Summary

- **emptyDir:** Ephemeral storage, deleted with Pod, use for temporary files
- **PVC:** Persistent storage, survives Pod restarts, use for databases/data
- **Access Modes:** RWO (single node), ROX (read-only many), RWX (read-write many)
- **Storage Classes:** Define how storage is provisioned

**Choose emptyDir for temporary data, PVC for persistent data.**
