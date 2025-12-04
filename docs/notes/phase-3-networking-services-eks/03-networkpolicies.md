# NetworkPolicies

## Quick Reference

### Default Behavior

| Scenario | Behavior | Traffic Allowed |
|----------|----------|----------------|
| **No NetworkPolicy** | Default allow | All pods can communicate freely |
| **Default Deny Policy** | Deny all ingress | No ingress traffic allowed |
| **Allow Rule Added** | Specific allow | Only specified traffic allowed |

**Important:** Kubernetes defaults to **allow-all** - without NetworkPolicies, all pods can communicate with each other. NetworkPolicies are opt-in security that enforce deny-by-default when a policy exists.

### NetworkPolicy Selectors

| Selector Type | Purpose | Example |
|---------------|---------|---------|
| **podSelector** | Selects pods this policy applies to | `matchLabels: {app: backend}` |
| **from.podSelector** | Allows traffic from specific pods | `matchLabels: {app: frontend}` |
| **from.namespaceSelector** | Allows traffic from specific namespace | `matchLabels: {name: production}` |
| **from.ipBlock** | Allows traffic from specific IP ranges | `cidr: 192.168.5.1/32` |
| **to.podSelector** | Allows traffic to specific pods (egress) | `matchLabels: {app: database}` |
| **to.ipBlock** | Allows traffic to external IP ranges (egress) | `cidr: 192.168.5.0/24` |

## Traffic Direction Fundamentals

### Ingress vs Egress

**Critical Concept:** Traffic direction is determined by where the **request originates**, not the response.

**Ingress (Incoming Traffic):**
- Traffic coming **into** a pod
- Example: User request to web server on port 80
- Example: Web server request to API server on port 5000 (from API server's perspective)

**Egress (Outgoing Traffic):**
- Traffic going **out from** a pod
- Example: Web server sending request to API server on port 5000 (from web server's perspective)
- Example: API server querying database on port 3306 (from API server's perspective)

**Important:** Response traffic is **automatically allowed** - you don't need a separate rule for responses. Once you allow ingress traffic, the response (egress) is automatically permitted.

**Example Flow:**
```
User → Web Server (port 80) → API Server (port 5000) → Database (port 3306)

From Web Server perspective:
- Ingress: User request on port 80 (incoming)
- Egress: Request to API server on port 5000 (outgoing)

From API Server perspective:
- Ingress: Request from web server on port 5000 (incoming)
- Egress: Query to database on port 3306 (outgoing)

From Database perspective:
- Ingress: Query from API server on port 3306 (incoming)
```

## Common Use Cases

### When to Use NetworkPolicies

**Use NetworkPolicies when:**
- Need to restrict pod-to-pod communication
- Security requirements (least privilege)
- Multi-tenant environments
- Compliance requirements
- Isolate sensitive workloads

**Default Deny Pattern:**
1. Create default deny-all policy
2. Add specific allow rules for required traffic
3. More secure than default allow

## Essential Commands

### NetworkPolicy Operations

```bash
# Create/apply NetworkPolicy
kubectl apply -f exercises/networkpolicies/default-deny-all.yaml
kubectl apply -f exercises/networkpolicies/frontend-to-backend.yaml

# List NetworkPolicies
kubectl get networkpolicies
kubectl get netpol  # Short form

# Describe NetworkPolicy
kubectl describe networkpolicy <policy-name>

# Get NetworkPolicy details
kubectl get networkpolicy <policy-name> -o yaml

# Delete NetworkPolicy
kubectl delete networkpolicy <policy-name>
```

### NetworkPolicy Requirements

**Important:** NetworkPolicies require a CNI plugin that supports them. Not all network solutions support NetworkPolicies.

**Supported CNIs:**
- Calico
- Cilium
- Romana
- WeaveNet
- kube-router

**Not Supported:**
- Flannel (as of recording)

**Important Notes:**
- You can create NetworkPolicies even if your CNI doesn't support them, but they won't be enforced
- No error message is shown if CNI doesn't support NetworkPolicies
- Always verify your CNI supports NetworkPolicies before relying on them for security

**For Minikube:**
```bash
# Minikube default CNI (bridge) does NOT support NetworkPolicies
# Start with Calico CNI
minikube stop
minikube start --cni=calico

# Verify Calico is running
kubectl get pods -n kube-system | grep calico

# Expected output:
# calico-kube-controllers-xxx    1/1     Running
# calico-node-xxx                1/1     Running
```

### Testing NetworkPolicy

```bash
# Deploy test pods
kubectl apply -f exercises/networkpolicies/backend-pod.yaml
kubectl apply -f exercises/networkpolicies/frontend-pod.yaml
kubectl apply -f exercises/networkpolicies/other-pod.yaml

# Check pod labels
kubectl get pods --show-labels

# Test connectivity from allowed pod (frontend)
kubectl exec -it frontend-pod -- wget -qO- --timeout=2 http://backend-pod || echo "Connection failed"

# Test connectivity from denied pod (other)
kubectl exec -it other-pod -- wget -qO- --timeout=2 http://backend-pod || echo "Connection blocked"
```

## NetworkPolicy Examples

### Default Deny-All NetworkPolicy

**Example (`exercises/networkpolicies/default-deny-all.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-ingress
  namespace: default
spec:
  # Empty podSelector matches all pods
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress traffic
```

**Apply this:**
```bash
kubectl apply -f exercises/networkpolicies/default-deny-all.yaml
```

**Key Points:**
- `podSelector: {}` - Matches all pods (empty selector)
- `policyTypes: [Ingress]` - Controls incoming traffic
- No `ingress` rules - Denies all incoming traffic

### Allow from Specific Pod

**Example (`exercises/networkpolicies/frontend-to-backend.yaml`):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  # Applies to pods with label app: backend
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  # Allow traffic from pods with label app: frontend
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

**Apply this:**
```bash
kubectl apply -f exercises/networkpolicies/frontend-to-backend.yaml
```

**Traffic Flow:**
```
┌─────────────────┐
│ frontend-pod    │ ──────► │  backend-pod   │ ✅ ALLOWED
│ (app: frontend) │         │ (app: backend) │
└─────────────────┘         └────────────────┘

┌─────────────────┐
│ other-pod       │ ──────► │  backend-pod   │ ❌ BLOCKED
│ (app: other)    │         │ (app: backend) │
└─────────────────┘         └────────────────┘
```

### Allow from Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-namespace
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 80
```

**Traffic Flow:**
```
┌──────────────────────────┐
│ production namespace     │ ──────► │  backend-pod   │ ✅ ALLOWED
│ (name: production)       │         │ (app: backend) │
└──────────────────────────┘         └────────────────┘

┌─────────────────────────┐
│ staging namespace       │ ──────► │  backend-pod   │ ❌ BLOCKED
│ (name: staging)         │         │ (app: backend) │
└─────────────────────────┘         └────────────────┘
```

**Note:** When using only `namespaceSelector` (without `podSelector`), **all pods** in that namespace are allowed to reach the target pod.

### Allow from IP Block (External IPs)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-backup-server
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.5.1/32  # Single IP (backup server outside cluster)
    ports:
    - protocol: TCP
      port: 3306
```

**Use Case:** Allow external backup server (outside Kubernetes cluster) to connect to database pod.

**IP Block Format:**
- Single IP: `192.168.5.1/32`
- IP Range: `192.168.5.0/24` (all IPs from 192.168.5.0 to 192.168.5.255)
- CIDR notation: Standard CIDR format

### Combined Selectors (AND Logic)

When multiple selectors are in the **same rule** (same `from` array), they work as **AND** (all must match):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-prod-namespace
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    # Both selectors must match (AND logic)
    - podSelector:
        matchLabels:
          app: api
      namespaceSelector:
        matchLabels:
          name: production
    ports:
    - protocol: TCP
      port: 3306
```

**Traffic Flow:**
```
┌─────────────────────────────────────┐
│ production namespace                │
│ ┌─────────────────────────────────┐ │
│ │ api-pod (app: api)              │ │ ──────► │  database-pod   │ ✅ ALLOWED
│ └─────────────────────────────────┘ │         │ (app: database) │
└─────────────────────────────────────┘         └─────────────────┘

┌─────────────────────────────────────┐
│ production namespace                │
│ ┌─────────────────────────────────┐ │
│ │ web-pod (app: web)              │ │ ──────► │  database-pod   │ ❌ BLOCKED
│ └─────────────────────────────────┘ │         │ (app: database) │
└─────────────────────────────────────┘         └─────────────────┘
(api-pod label doesn't match)

┌─────────────────────────────────────┐
│ staging namespace                   │
│ ┌─────────────────────────────────┐ │
│ │ api-pod (app: api)              │ │ ──────► │  database-pod   │ ❌ BLOCKED
│ └─────────────────────────────────┘ │         │ (app: database) │
└─────────────────────────────────────┘         └─────────────────┘
(production namespace label doesn't match)
```

**Key Point:** Traffic must be from a pod with `app: api` label **AND** in namespace with `name: production` label.

### Rule Logic: OR vs AND

**OR Between Rules (Separate `-` entries):**
Multiple rules in the `ingress` or `egress` array work as **OR** (traffic matching ANY rule is allowed):

```yaml
ingress:
# Rule 1: Allow from frontend pods
- from:
  - podSelector:
      matchLabels:
        app: frontend
  ports:
  - protocol: TCP
    port: 80
# Rule 2: Allow from backup server (OR - separate rule)
- from:
  - ipBlock:
      cidr: 192.168.5.1/32
  ports:
  - protocol: TCP
    port: 3306
```

**Result:** Traffic from frontend pods **OR** from backup server is allowed.

**AND Within a Rule (Same `from` array):**
Multiple selectors in the **same** `from` array work as **AND** (all must match):

```yaml
ingress:
- from:
  # Both must match (AND logic)
  - podSelector:
      matchLabels:
        app: api
    namespaceSelector:
      matchLabels:
        name: production
  ports:
  - protocol: TCP
    port: 3306
```

**Result:** Traffic must be from pod with `app: api` label **AND** in `production` namespace.

**Common Mistake:**
```yaml
# WRONG: This creates two separate rules (OR logic)
ingress:
- from:
  - podSelector:
      matchLabels:
        app: api
- from:
  - namespaceSelector:
      matchLabels:
        name: production

# This allows: api pods from ANY namespace OR any pod from production namespace
```

```yaml
# CORRECT: Combined in one rule (AND logic)
ingress:
- from:
  - podSelector:
      matchLabels:
        app: api
    namespaceSelector:
      matchLabels:
        name: production

# This allows: api pods ONLY from production namespace
```

### Egress Rules

Egress rules control **outgoing traffic** from pods. Use `to` field instead of `from`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-egress-to-backup
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Egress  # Control outgoing traffic
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.1/32  # External backup server
    ports:
    - protocol: TCP
      port: 80
```

**Use Case:** Database pod pushing backup to external backup server.

**Egress Selectors:**
- `to.podSelector` - Allow traffic to specific pods
- `to.namespaceSelector` - Allow traffic to pods in specific namespace
- `to.ipBlock` - Allow traffic to external IP ranges

**Example: Database Egress to External Backup Server**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-egress-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.0/24  # Backup server network
    ports:
    - protocol: TCP
      port: 80
```

**Traffic Flow:**
```
┌──────────────┐
│ database-pod │ ──────► │  Backup Server │ ✅ ALLOWED
│ (app: db)    │         │ (192.168.5.1)  │
└──────────────┘         └────────────────┘
```

### Real-World Example: Web → API → Database

**Scenario:** Protect database so it only accepts traffic from API server on port 3306.

**Traffic Flow:**
```
User → Web Server (port 80) → API Server (port 5000) → Database (port 3306)
```

**NetworkPolicy to Protect Database:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: default
spec:
  # Apply to database pods
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  # Allow only from API pods on port 3306
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 3306
```

**What This Does:**
- Blocks all traffic to database pods by default
- Allows only ingress traffic from pods with `app: api` label
- Allows only on port 3306
- Web server cannot directly access database (blocked)
- API server can query database (allowed)

**Traffic Flow Diagram:**
```
┌──────────────┐
│ web-pod      │ ──────► │  api-pod    │ ✅ ALLOWED (no policy on api)
│ (app: web)   │         │ (app: api)  │
└──────────────┘         └─────────────┘

┌──────────────┐
│ api-pod      │ ──────► │  db-pod     │ ✅ ALLOWED (policy allows it)
│ (app: api)   │         │ (role: db)  │
└──────────────┘         └─────────────┘

┌──────────────┐
│ web-pod      │ ──────► │  db-pod     │ ❌ BLOCKED (policy denies it)
│ (app: web)   │         │ (role: db)  │
└──────────────┘         └─────────────┘
```

## Common Issues & Solutions

### NetworkPolicy Not Enforced

**Symptoms:** Traffic still allowed after applying NetworkPolicy

**Commands:**
```bash
# Check CNI supports NetworkPolicies
kubectl get pods -n kube-system | grep calico

# Check NetworkPolicy is applied
kubectl get networkpolicies

# Describe NetworkPolicy
kubectl describe networkpolicy <policy-name>
```

**Common causes:**
- CNI doesn't support NetworkPolicies → Use Calico CNI
- NetworkPolicy not applied → Verify with `kubectl get netpol`
- Wrong namespace → Check NetworkPolicy is in correct namespace

**Fix:**
```bash
# For Minikube: Start with Calico CNI
minikube stop
minikube start --cni=calico

# Verify Calico is running
kubectl get pods -n kube-system | grep calico
```

### All Traffic Blocked After Default Deny

**Symptoms:** All connectivity fails after applying default deny

**Commands:**
```bash
# List NetworkPolicies
kubectl get networkpolicies

# Check if allow rules exist
kubectl get networkpolicy -o yaml | grep -A 10 "ingress:"

# Test connectivity
kubectl exec -it <pod> -- wget -qO- --timeout=2 http://<service> || echo "Blocked"
```

**Common causes:**
- Default deny applied but no allow rules → Add specific allow rules
- Allow rules don't match → Verify pod labels match selectors
- Wrong port in allow rule → Check port matches service port

**Fix:**
```yaml
# Add allow rule for required traffic
ingress:
- from:
  - podSelector:
      matchLabels:
        app: allowed-client
  ports:
  - protocol: TCP
    port: 80  # Must match service port
```

### Allow Rule Not Working

**Symptoms:** Traffic still blocked even with allow rule

**Commands:**
```bash
# Check pod labels match selector
kubectl get pods --show-labels

# Check NetworkPolicy selector
kubectl get networkpolicy <policy-name> -o yaml | grep -A 5 "podSelector"

# Check ingress rules
kubectl get networkpolicy <policy-name> -o yaml | grep -A 10 "ingress:"
```

**Common causes:**
- Pod labels don't match selector → Fix labels or selector
- Wrong namespace → Check pods and NetworkPolicy in same namespace
- Port mismatch → Verify port in allow rule matches service port
- Rule logic error → Check if using OR when you need AND (or vice versa)

**Fix:**
```yaml
# Verify labels match
# NetworkPolicy:
podSelector:
  matchLabels:
    app: backend

# Pod:
metadata:
  labels:
    app: backend  # Must match
```

### Rule Logic Issues

**Symptoms:** Traffic allowed when it shouldn't be, or blocked when it should be allowed

**Commands:**
```bash
# Check NetworkPolicy rules structure
kubectl get networkpolicy <policy-name> -o yaml

# Verify rule structure (count dashes)
kubectl get networkpolicy <policy-name> -o yaml | grep -A 15 "ingress:\|egress:"
```

**Common causes:**
- Using separate rules (OR) when you need combined (AND) → Combine selectors in same rule
- Using combined selectors (AND) when you need separate rules (OR) → Split into separate rules
- Missing namespace label → Ensure namespace has required label for namespaceSelector

**Fix:**
```yaml
# If you need AND logic (pod must match BOTH selectors):
ingress:
- from:
  - podSelector:
      matchLabels:
        app: api
    namespaceSelector:
      matchLabels:
        name: production

# If you need OR logic (pod matches EITHER selector):
ingress:
- from:
  - podSelector:
      matchLabels:
        app: api
- from:
  - namespaceSelector:
      matchLabels:
        name: production
```
