# Day 05 — Ingress Basics

**Goal:** Understand how Ingress exposes HTTP routes.

---

## Official Documentation

- https://kubernetes.io/docs/concepts/services-networking/ingress/
- https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/
- https://kind.sigs.k8s.io/docs/user/ingress/

---

## What is Ingress?

From official Kubernetes docs:

> "An Ingress is an API object that manages external access to the services in a cluster, typically HTTP."

**Key Points:**
- **Service** exposes Pods within the cluster
- **Ingress** exposes Services via HTTP/HTTPS from outside the cluster
- Ingress defines routing rules (paths, hosts)
- An **Ingress Controller** watches these rules and configures a reverse proxy

---

## Step 1 – Install Ingress Controller

### For Minikube

```bash
minikube addons enable ingress
```

**What this does:**
- Installs NGINX Ingress Controller in `ingress-nginx` namespace
- Creates Deployment, Service, and ConfigMaps
- Sets up admission webhooks

**Verify installation:**
```bash
kubectl get pods -n ingress-nginx
```

**Expected output:**
```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxx                1/1     Running   0          Xm
```

**Note:** Controller may take a few minutes to start (pulling large image).

**Check controller service:**
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

**Expected output:**
```
NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.109.13.56   <none>        80:31942/TCP,443:32353/TCP   Xm
```

**Key:** Note the NodePort (e.g., `31942`) - you'll need it for testing.

### For kind

Follow: https://kind.sigs.k8s.io/docs/user/ingress/

---

## Step 2 – Create ClusterIP Service

Ingress requires a **ClusterIP** service (not NodePort).

**File:** `exercises/ingress/nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
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

**Apply:**
```bash
kubectl apply -f exercises/ingress/nginx-service.yaml
```

**Verify:**
```bash
kubectl get svc nginx-service
kubectl get endpoints nginx-service
```

**Expected output:**
```
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.110.182.205   <none>        80/TCP    Xs

NAME            ENDPOINTS                                     AGE
nginx-service   10.244.0.14:80,10.244.0.15:80,10.244.0.4:80   Xs
```

---

## Step 3 – Create Ingress Resource

**File:** `exercises/ingress/nginx-ingress.yaml`

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

**Key components:**
- `ingressClassName: nginx` - **REQUIRED** - matches the controller
- `path: /` - Routes root path
- `pathType: Prefix` - Matches `/` and all subpaths
- `backend.service.name` - Service to route to

**Apply:**
```bash
kubectl apply -f exercises/ingress/nginx-ingress.yaml
```

**Verify:**
```bash
kubectl get ingress
```

**Expected output:**
```
NAME            CLASS   HOSTS   ADDRESS   PORTS   AGE
nginx-ingress   nginx   *                80      Xs
```

**Describe ingress:**
```bash
kubectl describe ingress nginx-ingress
```

**Expected output:**
```
Name:             nginx-ingress
Namespace:        default
Ingress Class:    nginx
Rules:
  Host  Path  Backends
  ----  ----  --------
  *     
       /   nginx-service:80 (10.244.0.14:80,10.244.0.15:80,10.244.0.4:80)
```

---

## Step 4 – Test Ingress

### For Minikube

1. **Get minikube IP:**
   ```bash
   minikube ip
   ```
   Example output: `192.168.39.254`

2. **Get NodePort:**
   ```bash
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   ```
   Look for: `80:31942/TCP` (31942 is the NodePort)

3. **Test with curl:**
   ```bash
   MINIKUBE_IP=$(minikube ip)
   curl http://$MINIKUBE_IP:31942/
   ```

**Expected output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

### For kind

Since kind maps port 80 directly:

```bash
curl http://localhost/
```

**Expected output:** NGINX welcome page HTML

---

## How to Find targetPort, port, nodePort

### Using kubectl get

```bash
# Service ports
kubectl get svc nginx-service
# Shows: PORT(S) = 80/TCP
# port = 80

# Ingress controller NodePort
kubectl get svc -n ingress-nginx ingress-nginx-controller
# Shows: PORT(S) = 80:31942/TCP
# port = 80, nodePort = 31942
```

### Using kubectl describe

```bash
kubectl describe svc nginx-service
```

**Shows:**
```
Port:                    80/TCP
TargetPort:              80/TCP
```

**Understanding:**
- **port:** Service port (what clients use)
- **targetPort:** Container port (where traffic goes)
- **nodePort:** External port on node (NodePort services only)

---

## Common Pitfalls

### ❌ No ingressClassName

**Problem:** Ingress won't be handled by controller

**Solution:** Always specify `ingressClassName: nginx`

### ❌ Wrong Service selector

**Problem:** No endpoints → 502 Bad Gateway

**Solution:** Verify service selector matches pod labels
```bash
kubectl get endpoints nginx-service
```

### ❌ Testing wrong port

**Problem:** Connection refused

**Solution:** Use Ingress Controller NodePort, not Service NodePort
```bash
# Correct: Use ingress controller NodePort
kubectl get svc -n ingress-nginx ingress-nginx-controller
# Look for: 80:XXXXX/TCP
```

---

## Key Commands

```bash
# Install controller (minikube)
minikube addons enable ingress

# Verify controller
kubectl get pods -n ingress-nginx

# Create service
kubectl apply -f exercises/ingress/nginx-service.yaml

# Create ingress
kubectl apply -f exercises/ingress/nginx-ingress.yaml

# List ingress
kubectl get ingress

# Describe ingress
kubectl describe ingress nginx-ingress

# Test (minikube)
MINIKUBE_IP=$(minikube ip)
curl http://$MINIKUBE_IP:31942/

# Test (kind)
curl http://localhost/
```

---

## Key Learnings

1. **Ingress vs Service:**
   - Service: Stable networking within cluster
   - Ingress: HTTP/HTTPS routing from outside cluster

2. **Ingress Controller Required:**
   - Ingress resource alone does nothing
   - Controller watches Ingress and configures proxy

3. **ingressClassName is Critical:**
   - Must match installed controller (`nginx`)
   - Without it, Ingress won't work

4. **Service Must Be ClusterIP:**
   - Ingress requires ClusterIP service
   - Verify endpoints exist before testing

---

## Troubleshooting

**Issue: ADDRESS is empty**
- Controller not ready → Wait and check controller pods

**Issue: 502 Bad Gateway**
- No endpoints → Check service selector matches pod labels

**Issue: Connection Refused**
- Wrong port → Use ingress controller NodePort, not service port

**Issue: 404 Not Found**
- Path mismatch → Verify path and pathType in Ingress

---

## Deliverables

**Ingress controller installed** (minikube addon)
**nginx-service.yaml** - ClusterIP service
**nginx-ingress.yaml** - Routes `/` to nginx-service
**Tested with curl** - Verified HTTP access

---

## Summary

You now understand:
- How to install Ingress controller (minikube addon)
- How to create Ingress resource
- How to test Ingress with curl
- Difference between ClusterIP and NodePort
- How to find port, targetPort, nodePort

**Ingress is essential for exposing HTTP applications in Kubernetes.**
