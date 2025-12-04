# Ingress

## Services vs Ingress Evolution

### The Problem: Exposing Applications Externally

**Scenario:** Deploying an online store application on Kubernetes that needs to be accessible to users.

**Evolution Path:**

#### 1. NodePort Service

**Initial Setup:**
- Deploy application as Pod/Deployment
- Create MySQL database with ClusterIP service (internal access)
- Create NodePort service for web application (external access)

**How it works:**
- Kubernetes allocates a high port (30000+) on all nodes
- Users access via: `http://<node-ip>:30080`
- Service splits traffic between pod replicas

**Limitations:**
- Users must remember port numbers (30080, not standard port 80)
- High ports only (30000+), can't use standard ports
- Need to configure DNS to point to node IPs
- Still need external proxy to map port 80 → 30080
- Users see port numbers in URLs

#### 2. LoadBalancer Service (Cloud)

**On Public Cloud (e.g., GCP, AWS, Azure):**
- Set service type to `LoadBalancer`
- Kubernetes provisions cloud-native load balancer
- Load balancer gets external IP
- Users access via: `http://my-online-store.com` (DNS points to load balancer IP)

**Limitations:**
- **One load balancer per service** (expensive)
- Need another proxy/load balancer for URL-based routing
- Multiple services = multiple load balancers = high costs
- SSL/TLS configuration at multiple levels (app, load balancer, proxy)
- Difficult to manage at scale

**Example Problem:**
```
Web App   → LoadBalancer #1 (IP: 1.2.3.4)
Video App → LoadBalancer #2 (IP: 5.6.7.8)
Wear App  → LoadBalancer #3 (IP: 9.10.11.12)

Need another proxy to route:
- my-online-store.com       → LoadBalancer #1
- watch.my-online-store.com → LoadBalancer #2
- wear.my-online-store.com  → LoadBalancer #3
```

#### 3. Ingress (Solution)

**Ingress Benefits:**
- **Single entry point** for all HTTP/HTTPS traffic
- **Path-based routing:** `my-online-store.com/` → web, `/watch` → video
- **Host-based routing:** `watch.my-online-store.com` → video app
- **SSL/TLS termination** in one place
- **Managed within Kubernetes** (just another YAML file)
- **Cost-effective:** One load balancer (or NodePort) for Ingress Controller

**How it works:**
- Deploy Ingress Controller (nginx, HAProxy, etc.) once
- Create Ingress Resources (rules) for each application
- Controller monitors cluster and configures itself
- All traffic flows through single controller

**Comparison:**

| Approach | Entry Points | URL Routing | SSL Config | Cost | Management |
|----------|--------------|-------------|------------|------|------------|
| **NodePort** | Multiple (one per service) | No | Per app | Low | Manual |
| **LoadBalancer** | One per service | No | Multiple levels | High | Complex |
| **Ingress** | One (controller) | Yes (path/host) | Centralized | Medium | Kubernetes-native |

## Quick Reference

### Ingress vs Service

| Feature | Service | Ingress |
|---------|---------|---------|
| **Purpose** | Exposes Pods within cluster | Exposes Services via HTTP/HTTPS from outside |
| **Type** | ClusterIP, NodePort, LoadBalancer | HTTP/HTTPS routing rules |
| **Controller** | Built-in | Requires Ingress Controller |
| **Use Case** | Internal access | External HTTP/HTTPS access |
| **Routing** | Load balancing only | Path-based, host-based routing |
| **SSL/TLS** | Per service | Centralized termination |
| **Cost** | Low (NodePort) to High (LoadBalancer) | Medium (one controller) |
| **Management** | Kubernetes-native | Kubernetes-native |

### Path Types

| Path Type | Behavior | Example |
|-----------|----------|---------|
| **Prefix** | Matches path and all subpaths | `/` matches `/`, `/api`, `/api/v1` |
| **Exact** | Matches exact path only | `/api` matches only `/api` |
| **ImplementationSpecific** | Controller-specific behavior | Varies by controller |

## Common Use Cases

### When to Use What

**Use NodePort when:**
- Quick testing and development
- Simple setups with single service
- No cloud load balancer available
- Temporary access needed
- **Limitation:** Users must use high ports (30000+)

**Use LoadBalancer when:**
- Single service needs external access
- Non-HTTP protocols (TCP/UDP)
- Cloud-native load balancer required
- **Limitation:** One load balancer per service (expensive)

**Use Ingress when:**
- **Production HTTP/HTTPS** services
- **Multiple services** need external access
- **Path-based routing** needed (`/api`, `/web`, `/watch`)
- **Host-based routing** needed (`app.example.com`, `api.example.com`)
- **SSL/TLS termination** in one place
- **Cost-effective** solution (one controller for all services)
- **Kubernetes-native** configuration management

**Don't use Ingress when:**
- TCP/UDP protocols (use NodePort or LoadBalancer)
- Simple internal access (use ClusterIP)
- Quick testing (use NodePort)
- Single service with no routing needs (use LoadBalancer)

## Ingress Controller vs Ingress Resources

### Key Concepts

**Ingress Controller:**
- The **actual load balancer/proxy** deployed in the cluster
- Examples: NGINX, HAProxy, Traefik, Contour, Istio
- Monitors Kubernetes cluster for Ingress resources
- Automatically configures itself based on Ingress resources
- **Not included by default** in Kubernetes clusters

**Ingress Resources:**
- **Configuration rules** applied to the Ingress Controller
- Created using YAML definition files (like Pods, Deployments)
- Define routing rules (paths, hosts, backends)
- Controller reads these resources and configures itself

**How They Work Together:**
```
1. Deploy Ingress Controller (once per cluster)
   └─> Controller pod runs and monitors cluster

2. Create Ingress Resources (one per application/routing need)
   └─> Controller detects new Ingress resources
   └─> Controller configures itself based on rules

3. Traffic flows:
   User → Ingress Controller → Backend Service → Pods
```

### Supported Ingress Controllers

**Kubernetes-maintained:**
- **GCE** (Google Cloud HTTP Load Balancer)
- **NGINX** (most popular, used in this course)

**Community-maintained:**
- **Contour** (Envoy-based)
- **HAProxy**
- **Traefik**
- **Istio** (service mesh with ingress)

**Important:** Choose one controller for your cluster. Multiple controllers can coexist but require different `ingressClassName` values.

## Essential Commands

### Ingress Operations

```bash
# Create/apply ingress resources
kubectl apply -f exercises/ingress/nginx-service.yaml
kubectl apply -f exercises/ingress/nginx-ingress.yaml

# Create ingress imperatively (k8s 1.20+)
kubectl create ingress <ingress-name> \
  --rule="host/path=service:port" \
  --class=nginx

# Example: Create ingress with path-based routing
kubectl create ingress ingress-test \
  --rule="wear.my-online-store.com/wear*=wear-service:80" \
  --class=nginx

# List ingress
kubectl get ingress
kubectl get ing  # Short form

# Describe ingress
kubectl describe ingress <ingress-name>

# Get ingress details
kubectl get ingress <ingress-name> -o yaml

# Delete ingress
kubectl delete ingress <ingress-name>
```

### Ingress Controller (Minikube)

**Quick Setup (Minikube Addon):**
```bash
# Install NGINX Ingress Controller (simplified)
minikube addons enable ingress

# Verify installation
kubectl get pods -n ingress-nginx

# Check controller service (note NodePort)
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Get controller NodePort
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}'
```

**Note:** Minikube addon simplifies deployment. In production, you deploy manually with:
- Deployment (nginx-ingress-controller image)
- Service (NodePort or LoadBalancer)
- ConfigMap (nginx configuration)
- ServiceAccount (RBAC permissions)

### Ingress Controller Components

**Full Deployment (Understanding Components):**

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
      - name: nginx-ingress-controller
        image: k8s.gcr.io/ingress-nginx/controller:v1.8.1
        command:
        - /nginx-ingress-controller
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        ports:
        - containerPort: 80
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
```

**Key Components:**
- **Deployment:** Runs nginx-ingress-controller image
- **Service:** Exposes controller (NodePort or LoadBalancer)
- **ConfigMap:** Stores nginx configuration (can be empty initially)
- **ServiceAccount:** Provides RBAC permissions to monitor cluster
- **Environment Variables:** POD_NAME, POD_NAMESPACE (required by controller)
- **Ports:** 80 (HTTP), 443 (HTTPS)

### Testing Ingress

```bash
# Get minikube IP
MINIKUBE_IP=$(minikube ip)

# Get Ingress Controller NodePort
NODEPORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}')

# Test Ingress
curl http://$MINIKUBE_IP:$NODEPORT/

# Test with host header (if host-based routing)
curl -H "Host: app.example.com" http://$MINIKUBE_IP:$NODEPORT/
```

## Ingress Configuration

### Path-based Routing

**Example (`exercises/ingress/nginx-ingress.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

**Apply this:**
```bash
# First apply the service
kubectl apply -f exercises/ingress/nginx-service.yaml

# Then apply the ingress
kubectl apply -f exercises/ingress/nginx-ingress.yaml
```

**Key components:**
- `ingressClassName: nginx` - **REQUIRED** - matches the controller
- `path: /` - Routes root path
- `pathType: Prefix` - Matches `/` and all subpaths
- `backend.service.name` - Service to route to

### Host-based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

**Key components:**
- `host: app.example.com` - Routes based on Host header
- Multiple rules can have different hosts
- Each host can have multiple paths

### Default Backend

**Purpose:**
- Handles traffic that **doesn't match any rules**
- Used for **404 Not Found** pages
- Catch-all route for unmatched requests

**Configuration:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx
  defaultBackend:  # Traffic not matching any rules
    service:
      name: default-http-backend
      port:
        number: 80
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

**Important:**
- Default backend service **must be deployed** separately
- If not specified, controller uses its own default backend
- Useful for showing custom 404 pages

**Example Default Backend Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
spec:
  selector:
    app: default-backend
  ports:
  - port: 80
    targetPort: 8080
```

### Multiple Rules with Multiple Paths

**Scenario:** Online store with multiple services accessible via different paths and hosts.

#### Example 1: Single Rule with Multiple Paths (Path-based Routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: my-online-store.com
    http:
      paths:
      # Root path → web app
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      # /watch path → video streaming app
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 80
      # /wear path → wear app
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
```

**Traffic Flow:**
```
my-online-store.com/          → web-service
my-online-store.com/watch      → video-service
my-online-store.com/watch/movie → video-service (Prefix matches)
my-online-store.com/wear       → wear-service
my-online-store.com/wear/shirt → wear-service (Prefix matches)
```

#### Example 2: Multiple Rules with Single Path (Host-based Routing)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
spec:
  ingressClassName: nginx
  rules:
  # Rule 1: watch subdomain → video app
  - host: watch.my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 80
  # Rule 2: wear subdomain → wear app
  - host: wear.my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
  # Rule 3: Main domain → web app
  - host: my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Traffic Flow:**
```
watch.my-online-store.com/     → video-service
wear.my-online-store.com/      → wear-service
my-online-store.com/           → web-service
```

#### Example 3: Multiple Rules with Multiple Paths (Combined)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: store-ingress
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
  rules:
  # Rule 1: Main domain with multiple paths
  - host: my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /support
        pathType: Prefix
        backend:
          service:
            name: support-service
            port:
              number: 80
  # Rule 2: Watch subdomain with multiple paths
  - host: watch.my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: video-service
            port:
              number: 80
      - path: /movies
        pathType: Prefix
        backend:
          service:
            name: movies-service
            port:
              number: 80
      - path: /tv
        pathType: Prefix
        backend:
          service:
            name: tv-service
            port:
              number: 80
  # Rule 3: Wear subdomain
  - host: wear.my-online-store.com
    http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
```

**Traffic Flow:**
```
my-online-store.com/            → web-service
my-online-store.com/api         → api-service
my-online-store.com/support     → support-service
watch.my-online-store.com/      → video-service
watch.my-online-store.com/movies → movies-service
watch.my-online-store.com/tv    → tv-service
wear.my-online-store.com/wear   → wear-service
<anything else>                 → default-http-backend (404)
```

**Path Matching Priority:**
1. **Exact match** (pathType: Exact) takes precedence
2. **Longest prefix** match (pathType: Prefix)
3. **Default backend** if no match

### Ingress Requirements

1. **Ingress Controller** must be installed (e.g., NGINX Ingress Controller)
2. **ClusterIP Service** (Ingress requires ClusterIP, not NodePort)
3. **ingressClassName** must match the controller

### API Version Changes

**Evolution:**
- **Old API (deprecated):** `extensions/v1beta1` (Kubernetes < 1.19)
- **Current API:** `networking.k8s.io/v1` (Kubernetes 1.19+)

**Key Changes:**

| Old Format (v1beta1) | New Format (v1) |
|---------------------|-----------------|
| `backend.serviceName` | `backend.service.name` |
| `backend.servicePort` | `backend.service.port.number` |
| Annotation: `kubernetes.io/ingress.class` | Field: `ingressClassName` |

**Migration Example:**

**Old Format (deprecated):**
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: old-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
```

**New Format (current):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: new-ingress
spec:
  ingressClassName: nginx  # Field, not annotation
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix  # REQUIRED in v1
        backend:
          service:
            name: nginx-service  # Changed structure
            port:
              number: 80  # Changed structure
```

**Important Notes:**
- `pathType` is **required** in `networking.k8s.io/v1` (Prefix, Exact, or ImplementationSpecific)
- `ingressClassName` is a **field** in v1, not an annotation
- Always use `networking.k8s.io/v1` for new Ingress resources

## Common Issues & Solutions

### Ingress ADDRESS is Empty

**Symptoms:** `kubectl get ingress` shows empty ADDRESS

**Commands:**
```bash
# Check Ingress Controller pods
kubectl get pods -n ingress-nginx

# Check controller service
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Check ingress events
kubectl describe ingress <ingress-name>
```

**Common causes:**
- Ingress Controller not ready → Wait for controller pods to be Running
- Controller not installed → Install with `minikube addons enable ingress`
- Wrong ingressClassName → Verify matches controller

**Fix:**
```bash
# Wait for controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 502 Bad Gateway

**Symptoms:** Ingress returns 502 Bad Gateway

**Commands:**
```bash
# Check service endpoints
kubectl get endpoints <service-name>

# Check service selector matches pod labels
kubectl get pods --show-labels
kubectl get svc <service-name> -o yaml | grep selector

# Check pods are running
kubectl get pods -l app=sample-app
```

**Common causes:**
- No endpoints on service → Service selector doesn't match pod labels
- Pods not running → Check deployment/pod status
- Wrong service name → Verify service name in Ingress matches actual service

**Fix:**
```yaml
# Verify service selector matches pod labels
# Service:
spec:
  selector:
    app: nginx  # Must match pod label

# Pods:
metadata:
  labels:
    app: nginx  # Must match service selector
```

### Connection Refused

**Symptoms:** Connection refused when accessing Ingress

**Commands:**
```bash
# Get Ingress Controller NodePort (correct port)
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Get minikube IP
minikube ip

# Test with correct port
curl http://$(minikube ip):<nodePort>/
```

**Common causes:**
- Using wrong port → Use Ingress Controller NodePort, not Service port
- Wrong IP → Use minikube IP, not service ClusterIP
- Controller not running → Check controller pods

**Fix:**
```bash
# Use Ingress Controller NodePort (not Service port)
NODEPORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}')
MINIKUBE_IP=$(minikube ip)
curl http://$MINIKUBE_IP:$NODEPORT/
```

### 404 Not Found

**Symptoms:** Ingress returns 404 Not Found

**Commands:**
```bash
# Check ingress rules
kubectl describe ingress <ingress-name>

# Test different paths
curl http://$MINIKUBE_IP:$NODEPORT/
curl http://$MINIKUBE_IP:$NODEPORT/api
```

**Common causes:**
- Path mismatch → Verify path and pathType in Ingress
- Wrong pathType → Use Prefix for subpaths, Exact for exact match
- Service not accessible → Check service endpoints

**Fix:**
```yaml
# Use Prefix for subpaths
paths:
- path: /api
  pathType: Prefix  # Matches /api, /api/v1, /api/v2, etc.
  backend:
    service:
      name: nginx-service
      port:
        number: 80
```
