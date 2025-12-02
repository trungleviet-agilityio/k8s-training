# Pods – Basic Manifests and Debugging

**Goal:** Understand the Pod as the basic deployable unit and be able to write Pod manifests from scratch.

---

## Pod Manifests Created

### 1. nginx-pod.yaml
A simple nginx web server container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
    ports:
    - containerPort: 80
      protocol: TCP
```

### 2. busybox-sleep-pod.yaml
A busybox container running a sleep command for debugging purposes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-pod
  labels:
    app: busybox
    purpose: debug
spec:
  containers:
  - name: busybox-container
    image: busybox:1.36
    command: ["sleep", "3600"]
```

### 3. env-pod.yaml
A pod with environment variables defined.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
  labels:
    app: env-demo
spec:
  containers:
  - name: env-container
    image: busybox:1.36
    command: ["sh", "-c", "env && sleep 3600"]
    env:
    - name: ENVIRONMENT
      value: "development"
    - name: APP_NAME
      value: "kubernetes-learning"
```

---

## Pod Structure

Every Pod manifest follows this structure:

### 1. apiVersion
- **Value:** `v1` for Pods
- **Purpose:** Specifies the Kubernetes API version to use
- **Note:** Different resources use different API versions (e.g., Deployments use `apps/v1`)

### 2. kind
- **Value:** `Pod`
- **Purpose:** Defines the type of Kubernetes resource

### 3. metadata
Contains information about the Pod:
- **name:** Unique identifier for the Pod (required, DNS subdomain format)
- **labels:** Key-value pairs for organizing and selecting Pods
- **namespace:** Optional, defaults to `default`
- **annotations:** Optional metadata (not used for selection)

### 4. spec
Defines the desired state of the Pod:
- **containers:** Array of container definitions
  - **name:** Container name (required)
  - **image:** Container image to use (required)
  - **ports:** Exposed ports (optional, informational)
  - **command:** Override the default entrypoint
  - **env:** Environment variables (array of name/value pairs)

---

## Verification

All pods were successfully created and are running:

```bash
kubectl get pods
```

**Output:**
```
NAME                READY   STATUS    RESTARTS   AGE
busybox-sleep-pod   1/1     Running   0          19s
env-pod             1/1     Running   0          12s
nginx-pod           1/1     Running   0          22s
```

---

## Debugging with kubectl describe

The `kubectl describe pod <pod-name>` command provides detailed information about a Pod.

### Key Information from `kubectl describe`:

#### 1. **Pod Metadata**
- Name, Namespace, Labels, Annotations
- Service Account used

#### 2. **Status Information**
- **Status:** Current state (Running, Pending, Failed, etc.)
- **IP:** Pod's IP address
- **Node:** Which node the Pod is scheduled on
- **Start Time:** When the Pod was created

#### 3. **Container Details**
- **Container ID:** Docker container ID
- **Image:** Exact image used (with digest)
- **Port:** Exposed ports
- **State:** Current container state (Running, Waiting, Terminated)
- **Ready:** Whether the container is ready to serve traffic
- **Restart Count:** Number of times the container has restarted
- **Environment:** Environment variables set in the container

#### 4. **Conditions**
- **Initialized:** All init containers completed
- **Ready:** Pod is ready to accept traffic
- **ContainersReady:** All containers are ready
- **PodScheduled:** Pod has been assigned to a node

#### 5. **Volumes**
- Shows mounted volumes (e.g., service account tokens, ConfigMaps, Secrets)

#### 6. **Events**
- Chronological log of what happened to the Pod
- Shows scheduling, image pulling, container creation, and startup

### Example: nginx-pod describe output

```
Name:             nginx-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.39.254
Start Time:       Wed, 19 Nov 2025 15:11:08 +0700
Labels:           app=nginx
                  tier=frontend
Status:           Running
IP:               10.244.0.4
Containers:
  nginx-container:
    Container ID:   docker://afe14e7e144213361deb2bf21daf245139ae2902cf89b0a088999a306d5ae524
    Image:          nginx:1.25
    Image ID:       docker-pullable://nginx@sha256:a484819eb60211f5299034ac80f6a681b06f89e65866ce91f356ed7c72af059c
    Port:           80/TCP
    State:          Running
      Started:      Wed, 19 Nov 2025 15:11:20 +0700
    Ready:          True
    Restart Count:  0
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  30s   default-scheduler  Successfully assigned default/nginx-pod to minikube
  Normal  Pulling    29s   kubelet            Pulling image "nginx:1.25"
  Normal  Pulled     19s   kubelet            Successfully pulled image "nginx:1.25"
  Normal  Created    19s   kubelet            Created container nginx-container
  Normal  Started    18s   kubelet            Started container nginx-container
```

### Example: env-pod describe output (showing environment variables)

```
Containers:
  env-container:
    Image:         busybox:1.36
    Command:
      sh
      -c
      env && sleep 3600
    Environment:
      ENVIRONMENT:  development
      APP_NAME:     kubernetes-learning
```

---

## Debugging with kubectl logs

The `kubectl logs <pod-name>` command shows the logs from a container.

### Key Observations:

#### 1. **nginx-pod logs**
Shows nginx startup sequence:
- Configuration scripts execution
- Worker processes starting
- Server ready messages

**Example output:**
```
/docker-entrypoint.sh: Configuration complete; ready for start up
2025/11/19 08:11:20 [notice] 1#1: nginx/1.25.5
2025/11/19 08:11:20 [notice] 1#1: start worker processes
```

#### 2. **env-pod logs**
Shows all environment variables available in the container:
- **Custom variables:** `ENVIRONMENT=development`, `APP_NAME=kubernetes-learning`
- **Kubernetes-injected variables:** `KUBERNETES_SERVICE_HOST`, `KUBERNETES_PORT`, etc.

**Example output:**
```
ENVIRONMENT=development
APP_NAME=kubernetes-learning
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
...
```

### Useful log commands:

```bash
# View logs
kubectl logs <pod-name>

# Follow logs (like tail -f)
kubectl logs -f <pod-name>

# View logs from a specific container (if multi-container pod)
kubectl logs <pod-name> -c <container-name>

# View previous container logs (if container restarted)
kubectl logs <pod-name> --previous
```

---

## Key Learnings

### 1. **Pod is the Atomic Unit**
- Pods are the smallest deployable unit in Kubernetes
- A Pod can contain one or more containers
- Containers in a Pod share network and storage

### 2. **Pod Lifecycle**
- **Pending:** Pod accepted, but containers not created
- **Running:** Pod bound to node, all containers created, at least one running
- **Succeeded:** All containers terminated successfully
- **Failed:** At least one container terminated with failure
- **Unknown:** Pod state cannot be determined

### 3. **Environment Variables**
- Can be set directly in the manifest (as we did)
- Can be injected from ConfigMaps and Secrets
- Kubernetes automatically injects service discovery variables

### 4. **Debugging Workflow**
1. **Check status:** `kubectl get pods` - Quick overview
2. **Get details:** `kubectl describe pod` - Detailed information, events, conditions
3. **Check logs:** `kubectl logs` - Application output and errors
4. **Check events:** Events section in describe shows what happened

### 5. **Best Practices**
- Always use labels for organization and selection
- Use meaningful names for Pods and containers
- Define containerPorts for documentation (even if not required)
- Use environment variables for configuration (not hardcoded values)

---

## Deliverables

**3 pod manifests** created in `exercises/pods/`:
- `nginx-pod.yaml`
- `busybox-sleep-pod.yaml`
- `env-pod.yaml`

**All pods verified** as Running

**Documentation** covering:
- Pod structure (apiVersion, kind, metadata, spec)
- What you see in `kubectl describe pod`
- What you see in `kubectl logs`

---

## Next Steps

- Practice with multi-container pods
- Learn about Pod lifecycle and restart policies
- Explore resource limits and requests
- Work with ConfigMaps and Secrets for environment variables
