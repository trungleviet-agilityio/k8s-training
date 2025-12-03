# Observability and Debugging

## Quick Reference

### kubectl Command Quick Reference

| Command | Purpose | Example |
|---------|---------|---------|
| `kubectl get pods` | List pods | `kubectl get pods -l app=my-app` |
| `kubectl describe pod` | Detailed info | `kubectl describe pod <name>` |
| `kubectl logs` | View logs | `kubectl logs <pod> -f` |
| `kubectl exec` | Execute command | `kubectl exec -it <pod> -- /bin/sh` |
| `kubectl get events` | View events | `kubectl get events --sort-by='.lastTimestamp'` |
| `kubectl get all` | All resources | `kubectl get all -l app=my-app` |

### Problem → Command Reference

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

## Debugging

### Debugging Workflow

A systematic approach to debugging Kubernetes applications:

1. **`kubectl get`** - Quick status check
2. **`kubectl describe`** - Detailed information
3. **`kubectl logs`** - Application output
4. **`kubectl exec`** - Interactive debugging

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

## Logging

### Logging Commands

**Basic Logging (Single Container Pod):**
- `kubectl logs <pod-name>` - View logs from pod (similar to `docker logs`)
- Logs come from container's stdout/stderr
- Shows all log output from application

**Following Logs:**
- `kubectl logs <pod-name> -f` - Stream logs live (like `docker logs -f`)
- Continuously displays new log entries as they're generated

**Multi-container Pods:**
- Must specify container name with `-c` option
- `kubectl logs <pod-name> -c <container-name>`
- Without `-c`, command fails asking to specify container name
- Each container has separate log stream

**Other Options:**
- `kubectl logs <pod-name> --tail=50` - Show last 50 lines
- `kubectl logs <pod-name> --previous` - Show logs from previous container instance (if crashed/restarted)
- `kubectl logs <pod-name> --since=10m` - Show logs from last 10 minutes

**Examples:**
```bash
# Single container pod
kubectl logs event-simulator-pod
kubectl logs event-simulator-pod -f

# Multi-container pod (must specify container)
kubectl logs my-pod -c event-simulator
kubectl logs my-pod -c image-processor -f

# View logs from crashed container
kubectl logs my-pod --previous
```

**Note:** This covers basic Kubernetes logging. Advanced logging with third-party tools (ELK, Fluentd, etc.) and log aggregation are separate topics.

**Reference:** [Kubernetes Logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/)

## Monitoring

### Monitor and Debug Applications

**What to Monitor:**
- **Node-level metrics:** Number of nodes, health status, CPU, memory, network, disk utilization
- **Pod-level metrics:** Number of pods, CPU and memory consumption per pod

**Monitoring Solutions:**
- **Metrics Server:** Built-in, in-memory solution (one per cluster)
- **Advanced solutions:** Prometheus, Elastic Stack, Datadog, Dynatrace (for historical data)

**Metrics Server:**
- Retrieves metrics from each Kubernetes node and pod
- Aggregates and stores metrics in memory only
- **No historical data** (metrics not stored on disk)
- Required for `kubectl top` commands to work

**How Metrics are Generated:**
- **Kubelet:** Agent running on each node, receives instructions from Kubernetes API
- **cAdvisor (Container Advisor):** Subcomponent of Kubelet
  - Retrieves performance metrics from pods
  - Exposes metrics through Kubelet API
  - Makes metrics available to Metrics Server

**Enable Metrics Server:**

```bash
# Minikube
minikube addons enable metrics-server

# Other environments
# Deploy from GitHub: https://github.com/kubernetes-sigs/metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**View Metrics:**

```bash
# Node-level metrics (CPU and memory)
kubectl top nodes

# Pod-level metrics (CPU and memory)
kubectl top pods
kubectl top pods -l app=my-app

# View events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod-name>

# View all resources
kubectl get all
kubectl get all -l app=my-app
```

**Note:** Metrics Server provides current metrics only. For historical data and advanced analytics, use solutions like Prometheus or Elastic Stack.

**Reference:** [Kubernetes Metrics](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/) | [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
