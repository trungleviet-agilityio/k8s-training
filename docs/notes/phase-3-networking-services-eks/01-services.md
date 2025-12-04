# Services

## Quick Reference

### Service Types Comparison

| Type | Access | IP | Use Case | Example |
|------|--------|----|----------|---------|
| **ClusterIP** | Internal cluster only | Cluster IP (e.g., 10.111.92.163) | Internal services, microservices communication | Database, internal APIs |
| **NodePort** | Internal + External (via node IP) | Cluster IP + Node IP | Development, testing, external access without LoadBalancer | `<node-ip>:30080` |
| **LoadBalancer** | External via cloud LB | Cluster IP + External IP | Production external access (cloud providers) | AWS ELB, GCP LB |

### Port Types

| Port Type | Location | Purpose | Example |
|-----------|----------|---------|---------|
| **port** | Service spec | Port exposed by Service | `port: 80` |
| **targetPort** | Service spec | Port on Pod/container | `targetPort: 80` |
| **nodePort** | Service spec (NodePort only) | Port on node IP | `nodePort: 30080` |
| **containerPort** | Pod spec | Port container listens on | `containerPort: 80` |

## Common Use Cases

### When to Use Each Service Type

**Use ClusterIP when:**
- Internal service communication (default, most common)
- Microservices talking to each other
- Database services
- Services accessed via Ingress

**Use NodePort when:**
- Development and testing
- External access without LoadBalancer
- Quick access from outside cluster
- Temporary external exposure

**Use LoadBalancer when:**
- Production external access
- Cloud provider environment (AWS, GCP, Azure)
- Need stable external IP
- High availability requirements

## Essential Commands

### Service Operations

```bash
# Create/apply service
kubectl apply -f exercises/services/nginx-clusterip.yaml
kubectl apply -f exercises/services/nginx-nodeport.yaml

# List services
kubectl get svc
kubectl get svc -l app=nginx

# Describe service
kubectl describe svc <service-name>

# Get service details
kubectl get svc <service-name> -o yaml

# Delete service
kubectl delete svc <service-name>
```

### Port Information

```bash
# Get ClusterIP
kubectl get svc <name> -o jsonpath='{.spec.clusterIP}'

# Get Service port
kubectl get svc <name> -o jsonpath='{.spec.ports[0].port}'

# Get targetPort
kubectl get svc <name> -o jsonpath='{.spec.ports[0].targetPort}'

# Get nodePort (if NodePort service)
kubectl get svc <name> -o jsonpath='{.spec.ports[0].nodePort}'

# Get endpoints (actual pod IPs)
kubectl get endpoints <service-name>
```

### DNS-based Service Discovery

```bash
# Test service name (same namespace)
kubectl exec -it <pod-name> -- curl http://nginx-clusterip

# Test FQDN (cross-namespace)
kubectl exec -it <pod-name> -- curl http://nginx-clusterip.default.svc.cluster.local

# Test using ClusterIP
CLUSTER_IP=$(kubectl get svc nginx-clusterip -o jsonpath='{.spec.clusterIP}')
kubectl exec -it <pod-name> -- curl http://$CLUSTER_IP
```

## Service Types Details

### ClusterIP Service

**Characteristics:**
- Default Service type (if not specified)
- Accessible only from within the cluster
- Gets a virtual IP in cluster IP range
- Most common for internal services

**Example (`exercises/services/nginx-clusterip.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

**Apply this:**
```bash
kubectl apply -f exercises/services/nginx-clusterip.yaml
```

**Access Methods:**
- Service name: `http://nginx-clusterip`
- Cluster IP: `http://10.111.92.163`
- FQDN: `http://nginx-clusterip.default.svc.cluster.local`

### NodePort Service

**Characteristics:**
- Exposes service on each node's IP at a static port
- Port range: 30000-32767 (default)
- Accessible from outside cluster
- Also accessible internally (like ClusterIP)

**Example (`exercises/services/nginx-nodeport.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
```

**Apply this:**
```bash
kubectl apply -f exercises/services/nginx-nodeport.yaml
```

**Access Methods:**
- External: `http://<node-ip>:30080`
- Internal: `http://nginx-nodeport` (works like ClusterIP)

### LoadBalancer Service

**Characteristics:**
- Cloud provider creates external load balancer
- Gets external IP address
- Most expensive option
- Requires cloud provider support

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

**Access Methods:**
- External IP: `http://<external-ip>`
- Internal: `http://nginx-loadbalancer` (works like ClusterIP)

## DNS-based Service Discovery

### Service DNS Names

Services automatically get DNS names:

- **Short name:** `nginx-clusterip` (same namespace)
- **FQDN:** `nginx-clusterip.default.svc.cluster.local`
  - Format: `<service-name>.<namespace>.svc.cluster.local`

### Cross-Namespace Access

To access a service from another namespace, use FQDN:

```bash
# From different namespace
curl http://nginx-clusterip.default.svc.cluster.local
```

## Common Issues & Solutions

### Service Has No Endpoints

**Symptoms:** `ENDPOINTS` shows `<none>` or empty

**Commands:**
```bash
# Check service selector
kubectl get svc <service-name> -o yaml | grep selector

# Check pod labels
kubectl get pods --show-labels

# Verify selector matches pod labels
kubectl get pods -l app=nginx
```

**Common causes:**
- Selector doesn't match pod labels → Fix selector or pod labels
- No pods running → Check deployment/pod status
- Wrong namespace → Check service and pods are in same namespace

**Fix:**
```yaml
# Verify selector matches pod labels
spec:
  selector:
    app: nginx  # Must match pod label
```

### Can't Access Service from Pod

**Symptoms:** Connection refused, timeout

**Commands:**
```bash
# Test service name
kubectl exec -it <pod-name> -- curl http://nginx-clusterip

# Test FQDN
kubectl exec -it <pod-name> -- curl http://nginx-clusterip.default.svc.cluster.local

# Check service exists
kubectl get svc nginx-clusterip

# Check endpoints
kubectl get endpoints nginx-clusterip
```

**Common causes:**
- Wrong service name → Use correct service name or FQDN
- Wrong namespace → Use FQDN for cross-namespace access
- Service not created → Verify service exists

**Fix:**
```bash
# For cross-namespace access, use FQDN
kubectl exec -it <pod> -- curl http://nginx-clusterip.default.svc.cluster.local
```

### NodePort Not Accessible

**Symptoms:** Connection refused from outside cluster

**Commands:**
```bash
# Get node IP
kubectl get nodes -o wide

# Get nodePort
kubectl get svc <service-name> -o jsonpath='{.spec.ports[0].nodePort}'

# Test from host
curl http://<node-ip>:<nodePort>
```

**Common causes:**
- Wrong node IP → Use correct node IP
- Firewall blocking → Check firewall rules
- Wrong nodePort → Verify nodePort number
