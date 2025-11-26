# Services – ClusterIP and NodePort

**Goal:** Learn how Services provide stable networking for Pods.

---

## Service Manifests

### 1. ClusterIP Service

Created `exercises/services/nginx-clusterip.yaml`:

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

### 2. NodePort Service

Created `exercises/services/nginx-nodeport.yaml`:

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

---

## Understanding Service Ports

### Port Configuration Explained

A Service has three types of ports:

1. **port** (Service Port)
   - The port exposed by the Service within the cluster
   - Used by other pods/services to access this service
   - Example: `port: 80` means the service is accessible on port 80

2. **targetPort** (Container Port)
   - The port on the Pod/container that receives traffic
   - Must match the `containerPort` in the Pod spec
   - Example: `targetPort: 80` means traffic goes to container port 80

3. **nodePort** (Node Port - NodePort services only)
   - The port exposed on each node's IP address
   - Accessible from outside the cluster
   - Range: 30000-32767 (default Kubernetes range)
   - Example: `nodePort: 30080` means accessible on `<node-ip>:30080`

### How to Find These Ports

#### Using kubectl get

```bash
kubectl get svc <service-name>
```

**Output:**
```
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-clusterip   ClusterIP   10.111.92.163   <none>        80/TCP         9s
nginx-nodeport    NodePort    10.104.174.69   <none>        80:30080/TCP   3s
```

**Reading the output:**
- **PORT(S):** Shows `port/nodePort` format for NodePort services
  - `80/TCP` = port 80 only (ClusterIP)
  - `80:30080/TCP` = port 80, nodePort 30080 (NodePort)

#### Using kubectl describe

```bash
kubectl describe svc <service-name>
```

**ClusterIP Service Output:**
```
Name:                     nginx-clusterip
Type:                     ClusterIP
IP:                       10.111.92.163
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                10.244.0.15:80,10.244.0.4:80,10.244.0.14:80
```

**NodePort Service Output:**
```
Name:                     nginx-nodeport
Type:                     NodePort
IP:                       10.104.174.69
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.0.4:80,10.244.0.14:80,10.244.0.15:80
```

**Key Information:**
- **Port:** Service port (80)
- **TargetPort:** Container port (80)
- **NodePort:** Node port (30080 for NodePort services)
- **Endpoints:** Actual pod IPs and ports that receive traffic

#### Using kubectl get with JSONPath

```bash
# Get ClusterIP
kubectl get svc nginx-clusterip -o jsonpath='{.spec.clusterIP}'

# Get nodePort
kubectl get svc nginx-nodeport -o jsonpath='{.spec.ports[0].nodePort}'

# Get port
kubectl get svc nginx-clusterip -o jsonpath='{.spec.ports[0].port}'

# Get targetPort
kubectl get svc nginx-clusterip -o jsonpath='{.spec.ports[0].targetPort}'
```

---

## ClusterIP vs NodePort

### ClusterIP Service

**Characteristics:**
- **Type:** `ClusterIP` (default if not specified)
- **Access:** Only accessible from within the cluster
- **IP:** Gets a virtual IP in the cluster IP range (e.g., 10.111.92.163)
- **Use Case:** Internal communication between services

**Access Methods:**
1. **Service Name:** `http://nginx-clusterip` (DNS resolution)
2. **Cluster IP:** `http://10.111.92.163`
3. **FQDN:** `http://nginx-clusterip.default.svc.cluster.local`

**Example:**
```bash
# From inside a pod
curl http://nginx-clusterip
curl http://10.111.92.163
```

### NodePort Service

**Characteristics:**
- **Type:** `NodePort`
- **Access:** Accessible from outside the cluster via any node's IP
- **IP:** Gets a ClusterIP (for internal access) + exposes on node IP
- **Port Range:** 30000-32767 (default)
- **Use Case:** External access to services (development/testing)

**Access Methods:**
1. **From outside cluster:** `<node-ip>:<nodePort>` (e.g., `192.168.39.254:30080`)
2. **From inside cluster:** Same as ClusterIP (service name or ClusterIP)

**Example:**
```bash
# From host machine
curl http://192.168.39.254:30080

# From inside cluster (works like ClusterIP)
curl http://nginx-nodeport
```

### Key Differences

| Feature | ClusterIP | NodePort |
|---------|-----------|----------|
| **Access** | Internal only | Internal + External |
| **IP** | Cluster IP only | Cluster IP + Node IP |
| **Port** | Service port only | Service port + Node port |
| **Use Case** | Internal services | External access (dev/test) |
| **Security** | More secure | Less secure (exposed externally) |

---

## Testing Connectivity

### Apply Services

```bash
kubectl apply -f exercises/services/nginx-clusterip.yaml
kubectl apply -f exercises/services/nginx-nodeport.yaml
```

**Output:**
```
service/nginx-clusterip created
service/nginx-nodeport created
```

### Verify Services

```bash
kubectl get svc -l app=nginx
```

**Output:**
```
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-clusterip   ClusterIP   10.111.92.163   <none>        80/TCP         9s
nginx-nodeport    NodePort    10.104.174.69   <none>        80:30080/TCP   3s
```

### Test ClusterIP from Inside Cluster

#### Method 1: Using Service Name (DNS)

```bash
# Get a pod name
kubectl get pods -l app=nginx -o name | head -1

# Exec into pod and curl the service
kubectl exec -it <pod-name> -- curl -s http://nginx-clusterip
```

**Output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

#### Method 2: Using Cluster IP

```bash
kubectl exec -it <pod-name> -- curl -s http://10.111.92.163
```

**Result:** Same HTML response, confirming ClusterIP works.

**Key Observations:**
- ✅ Service name resolves via DNS
- ✅ ClusterIP is accessible from pods
- ✅ Load balancing works (traffic distributed to multiple pods)

### Test NodePort from Host

#### Get Minikube Node IP

```bash
minikube ip
```

**Output:**
```
192.168.39.254
```

#### Access via NodePort

```bash
curl http://$(minikube ip):30080
```

**Output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

**Key Observations:**
- ✅ NodePort is accessible from host machine
- ✅ Traffic reaches the pods through the service
- ✅ Can access via any node's IP on the nodePort

### Verify Endpoints

```bash
kubectl get endpoints nginx-clusterip
```

**Output:**
```
NAME              ENDPOINTS                                     AGE
nginx-clusterip   10.244.0.14:80,10.244.0.15:80,10.244.0.4:80   70s
```

**Understanding Endpoints:**
- Endpoints show the actual pod IPs that the service routes to
- Multiple endpoints = load balancing across pods
- Endpoints are automatically updated when pods are created/deleted

---

## Service Selector and Pod Matching

### How Services Find Pods

Services use **selectors** to find pods:

```yaml
spec:
  selector:
    app: nginx
```

This matches pods with the label `app: nginx`.

**Verification:**
```bash
# Check which pods match the selector
kubectl get pods -l app=nginx

# Check service endpoints (should match pod IPs)
kubectl get endpoints nginx-clusterip
```

**Important:**
- Selector labels must match pod labels
- If no pods match, endpoints will be empty
- Service will still be created, but won't route traffic

---

## Service DNS Resolution

### DNS Names

Services get DNS names automatically:

- **Short name:** `nginx-clusterip` (same namespace)
- **FQDN:** `nginx-clusterip.default.svc.cluster.local`
  - Format: `<service-name>.<namespace>.svc.cluster.local`

### Testing DNS

```bash
# From inside a pod
kubectl exec -it <pod-name> -- nslookup nginx-clusterip
```

**Note:** Not all container images have `nslookup`. Alternative:

```bash
# Test DNS resolution by curling the service name
kubectl exec -it <pod-name> -- curl -s http://nginx-clusterip.default.svc.cluster.local
```

---

## Key Learnings

### 1. **Service Provides Stable Networking**
- Pods are ephemeral (IPs change when recreated)
- Services provide stable IP and DNS name
- Traffic is automatically routed to healthy pods

### 2. **Port Mapping**
- **port:** What clients use to connect
- **targetPort:** Where traffic goes in the container
- **nodePort:** External access point (NodePort only)

### 3. **ClusterIP vs NodePort**
- **ClusterIP:** Internal only, more secure
- **NodePort:** External access, less secure, good for dev/test
- **Production:** Usually use ClusterIP with Ingress or LoadBalancer

### 4. **Load Balancing**
- Services automatically load balance across matching pods
- Round-robin by default
- Health checks ensure only ready pods receive traffic

### 5. **Service Discovery**
- DNS-based service discovery within cluster
- Service names resolve to ClusterIPs
- Works across namespaces using FQDN

### 6. **Endpoints**
- Endpoints are automatically managed
- Show actual pod IPs that receive traffic
- Empty endpoints = no matching pods

---

## Common Commands

### Service Management

```bash
# Create service
kubectl apply -f service.yaml

# List services
kubectl get svc

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

# Get nodePort
kubectl get svc <name> -o jsonpath='{.spec.ports[0].nodePort}'

# Get all port info
kubectl get svc <name> -o jsonpath='{.spec.ports[*]}'
```

### Endpoints

```bash
# List endpoints
kubectl get endpoints

# Describe endpoints
kubectl describe endpoints <service-name>
```

### Testing

```bash
# Test from inside cluster
kubectl exec -it <pod-name> -- curl http://<service-name>

# Test NodePort from host
curl http://<node-ip>:<nodePort>

# Port forward (alternative to NodePort)
kubectl port-forward svc/<service-name> 8080:80
```

---

## Deliverables

**2 Service YAML files** created in `exercises/services/`:
- `nginx-clusterip.yaml` - ClusterIP service for internal access
- `nginx-nodeport.yaml` - NodePort service for external access

**Documentation** covering:
- Difference between ClusterIP vs NodePort
- How to find targetPort, port, nodePort
- Testing connectivity from inside and outside cluster
- Service discovery and DNS
- Load balancing and endpoints

---

## Best Practices

1. **Use ClusterIP by default** for internal services
2. **Use NodePort sparingly** (mainly for development/testing)
3. **Match selectors carefully** - ensure service selector matches pod labels
4. **Use meaningful service names** - they become DNS names
5. **Document port mappings** - especially when port != targetPort
6. **Monitor endpoints** - empty endpoints indicate no matching pods
7. **For production:** Use ClusterIP with Ingress or LoadBalancer for external access

---

## Next Steps

- Learn about LoadBalancer services
- Explore Ingress for advanced routing
- Practice with headless services
- Learn about service mesh concepts
